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

- makes sure `ssh.service` is enabled and `ssh.socket` is stopped and disabled. Ubuntu 24.04 uses socket activation by default, and then restarting `ssh.service` alone does not apply a new `Port`. The role switches sshd to a plain service so that a restart applies config changes
- builds `AllowUsers` from the managed users plus `security_ssh_allow_users`
- refuses to continue if the user Ansible is connected as would be missing from `AllowUsers`
- deploys `/etc/ssh/sshd_config.d/00-hardening.conf`, validated with `sshd -t` before it is written
- before restarting, checks with `sshd -T` that the effective `Port`, `PermitRootLogin`, `PasswordAuthentication`, `KbdInteractiveAuthentication` and `AllowUsers` match the role variables, so a value set earlier by another drop-in (sshd uses the first value it finds) stops the run
- after restart, checks that sshd actually listens on the configured port

**ufw** installs ufw, sets default policies, allows the SSH port and a list of configured ports, enables the firewall. A full reset is available behind a flag.

**fail2ban** installs `fail2ban` (and `python3-pyinotify` when the `pyinotify` backend is used), deploys an sshd jail to `/etc/fail2ban/jail.d/sshd.local` with explicit `port`, `logpath` and `backend`, so the jail does not depend on distribution defaults, validates the whole configuration with `fail2ban-client -t`, then starts and enables the service.

## Requirements

- ansible-core 2.16 or newer
- Ubuntu 22.04 (jammy) or 24.04 (noble)
- `ansible_user` defined in the inventory: the lockout check compares it with `AllowUsers`
- collections `ansible.posix` and `community.general`. They are not installed together with the role, add them to your own `requirements.yml` (see Usage)
- `sshpass` on the control machine, if you connect with a password (a fresh host from a provider, before keys are deployed)
- the host key of the target in `known_hosts`: the role does not disable host key checking, add the key with `ssh-keyscan` before the first run

## Usage

A role does not run on its own. It needs a project around it: an inventory with your hosts, variables for those hosts, and a playbook that calls the role. Pick the case that matches yours.

### Case 1. New project from scratch

You have a control machine with Ansible and a server you want to harden, and nothing else yet.

1. Install ansible-core 2.16 or newer on the control machine. Check with `ansible --version`.

2. Create a project directory and go into it:

   ```bash
   mkdir my-infra && cd my-infra
   ```

3. Create `requirements.yml`:

   ```yaml
   roles:
     - name: security
       src: https://github.com/ituch136/security_hardening.git
       scm: git
       version: v1.2.0

   collections:
     - name: ansible.posix
     - name: community.general
       version: "<12"
   ```

   Keep `name: security`: without it the role is installed under the repository name, and the playbook will not find it as `security`. The version limit on `community.general` matters on ansible-core 2.16: newer releases of the collection do not support it and print a warning on every run.

4. Create `ansible.cfg`, so Ansible installs and looks for the role inside the project:

   ```ini
   [defaults]
   inventory = inventory.ini
   roles_path = ./roles
   ```

5. Install the role and the collections:

   ```bash
   ansible-galaxy install -r requirements.yml
   ```

   The role lands in `./roles/security`.

6. Create `inventory.ini`:

   ```ini
   [app]
   server1 ansible_host=203.0.113.10 ansible_user=ansible ansible_port=22
   ```

   `server1` is the host name inside Ansible. It can be anything, but the variables file in the next step must have exactly this name.

7. Create `host_vars/server1.yml`:

   ```yaml
   security_ansible_user_name: "ansible"
   security_ansible_user_ssh_key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"

   security_admin_user_name: "admin"
   security_admin_user_ssh_key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"

   security_ufw_allowed_ports:
     - { port: 80, proto: tcp, comment: "http" }
     - { port: 443, proto: tcp, comment: "https" }
   ```

   The file name must match the host name from the inventory (`server1` → `server1.yml`), otherwise Ansible does not load it and preflight stops the run. The key path is on the control machine, and `{{ ... }}` needs both double braces.

8. Create `playbook.yml`:

   ```yaml
   - name: Harden servers
     hosts: app
     become: true
     roles:
       - security
   ```

   `become: true` is required: the role changes system files.

9. Check and run:

   ```bash
   ansible-playbook --syntax-check playbook.yml
   ansible-playbook playbook.yml
   ```

   On a fresh host read **First run** in "Read before running" first: the service user does not exist yet, so the first run goes through another account with `-K`.

10. Run the playbook a second time. It should report `changed=0`.

### Case 2. Adding the role to an existing project

You already have a project with an inventory and playbooks.

1. Add the role and the collections to the project's `requirements.yml` (see step 3 of Case 1) and run `ansible-galaxy install -r requirements.yml`.

2. Put the role variables into `host_vars/<host>.yml` or `group_vars/<group>.yml`, whichever your project uses.

3. Call the role from a playbook. Either as a separate play, usually first, so later plays already run on a hardened host:

   ```yaml
   - name: Harden servers
     hosts: app
     become: true
     roles:
       - security

   - name: Deploy application
     hosts: app
     become: true
     roles:
       - my_app
   ```

   or from tasks of an existing play:

   ```yaml
   - name: Harden
     ansible.builtin.import_role:
       name: security
   ```

4. Before the first run on hosts that are already in use, check two things:

   - everyone who logs in over SSH is either a managed user or listed in `security_ssh_allow_users` (see **AllowUsers**)
   - ports your services need are listed in `security_ufw_allowed_ports`, because the default incoming policy is `deny`. Ports published by Docker are not affected, see **Docker**

5. To run only part of the role, use tags, for example `ansible-playbook playbook.yml --tags ssh`. See [Tags](#tags).

### Case 3. Running from a local clone

For testing changes to the role before they are tagged.

1. Clone the repository:

   ```bash
   git clone https://github.com/ituch136/security_hardening.git
   cd security_hardening
   ```

2. Create `ansible.cfg`, `inventory.ini`, `host_vars/<host>.yml` and `playbook.yml` in the repository root, as in Case 1, with two differences:

   - in `ansible.cfg` set `roles_path = ..`: the role is the repository directory itself, so Ansible has to look one level up
   - in `playbook.yml` call the role by the directory name, `security_hardening`, not `security`

3. Keep these files out of git. `inventory.ini` and `host_vars/` are already in `.gitignore`. Exclude the other two locally, without touching `.gitignore`:

   ```bash
   echo -e "ansible.cfg\nplaybook.yml" >> .git/info/exclude
   ```

4. Install the collections:

   ```bash
   ansible-galaxy collection install ansible.posix 'community.general:<12'
   ```

5. Run the playbook as in Case 1, steps 9 and 10.

### Common errors

| Error | Cause |
|---|---|
| `You should set security_ansible_user_name ...` in preflight | Variables not loaded: the `host_vars` file name does not match the host name in the inventory |
| `invalid key specified: {lookup(...` | A brace is missing in `"{{ lookup(...) }}"`, the string was not templated |
| `the role 'security' was not found` | The role was installed without `name: security`, or `roles_path` points elsewhere |
| `Collection community.general does not support Ansible version` | Collection too new for your ansible-core, install `'community.general:<12'` |
| `Current connection user ... is not in allowed users list` | `ansible_user` is not in `AllowUsers`, see **First run** |

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

The default is `nopasswd` only because `passwd` mode needs a password hash, and no hash can ship in `defaults`. For production use `passwd`, so that a stolen SSH key alone does not give root.

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
| `security_ssh_kbd_interactive_authentication` | `"no"` | `KbdInteractiveAuthentication` |
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
| `security_fail2ban_sshd_backend` | `systemd` | Log backend |
| `security_fail2ban_sshd_mode` | `normal` | Filter mode: `normal`, `ddos`, `extra`, `aggressive` |
| `security_fail2ban_sshd_logpath` | `/var/log/auth.log` | Log file |
| `security_fail2ban_sshd_ignoreip` | `[]` | Addresses that are never banned |

The default backend is `systemd`: it reads the journal and needs no log file. The file based backends (`pyinotify`, `polling`, `auto`) read `security_fail2ban_sshd_logpath`, and on images without rsyslog that file does not exist, so preflight stops the run. Install rsyslog on such hosts or keep `systemd`.

With the `systemd` backend `logpath` is not written to the jail at all.

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

## Tags

| Tag | Covers |
|---|---|
| `always` | Preflight checks, run with any `--tags` |
| `apt` | Package cache and upgrades |
| `users` | Service and admin users, keys, sudoers |
| `ssh` | sshd, plus installing ufw and allowing the SSH port |
| `ufw` | Firewall |
| `fail2ban` | fail2ban |

The ufw rule for the SSH port is added inside the sshd block, before sshd is restarted, so changing the port does not cut access: neither with `--tags ssh` nor in a full run. The `ufw` block adds the same rule once more, because `security_ufw_reset` wipes the earlier one.

## Read before running

**First run.** On a fresh host the service user does not exist yet. Connect as an existing user and pass the sudo password once with `-K`:

```bash
ansible-playbook -K playbook.yml
```

Set `ansible_user` to the bootstrap user in the inventory (or pass `-e ansible_user=...`). `-u` is not enough: the inventory value takes precedence, and the lockout check looks at the variable.

Add that bootstrap user to `security_ssh_allow_users` for this run, otherwise the lockout check stops the role before sshd is touched. After the run, switch `ansible_user` in the inventory to the service user.

**AllowUsers.** Any account not listed loses SSH access after sshd restarts. The role checks the user Ansible is connected as, but not other people who log in to the host. Add them to `security_ssh_allow_users`.

**SSH keys.** Keys are added with `exclusive: false`, so other entries in `authorized_keys` are kept. Changing `security_ansible_user_ssh_key` or `security_admin_user_ssh_key` adds the new key and leaves the old one in place: revoking a key is a manual step on the host.

**Changing the SSH port.** The current run keeps working over the existing connection. The next run connects to `ansible_port` from the inventory, so update it. The ufw rule for the old port stays in place unless ufw is reset.

**`security_ufw_reset`.** Deletes every ufw rule, including ones this role did not create, and restores the files in `/etc/ufw/` to package defaults. The firewall is disabled between the reset and the final enable task, and the reset task reports `changed` on every run. Do not turn it on for hosts with manually added rules.

**Docker.** ufw does not filter ports published by Docker containers: that traffic goes through the FORWARD chain, not INPUT. This role does not change that.

**fail2ban.** With an empty `ignoreip` you can ban yourself while testing. Keep console access or a short `bantime` at hand.

**fail2ban backend.** Switching to `pyinotify` or another file based backend requires `/var/log/auth.log` (or whatever `security_fail2ban_sshd_logpath` points to) to exist on the host. Minimal cloud images often ship without rsyslog and keep everything in the journal, so the file is missing and the jail cannot start.

## Testing

Tested on Ubuntu 22.04 and 24.04 with ansible-core 2.16. With the example variables a second run reports `changed=0`.

## License

MIT