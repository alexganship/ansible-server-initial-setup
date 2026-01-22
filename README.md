# Ubuntu bootstrap via Ansible

Opinionated Ansible bootstrap for fresh Ubuntu servers: create or reuse a non-root sudo user, harden SSH, configure firewall and fail2ban, install Docker, set timezone, and install common admin tools.

This playbook configures an Ubuntu server over SSH. It can start from `root` or an existing non-root user.

## Prerequisites
- Ansible installed on your local machine.
- SSH access to the server (key or password).

## What it does
- Change root password (if provided)
- Create a new sudo user (optional, default when connecting as root)
- Set user password and add your public SSH key
- Update/upgrade packages
- Install base packages
- Configure SSH (custom port, disable root login/password auth, public-key only)
- Install fail2ban
- Configure iptables persistence and allow the SSH port
- Install Docker and add the user to `docker` group
- Set timezone to Europe/Moscow

## Inventory
Edit `inventory/hosts.ini`:

```
[servers]
ubuntu-1 ansible_host=203.0.113.10 ansible_user=root
```

If you already have a non-root user, set `ansible_user` to that user.

## Vars
Defaults are in `vars/defaults.yml`. Copy and edit:

```
cp vars/defaults.yml vars/local.yml
```

Then run with:

```
ansible-playbook site.yml -e @vars/local.yml
```
or if you provide root password instead of ssh key
```
ansible-playbook site.yml -e @vars/local.yml --ask-pass
```

### Variable descriptions
`vars/defaults.yml` contains these keys:

```
new_user               # New sudo user to create (used when create_user is true)
new_user_password      # Password for new_user (plain text, will be hashed)
new_root_password      # New root password (optional, leave empty to skip)
ssh_port               # SSH port to configure on the server
local_public_key_path  # Path on controller to your public SSH key
timezone               # Server timezone
create_user            # auto|true|false (auto = create only when connecting as root)
copy_ssh_key           # Copy local_public_key_path to the primary user
install_docker         # Install Docker via get.docker.com script
```

### Notes
- `create_user` defaults to `auto` (create when connecting as `root`, skip when connecting as non-root).
- To skip user creation explicitly, set `create_user: false`.
- To force user creation while connecting as non-root, set `create_user: true`.
- If you do not want to change the root password, omit `new_root_password`.
- To skip SSH key copy, set `copy_ssh_key: false`.
- To skip Docker install, set `install_docker: false`.

## Connect after run
If the playbook completes successfully, connect with:

```
ssh -p 38575 deploy@203.0.113.10
```

If you used a different user or port, replace them accordingly.
