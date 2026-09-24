# security_hardening

Ansible role for basic hardening of an Ubuntu server: service and admin users, sshd, ufw and fail2ban.

The role is built around a few rules:

- own configuration goes into drop-in directories (`sshd_config.d`, `jail.d`, `sudoers.d`), package conffiles are not touched
- results are verified, not just applied: key sshd settings are checked with `sshd -T`, the listening port is checked after restart
- defaults must not break access
- nothing is removed unless a variable explicitly asks for it

## Quick start

This repository is an Ansible role. A role is a library: it has no inventory and no playbook inside, so it cannot be run on its own. You call it from a small project of your own, and the `examples/` directory here is exactly such a project, ready to copy.

You need a control machine with ansible-core 2.16 or newer (`ansible --version`) and an Ubuntu 22.04 or 24.04 server.

1. Copy the example project:

   ```bash
   git clone https://github.com/ituch136/security_hardening.git
   cp -r security_hardening/examples my-infra
   cd my-infra
   ```

   The clone was needed only for the examples. The role itself is installed in the next step, so you can delete the cloned directory afterwards.

2. Install the role and the collections it needs:

   ```bash
   ansible-galaxy install -r requirements.yml
   ```

3. Open `inventory.ini` and put in the address of your server and the user you log in as today.

4. Open `host_vars/testhost.yml` and put in the user names you want to exist on the server and the paths to your public SSH keys. Also list the ports your services need: the firewall denies everything that is not listed.

5. Run it:

   ```bash
   ansible-playbook playbook.yml
   ```

If the server is brand new and the only account you have is root with a password, do not start with step 5. Read the next section first, it is the common case and it needs two extra settings.

## First run on a new server

A provider usually gives you root and a password. The role creates your own users, switches sshd to keys only and takes SSH access away from root. That is the point of it, but it means the account you are connected with right now is the account that loses access at the end of the run. Ansible refuses to do that silently, so you have to say that it is intended.

1. Install `sshpass` on the control machine: Ansible cannot type an SSH password without it.

   ```bash
   sudo apt install sshpass
   ```

2. Decide what to do with the server's host key. The example `ansible.cfg` already contains:

   ```ini
   ssh_args = -o StrictHostKeyChecking=accept-new -o ControlMaster=auto -o ControlPersist=60s
   ```

   That accepts the key on the first connection without a prompt and verifies it on every connection after that, so nothing extra is needed to get started. See "Host keys" below for what this trades away and how to be strict about it.

3. In `inventory.ini` connect as root:

   ```ini
   [app]
   testhost ansible_host=203.0.113.10 ansible_user=root ansible_port=22
   ```

4. In `host_vars/testhost.yml` allow the role to cut off the account you are using:

   ```yaml
   security_ssh_allow_lockout: true
   ```

5. Run the playbook and type the root password when asked:

   ```bash
   ansible-playbook playbook.yml --ask-pass
   ```

   The users are created first, then sshd restarts. Root loses SSH access at that moment, while your current connection survives to the end of the run.

6. Switch the project to the new user: remove `security_ssh_allow_lockout` from `host_vars/testhost.yml`, change `ansible_user` in the inventory to the service user you created, and run again, this time without `--ask-pass`:

   ```bash
   ansible-playbook playbook.yml
   ```

   The second run should report `changed=0`. That is also the proof that the new access works.

If you changed `security_ssh_port`, update `ansible_port` in the inventory as well before the second run.

**The risk.** Between the sshd restart and the end of the run there is a window where root can no longer log in. If the connection drops in that window, you are left with the provider's console. On a server you cannot afford to lose this way, do it in two runs instead: `ansible-playbook playbook.yml --tags users --ask-pass` as root, then switch `ansible_user` to the service user and do the full run. No lockout flag is needed then.

**If your bootstrap account is not root** but an ordinary user with sudo, there is nothing to lock yourself out of: add that user to `security_ssh_allow_users` instead of setting the lockout flag, and pass the sudo password with `-K`.

### Host keys

`StrictHostKeyChecking=accept-new` in the example `ansible.cfg` means: the first time you connect, the server's key is stored without asking; from then on a changed key aborts the connection. It keeps the first run simple and still protects every run after it.

What it does not protect is that very first connection. If somebody sits between you and the server at that moment, you store their key and never notice, and on the first run you are sending the root password over that connection. Plain `ssh-keyscan` has exactly the same weakness: it trusts whoever answers.

The only way to remove that risk is to compare fingerprints. Most providers show the host key fingerprints in the web console or in the server's setup output. If you care, take them from there and add the key yourself:

```bash
ssh-keyscan 203.0.113.10 >> ~/.ssh/known_hosts
ssh-keygen -lf ~/.ssh/known_hosts | grep 203.0.113.10
```

Compare the printed fingerprint with the one in the console. If they differ, delete the line and find out why before you connect again.

Never replace this with `host_key_checking = False`: that setting stays in the config forever and silently drops the check on every host and every future run, including the ones where you send a password.

If you keep `accept-new`, remember that `ssh_args` replaces the defaults rather than adding to them. That is why `ControlMaster` and `ControlPersist` are in the same line: without them every task opens a new SSH connection and runs are noticeably slower.

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
- refuses to continue if the user Ansible is connected as would be missing from `AllowUsers`, unless `security_ssh_allow_lockout` is set
- deploys `/etc/ssh/sshd_config.d/00-hardening.conf`, validated with `sshd -t` before it is written
- before restarting, checks with `sshd -T` that the effective `Port`, `PermitRootLogin`, `PasswordAuthentication`, `KbdInteractiveAuthentication` and `AllowUsers` match the role variables, so a value set earlier by another drop-in (sshd uses the first value it finds) stops the run
- after restart, checks that sshd actually listens on the configured port

**ufw** installs ufw, sets default policies, allows the SSH port and a list of configured ports, enables the firewall. A full reset is available behind a flag.

**fail2ban** installs `fail2ban` (and `python3-pyinotify` when the `pyinotify` backend is used), deploys an sshd jail to `/etc/fail2ban/jail.d/sshd.local` with explicit `port`, `backend` and, for file based backends, `logpath`, so the jail does not depend on distribution defaults, validates the whole configuration with `fail2ban-client -t`, then starts and enables the service.

## Requirements

- ansible-core 2.16 or newer
- Ubuntu 22.04 (jammy) or 24.04 (noble) on the target
- collections `ansible.posix` and `community.general`. They do not come with the role, install them from your own `requirements.yml`
- `ansible_user` defined in the inventory: the lockout check compares it with `AllowUsers`
- `sshpass` on the control machine, if you log in with a password
- a decision about the server's host key. The example `ansible.cfg` sets `StrictHostKeyChecking=accept-new`, which accepts the key on the first connection and verifies it on every later one. See "Host keys" below

## Usage

The Quick start copies a ready project. This section explains what is inside it, how to add the role to a project you already have, and how to run the role from a local clone while working on it.

### Case 1. Building the project by hand

The same result as the Quick start, file by file.

1. Create a project directory and go into it:

   ```bash
   mkdir my-infra && cd my-infra
   ```

2. Create `requirements.yml`:

   ```yaml
   roles:
     - name: security
       src: https://github.com/ituch136/security_hardening.git
       scm: git
       version: v1.3.0

   collections:
     - name: ansible.posix
     - name: community.general
       version: "<12"
   ```

   Keep `name: security`: without it the role is installed under the repository name, and the playbook will not find it as `security`. The version limit on `community.general` matters on ansible-core 2.16: newer releases of the collection do not support it and print a warning on every run.

3. Create `ansible.cfg`, so Ansible installs and looks for the role inside the project:

   ```ini
   [defaults]
   inventory = inventory.ini
   roles_path = ./roles

   [ssh_connection]
   ssh_args = -o StrictHostKeyChecking=accept-new -o ControlMaster=auto -o ControlPersist=60s
   ```

   The `ssh_args` line accepts an unknown host key on the first connection and verifies it afterwards, see "Host keys". Ansible only reads `ansible.cfg` from the directory you run the command in, so always run `ansible-playbook` from the project root.

4. Install the role and the collections:

   ```bash
   ansible-galaxy install -r requirements.yml
   ```

   The role lands in `./roles/security`.

5. Create `inventory.ini`:

   ```ini
   [app]
   testhost ansible_host=203.0.113.10 ansible_user=ansible ansible_port=22
   ```

   `ansible_user` is the account you log in as **today**, not the one the role will create. On a brand new server that is usually `root`, see "First run on a new server".

   `testhost` is the host name inside Ansible. It can be anything, but the variables file in the next step must be named after it.

6. Create `host_vars/testhost.yml`:

   ```yaml
   security_ansible_user_name: "ansible"
   security_ansible_user_ssh_key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"

   security_admin_user_name: "admin"
   security_admin_user_ssh_key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"

   security_ufw_allowed_ports:
     - { port: 80, proto: tcp, comment: "http" }
     - { port: 443, proto: tcp, comment: "https" }
   ```

   Three things people get wrong here:

   - the file name must match the host name from the inventory (`testhost` → `testhost.yml`), otherwise Ansible does not load it and preflight stops the run
   - `{{ ... }}` needs both braces on each side, and the key path is on the control machine, not on the server
   - the incoming firewall policy is `deny`, so every port your services need has to be in `security_ufw_allowed_ports`. The SSH port is handled by the role itself. Ports published by Docker containers are a special case, see **Docker**

7. Create `playbook.yml`:

   ```yaml
   - name: Harden servers
     hosts: app
     become: true
     roles:
       - security
   ```

   `become: true` is required: the role changes system files.

8. Check and run:

   ```bash
   ansible-playbook --syntax-check playbook.yml
   ansible-playbook playbook.yml
   ```

9. Run the playbook a second time. It should report `changed=0`.

### Case 2. Adding the role to an existing project

1. Add the role and the collections to the project's `requirements.yml` (see step 2 of Case 1) and run `ansible-galaxy install -r requirements.yml`.

2. Put the role variables into `host_vars/<host>.yml` or `group_vars/<group>.yml`, whichever your project uses.

3. Call the role from a playbook, usually as the first play, so later plays already run on a hardened host:

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

   or from the tasks of an existing play:

   ```yaml
   - name: Harden
     ansible.builtin.import_role:
       name: security
   ```

4. On hosts that are already in use, check two things before the first run:

   - everyone who logs in over SSH is either a managed user or listed in `security_ssh_allow_users`, otherwise they lose access, see **AllowUsers**
   - every port your services need is in `security_ufw_allowed_ports`, because the default incoming policy is `deny`

5. To run only part of the role, use tags, for example `ansible-playbook playbook.yml --tags ssh`. See [Tags](#tags).

### Case 3. Running the role from a local clone

For working on the role itself, before the changes are tagged.

1. Clone the repository and go into it:

   ```bash
   git clone https://github.com/ituch136/security_hardening.git
   cd security_hardening
   ```

2. Create `ansible.cfg`, `inventory.ini`, `host_vars/<host>.yml` and `playbook.yml` in the repository root, as in Case 1, with two differences:

   - in `ansible.cfg` set `roles_path = ..`, because the role is this directory itself and Ansible has to look one level up
   - in `playbook.yml` call the role by the directory name, `security_hardening`, not `security`

3. Keep those files out of git. `inventory.ini`, `host_vars/` and `group_vars/` in the repository root are already in `.gitignore`. Exclude the other two locally, without touching `.gitignore`:

   ```bash
   echo -e "ansible.cfg\nplaybook.yml" >> .git/info/exclude
   ```

4. Install the collections:

   ```bash
   ansible-galaxy collection install ansible.posix 'community.general:<12'
   ```

5. Run the playbook as in Case 1.

### Common errors

| Error | Cause |
|---|---|
| `to use the 'ssh' connection type with passwords, you must install the sshpass program` | Logging in with a password without `sshpass` on the control machine |
| `Using a SSH password instead of a key is not possible because Host Key checking is enabled` | The server's key is unknown and `accept-new` is not in effect: check that you run from the project directory so its `ansible.cfg` is used, see "Host keys" |
| `Current connection user ... is not in allowed users list` | The account you are connected with would lose SSH access, see "First run on a new server" |
| `You should set security_ansible_user_name ...` in preflight | Variables not loaded: the `host_vars` file name does not match the host name in the inventory |
| `invalid key specified: {lookup(...` | A brace is missing in `"{{ lookup(...) }}"`, so the value was never templated |
| `couldn't resolve module/action 'ansible.posix.authorized_key'` | The collections were not installed, run `ansible-galaxy install -r requirements.yml` |
| `the role 'security' was not found` | The role was installed without `name: security`, or `roles_path` points elsewhere |
| `Collection community.general does not support Ansible version` | The collection is too new for your ansible-core, install `'community.general:<12'` |
| `Backend 'pyinotify' reads /var/log/auth.log, but the file does not exist` | Image without rsyslog, keep the `systemd` backend or install rsyslog |

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
| `security_ssh_allow_lockout` | `false` | Allow the connecting user to lose SSH access |

If the resulting user list is empty, `AllowUsers` is not written at all.

By default the role stops when the account Ansible is connected with would not be in `AllowUsers`, because that account loses SSH access the moment sshd restarts. `security_ssh_allow_lockout: true` replaces that check with a warning and lets the run continue. It is meant for the first run on a new server, where you connect as root and do not want root in `AllowUsers`. See "First run on a new server".

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
| `security_fail2ban_sshd_logpath` | `/var/log/auth.log` | Log file, file based backends only |
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

**AllowUsers.** Any account not listed loses SSH access after sshd restarts. The role checks the account Ansible is connected with, but knows nothing about other people who log in to the host. Add them to `security_ssh_allow_users`.

**SSH keys.** Keys are added with `exclusive: false`, so other entries in `authorized_keys` are kept. Changing `security_ansible_user_ssh_key` or `security_admin_user_ssh_key` adds the new key and leaves the old one in place: revoking a key is a manual step on the host.

**Changing the SSH port.** The current run keeps working over the existing connection. The next run connects to `ansible_port` from the inventory, so update it. The ufw rule for the old port stays in place unless ufw is reset.

**`security_ufw_reset`.** Deletes every ufw rule, including ones this role did not create, and restores the files in `/etc/ufw/` to package defaults. The firewall is disabled between the reset and the final enable task, and the reset task reports `changed` on every run. Do not turn it on for hosts with manually added rules.

**Docker.** ufw does not filter ports published by Docker containers: that traffic goes through the FORWARD chain, not INPUT. This role does not change that.

**fail2ban.** With an empty `ignoreip` you can ban yourself while testing. Keep console access or a short `bantime` at hand.

**fail2ban backend.** A file based backend needs `/var/log/auth.log` (or whatever `security_fail2ban_sshd_logpath` points to) to exist. Minimal cloud images often ship without rsyslog and keep everything in the journal, so the file is missing and the jail cannot start. Preflight catches this before anything is changed.

## Testing

Tested on Ubuntu 22.04 and 24.04 with ansible-core 2.16. With the example variables a second run reports `changed=0`.

## License

MIT