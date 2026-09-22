# security_hardening

Ansible role for basic hardening of an Ubuntu server: service and admin users, sshd, ufw and fail2ban.

The role is built around a few rules:

- own configuration goes into drop-in directories (`sshd_config.d`, `jail.d`, `sudoers.d`), package conffiles are not touched
- results are verified, not just applied: key sshd settings are checked with `sshd -T`, the listening port is checked after restart
- defaults must not break access
- nothing is removed unless a variable explicitly asks for it

## What it does

**Preflight** checks required variables and allowed values (sudo mode, ufw policies) before anything is changed.

**apt** optionally updates the cache, upgrades packages and removes unused dependencies. Upgrades are off by default, so a run does not change package versions unexpectedly.

**Users** creates two independent accounts, each behind its own flag:

- a service user for automation: SSH key, locked password, passwordless sudo
- a human admin: SSH key, sudo either without a password or with one (`security_admin_user_sudo_mode`)

Sudoers rules are written to `/etc/sudoers.d/50-<user>` and validated with `visudo -cf` before they are put in place.

**sshd**

- makes sure `ssh.service` is enabled and `ssh.socket` is stopped and disabled. On Ubuntu 24.04 sshd is socket-activated by default, and the socket listens on the port from its own unit (`ListenStream=22`), ignoring `Port` in the config
- builds `AllowUsers` from the managed users plus `security_ssh_allow_users`
- refuses to continue if the user Ansible is connected as would be missing from `AllowUsers`
- deploys `/etc/ssh/sshd_config.d/00-hardening.conf`, validated with `sshd -t` before it is written
- before restarting, checks with `sshd -T` that the effective `Port`, `PermitRootLogin`, `PasswordAuthentication` and `AllowUsers` match the role variables, so a value overridden by another drop-in stops the run
- after restart, checks that sshd actually listens on the configured port

**ufw** installs ufw, sets default policies, allows the SSH port and a list of configured ports, enables the firewall. A full reset is available behind a flag.

**fail2ban** installs `fail2ban` (and `python3-pyinotify` when the `pyinotify` backend is used), deploys an sshd jail to `/etc/fail2ban/jail.d/sshd.local` with explicit `port`, `logpath` and `backend`, so the jail does not depend on distribution defaults, validates the whole configuration with `fail2ban-client -t`, then starts and enables the service.

## Requirements

- ansible-core 2.16 or newer
- Ubuntu 22.04 (jammy) or 24.04 (noble)
- `ansible_user` defined in the inventory: the lockout check compares it with `AllowUsers`
- collections `ansible.posix` and `community.general`. They are not installed together with the role, add them to your own `requirements.yml` (see Installation)

## Installation

Quick install:

```bash
ansible-galaxy role install git+https://github.com/ituch136/security_hardening.git,v1.0.0,security
ansible-galaxy collection install ansible.posix community.general
```

For a project, declare the dependencies in its `requirements.yml`:

```yaml
roles:
  - name: security
    src: https://github.com/ituch136/security_hardening.git
    scm: git
    version: v1.0.0

collections:
  - name: ansible.posix
  - name: community.general
```

Then install:

```bash
ansible-galaxy install -r requirements.yml
```

Keep `name: security`: without it the role is installed under the repository name, and the playbook will not find it as `security`.

## Role variables

All variables use the `security_` prefix. Defaults are in `defaults/main.yml`.

### Users

| Variable | Default | Description |
|---|---|---|
| `security_manage_ansible_user` | `true` | Create and manage the service user |
| `security_ansible_user_name` | `""` | Service user name, required when managed |
| `security_ansible_user_ssh_key` | `""` | Public key for the service user, required when managed |
| `security_ansible_user_shell` | `/bin/bash` | Shell |
| `security_manage_admin_user` | `true` | Create and manage the admin user |
| `security_admin_user_name` | `""` | Admin user name, required when managed |
| `security_admin_user_ssh_key` | `""` | Public key for the admin, required when managed |
| `security_admin_user_shell` | `/bin/bash` | Shell |
| `security_admin_user_groups` | `[sudo]` | Extra groups, appended |
| `security_admin_user_sudo_mode` | `nopasswd` | `nopasswd` or `passwd` |
| `security_admin_user_password` | `""` | Password **hash**, required when sudo mode is `passwd` |

In `nopasswd` mode the admin gets sudo from a file in `sudoers.d`. In `passwd` mode that file is removed and sudo comes only from membership in the `sudo` group, so keep `sudo` in `security_admin_user_groups`. Groups are only appended, never removed.

The password must be a hash, not plain text. The `user` module writes the value to `/etc/shadow` as is. Generate one with:

```bash
mkpasswd --method=sha-512
```

Keep it in `ansible-vault`, not in plain host_vars.

### sshd

| Variable | Default | Description |
|---|---|---|
| `security_ssh_port` | `22` | SSH port |
| `security_ssh_permit_root_login` | `"no"` | `PermitRootLogin` |
| `security_ssh_password_authentication` | `"no"` | `PasswordAuthentication` |
| `security_ssh_x11_forwarding` | `"no"` | `X11Forwarding` |
| `security_ssh_max_auth_tries` | `3` | `MaxAuthTries` |
| `security_ssh_login_grace_time` | `30` | `LoginGraceTime` |
| `security_ssh_allow_users` | `[]` | Extra users for `AllowUsers` |

If the resulting user list is empty, `AllowUsers` is not written at all.

### ufw

| Variable | Default | Description |
|---|---|---|
| `security_ufw_policy_incoming` | `deny` | `allow`, `deny` or `reject` |
| `security_ufw_policy_outgoing` | `allow` | `allow`, `deny` or `reject` |
| `security_ufw_allowed_ports` | `[]` | Ports to allow, see below |
| `security_ufw_reset` | `false` | Reset ufw before applying rules |

Each item of `security_ufw_allowed_ports`:

```yaml
security_ufw_allowed_ports:
  - { port: 443, proto: tcp, comment: "https" }
  - { port: 9100, proto: tcp, from_ip: "203.0.113.10", comment: "node_exporter" }
```

`proto` defaults to `tcp`, `from_ip` to `any`, `comment` is optional. The SSH port is allowed by a separate task and does not need to be in this list.

### fail2ban

| Variable | Default | Description |
|---|---|---|
| `security_fail2ban_sshd_enabled` | `true` | Enable the sshd jail |
| `security_fail2ban_sshd_maxretry` | `3` | Failures before a ban |
| `security_fail2ban_sshd_findtime` | `10m` | Window for counting failures |
| `security_fail2ban_sshd_bantime` | `1h` | Ban duration |
| `security_fail2ban_sshd_backend` | `pyinotify` | Log backend |
| `security_fail2ban_sshd_mode` | `normal` | Filter mode: `normal`, `ddos`, `extra`, `aggressive` |
| `security_fail2ban_sshd_logpath` | `/var/log/auth.log` | Log file |
| `security_fail2ban_sshd_ignoreip` | `[]` | Addresses that are never banned |

The jail port always follows `security_ssh_port`.

### apt

| Variable | Default | Description |
|---|---|---|
| `security_apt_update_cache` | `true` | Update the package cache |
| `security_apt_cache_valid_time` | `3600` | Cache age in seconds before it is refreshed |
| `security_apt_upgrade` | `"no"` | Value for the `upgrade` option of the apt module |
| `security_apt_autoremove` | `false` | Remove unused dependencies |
| `security_apt_purge` | `false` | Purge configs of removed packages |

### Lists must be lists

`security_ssh_allow_users` and `security_fail2ban_sshd_ignoreip` are joined into one line by the role. Write them as YAML lists:

```yaml
security_fail2ban_sshd_ignoreip:
  - 203.0.113.10
  - 198.51.100.0/24
```

A comma-separated string such as `203.0.113.10, 198.51.100.5` is a single string, not a list: in `security_fail2ban_sshd_ignoreip` it produces a broken config, in `security_ssh_allow_users` it fails the run. Quotes are not needed for IPv4 addresses and subnets. Quote IPv6 addresses.

## Example

Inventory:

```ini
[app]
server1 ansible_host=203.0.113.10 ansible_user=ansible ansible_port=22
```

`host_vars/server1.yml`:

```yaml
security_ansible_user_name: "ansible"
security_ansible_user_ssh_key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"

security_admin_user_name: "admin"
security_admin_user_ssh_key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"

security_ufw_allowed_ports:
  - { port: 80, proto: tcp, comment: "http" }
  - { port: 443, proto: tcp, comment: "https" }
```

Playbook:

```yaml
- name: Harden servers
  hosts: app
  become: true
  roles:
    - security
```

## Tags

| Tag | Covers |
|---|---|
| `always` | Preflight checks, run with any `--tags` |
| `apt` | Package cache and upgrades |
| `users` | Service and admin users, keys, sudoers |
| `ssh` | sshd, plus installing ufw and allowing the SSH port |
| `ufw` | Firewall |
| `fail2ban` | fail2ban |

The `ssh` tag also adds the ufw rule for the SSH port. Without it, changing the port with `--tags ssh` on a host with an active firewall would lock you out.

## Read before running

**First run.** On a fresh host the service user does not exist yet. Connect as an existing user and pass the sudo password once with `-K`:

```bash
ansible-playbook -K playbook.yml
```

Add that bootstrap user to `security_ssh_allow_users` for this run, otherwise the lockout check stops the role before sshd is touched. After the run, switch `ansible_user` in the inventory to the service user.

**AllowUsers.** Any account not listed loses SSH access after sshd restarts. The role checks the user Ansible is connected as, but not other people who log in to the host. Add them to `security_ssh_allow_users`.

**Changing the SSH port.** The current run keeps working over the existing connection. The next run connects to `ansible_port` from the inventory, so update it. The ufw rule for the old port stays in place unless ufw is reset.

**`security_ufw_reset`.** Deletes every ufw rule, including ones this role did not create, and restores the files in `/etc/ufw/` to package defaults. The firewall is disabled between the reset and the final enable task, and the reset task reports `changed` on every run. Do not turn it on for hosts with manually added rules.

**Docker.** ufw does not filter ports published by Docker containers: that traffic goes through the FORWARD chain, not INPUT. This role does not change that.

**fail2ban.** With an empty `ignoreip` you can ban yourself while testing. Keep console access or a short `bantime` at hand.

## Testing

Tested on Ubuntu 22.04 and 24.04 with ansible-core 2.16. With default variables a second run reports `changed=0`.

## License

MIT