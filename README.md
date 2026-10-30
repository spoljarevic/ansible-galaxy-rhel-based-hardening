rhel-based-hardening
====================

Baseline security hardening for Rocky Linux / RHEL-based servers:

- Installs and enables `firewalld`, `fail2ban`, `chrony`, `dnf-automatic`, and `policycoreutils-python-utils` (via the `epel-release` repo for `fail2ban`).
- Opens ports 80/tcp, 443/tcp, and a custom SSH port in `firewalld`, and allows sshd to bind that port under SELinux.
- Deploys a hardened `fail2ban` jail for sshd.
- Hardens `/etc/ssh/sshd_config` (custom port, no root login, key-only auth).
- Creates an admin user in the `wheel` group with a home directory, and installs an SSH public key into that user's `authorized_keys`.
- Optionally sets the server hostname.
- Reloads `firewalld`, `fail2ban`, and `sshd` via handlers once all changes are applied.

`podman` and `neovim` are deliberately **not** installed - they aren't hardening concerns.

Requirements
------------

- Rocky Linux / RHEL 8 or 9 (or compatible), managed as root or with `become: true`.
- The `passlib` Python library on the control node (used by `password_hash` to hash the admin password).
- Ansible collections `ansible.posix` and `community.general` (used for the `firewalld` and SELinux `seport` modules). Install with:

  ```bash
  ansible-galaxy collection install -r requirements.yml
  ```

Role Variables
---------------

| Variable | Required | Default | Description |
|---|---|---|---|
| `ssh_port` | **Yes** (no default) | - | New SSH port. Opened in `firewalld`, allowed under SELinux (`ssh_port_t`), and written into `sshd_config`. The role fails fast if this isn't set. |
| `ssh_pubkey_path` | **Yes** | `""` | Local (control node) path to the public key file copied into the admin user's `~/.ssh/authorized_keys`, e.g. `~/.ssh/id_ed25519.pub`. |
| `vault_user_name` | **Yes** (from vault) | - | Username of the admin account created in the `wheel` group. Comes from the encrypted `user.yml` vault - see below. |
| `vault_user_password` | **Yes** (from vault) | - | Plaintext password for that user; the role hashes it with `password_hash('sha512')`. Comes from the encrypted `user.yml` vault. |
| `hardening_user_shell` | No | `/bin/bash` | Login shell for the admin user. |
| `hardening_hostname` | No | `""` | New hostname for the server. Leave empty to skip changing it. |
| `firewalld_zone` | No | `public` | firewalld zone that ports are opened in / set as the default zone. |
| `baseline_packages` | No | `firewalld`, `fail2ban`, `chrony`, `dnf-automatic`, `policycoreutils-python-utils` | Packages guaranteed to be present. |
| `fail2ban_bantime` | No | `1h` | fail2ban `bantime` for the `[DEFAULT]` and `[sshd]` sections. |
| `fail2ban_findtime` | No | `10m` | fail2ban `findtime` for the `[DEFAULT]` and `[sshd]` sections. |
| `fail2ban_maxretry` | No | `5` | fail2ban `maxretry` for the `[DEFAULT]` and `[sshd]` sections. |
| `fail2ban_ignoreip` | No | `127.0.0.1/8 ::1` | fail2ban `ignoreip` allowlist. |

`vault_user_name` and `vault_user_password` are **not** defaults of this role - they must come from your own ansible-vault encrypted `user.yml`, kept in your playbook/inventory (not inside this role). This keeps the role itself free of secrets.

Vault setup (user.yml)
-----------------------

Create the encrypted vault next to your playbook:

```bash
ansible-vault create user.yml
```

Add this content in the editor that opens:

```yaml
vault_user_name: yourusername
vault_user_password: yourplaintextpassword
```

Save and exit. To edit it later:

```bash
ansible-vault edit user.yml
```

Reference it from your playbook with `vars_files: [user.yml]` (see example below), and run the playbook with:

```bash
ansible-playbook -i inventory site.yml --ask-vault-pass
```

or, with a vault password file:

```bash
ansible-playbook -i inventory site.yml --vault-password-file ~/.vault_pass.txt
```

Dependencies
------------

None. Requires the `ansible.posix` and `community.general` collections (see Requirements above).

Example Playbook
-----------------

```yaml
- hosts: servers
  become: true
  vars_files:
    - user.yml
  vars:
    ssh_port: 2204
    ssh_pubkey_path: "~/.ssh/id_ed25519.pub"
    hardening_hostname: "web01"
  roles:
    - spoljarevic.rhel-based-hardening
```

License
-------

MIT

Author Information
-------------------

spoljarevic
