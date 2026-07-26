# Ubuntu 24.04 LTS (Noble) server bootstrap

One Ansible playbook prepares any number of Ubuntu 24.04 LTS (Noble) servers without
assuming that every server is in the same lifecycle state.

Each server is discovered from the controller before remote configuration:

1. Try `support` with the shared key and passwordless sudo on the target port.
2. Try `support` on `current_ssh_port` for a custom-port migration.
3. Otherwise require root key access on `bootstrap_ssh_port`.
4. On a fresh server, create `support`, install its key and passwordless sudo,
   then verify that independent connection before continuing.
5. Configure one server at a time and stop the batch on the first failure.

Root passwords are never stored in this repository.

## Resulting server state

- Managed account: `support`
- Console passwords: locked by default, or explicitly rotated through Ansible Vault
- Sudo: passwordless
- SSH: public-key only on configurable port `38575`
- Root SSH login: disabled after the managed path is verified
- SSH forwarding: X11 and agent forwarding disabled; TCP forwarding disabled by default
- Kernel networking: redirects and source-routed packets disabled without changing Docker forwarding
- Host firewall: persistent IPv4 and IPv6 `iptables`
- UFW: disabled when present
- Host inbound access: SSH plus loopback, established traffic, ICMP and DHCP
- Docker: official apt repository, installed on all servers by default
- Existing Docker apt sources: normalized to one managed `docker.asc` entry
- Published container ports: blocked by the `DOCKER-USER` chain by default
- `support` is not a member of the privileged `docker` group
- fail2ban: five attempts in ten minutes, one-hour initial ban, increasing to one week
- Reboots: reported but never performed automatically

## Prerequisites

Install Ansible on the controller:

```bash
python3 -m pip install ansible-core ansible-lint
```

The controller needs one SSH key pair. The defaults use:

```text
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
```

### One-time onboarding for password-only root servers

Before running the playbook, manually copy the same public key to root:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 22 root@SERVER_IP
```

Enter that server's root password when prompted. Repeat once for each
password-only server. If the provider disables root password SSH, install the
key through its console or recovery interface instead.

Verify each fresh server:

```bash
ssh -i ~/.ssh/id_ed25519 -p 22 root@SERVER_IP true
```

This one-time step makes the automated run consistent and avoids keeping root
passwords in inventory, shell history, or Ansible variables.

## Inventory

Create the ignored real inventory from the tracked, inactive example:

```bash
cp inventory/hosts.example.ini inventory/hosts.ini
```

Then uncomment and edit only the hosts you manage in `inventory/hosts.ini`:

```ini
[servers]
fresh-1 ansible_host=203.0.113.10
managed-1 ansible_host=203.0.113.11
old-port-1 ansible_host=203.0.113.12 current_ssh_port=2222
```

Do not set `ansible_user`. Discovery selects `root` or `support`.

For a server already managed on an old custom port, set
`current_ssh_port` to that old port. The target `ssh_port` remains global or can
be overridden per host.

## Configuration

Defaults are in `vars/defaults.yml`. Put local changes in the ignored
`vars/local.yml`:

```yaml
---
controller_private_key_path: /absolute/path/to/id_ed25519
controller_public_key_path: /absolute/path/to/id_ed25519.pub
ssh_port: 38575
timezone: Europe/Moscow
upgrade_packages: true
install_docker: true
manage_docker_repository: true
manage_docker_firewall: true
allow_ssh_tcp_forwarding: false
```

`tasks/load_local_vars.yml` loads this file automatically when it exists.

Important variables:

| Variable | Default | Purpose |
| --- | --- | --- |
| `managed_user` | `support` | Managed account; intentionally validated as `support` |
| `ssh_port` | `38575` | Final SSH port |
| `current_ssh_port` | `22` | Old port to probe during a later migration |
| `bootstrap_ssh_port` | `22` | Root-key port on fresh servers |
| `remote_python_interpreter` | `/usr/bin/python3` | Python path on Ubuntu nodes |
| `timezone` | `Europe/Moscow` | Server timezone |
| `upgrade_packages` | `true` | Apply `apt` distribution upgrade |
| `install_docker` | `true` | Install Docker from its official apt repository |
| `manage_docker_repository` | `true` | Normalize the official Docker apt source before apt runs |
| `manage_docker_firewall` | `true` | Enforce the managed `DOCKER-USER` policy for installed Docker |
| `allow_ssh_tcp_forwarding` | `false` | Allow SSH TCP tunnels; enable only for hosts that require them |
| `ssh_max_auth_tries` | `3` | Maximum authentication attempts per SSH connection |
| `fail2ban_maxretry` | `5` | Failed SSH attempts allowed during `fail2ban_findtime` |
| `fail2ban_findtime` | `600` | SSH failure observation window in seconds |
| `fail2ban_bantime` | `3600` | Initial SSH ban in seconds |
| `fail2ban_bantime_increment` | `true` | Increase bans for repeat offenders |
| `fail2ban_bantime_maxtime` | `604800` | Maximum incremental SSH ban in seconds |
| `manage_console_passwords` | `false` | Rotate and unlock root/support console passwords |

Use absolute controller key paths in `vars/local.yml`.

## Optional console passwords

Console passwords are independent from SSH authentication. Even when console
passwords are configured, SSH remains public-key only, root SSH remains
disabled, and `support` continues to use passwordless sudo.

### Create the password hashes

Generate the hashes on a trusted Ubuntu or WSL machine. Install `mkpasswd`,
which is provided by the `whois` package:

```bash
sudo apt-get install whois
```

Generate the root hash:

```bash
mkpasswd --method=yescrypt
```

Type the desired root console password at the `Password:` prompt. The command
prints one line beginning with `$y$`; copy the entire line.

Run the command again and enter a different password for `support`:

```bash
mkpasswd --method=yescrypt
```

Do not put the plaintext password after the command because that would save it
in shell history. If `yescrypt` is unavailable, generate an accepted SHA-512
crypt hash interactively instead:

```bash
openssl passwd -6
```

That output begins with `$6$`. The hash contains `$` characters, so wrap it in
single quotes when placing it in YAML.

### Store the hashes with Ansible Vault

Create the ignored local variable file as an encrypted Ansible Vault:

```bash
ansible-vault create vars/local.yml
```

Store the generated hashes, not plaintext passwords:

```yaml
---
manage_console_passwords: true
root_console_password_hash: '$y$...'
support_console_password_hash: '$y$...'
```

Replace each placeholder with the complete output from its corresponding
command. Choose different passwords for the two accounts; the playbook also
rejects identical hash values. The accepted formats are yescrypt (`$y$`) and
SHA-512 crypt (`$6$`).

A password hash is not the plaintext password, but it is still sensitive and
can be attacked offline. If a hash has been pasted into chat, a ticket, or any
other shared location, generate a new password and hash before production use.

To change the hashes later, edit the encrypted file:

```bash
ansible-vault edit vars/local.yml
```

Run the playbook with the Vault password:

```bash
ansible-playbook site.yml --ask-vault-pass --limit SERVER_NAME
```

When `manage_console_passwords` is `false`, the playbook keeps the `support`
password locked and does not modify the existing root password.

### Verify console passwords

`passwd -S root` and `passwd -S support` show whether the accounts are locked,
but cannot prove that a particular password is correct. Test the passwords at
the provider or physical console, where SSH restrictions do not apply:

1. Log in as `root` with the new root password.
2. Log out, then log in as `support` with its different password.
3. Run `sudo -n true` as `support` to confirm the managed passwordless sudo rule.

Do not temporarily enable SSH password authentication for this test.

## Validate before connecting

The inventory example has no active hosts, so these commands are safe:

```bash
ansible-playbook -i inventory/hosts.example.ini --syntax-check site.yml
ansible-lint --profile production
ansible-inventory -i inventory/hosts.example.ini --graph
```

After adding real hosts, first inspect what Ansible sees:

```bash
ansible-inventory --graph
ansible-inventory --host fresh-1
```

## Run

Start with one non-critical server:

```bash
ansible-playbook site.yml --ask-vault-pass --limit fresh-1
```

Then run the same playbook for every server:

```bash
ansible-playbook site.yml --ask-vault-pass
```

Servers are configured with `serial: 1`; a failure stops later servers.
Omit `--ask-vault-pass` only when `vars/local.yml` is absent or is not Vault
encrypted.

## SSH migration safety

When the current and target ports differ, the playbook:

1. Allows both ports in the live IPv4 and IPv6 firewall.
2. Configures `sshd` to listen on both ports.
3. Runs `sshd -t`.
4. Reloads systemd and restarts the active Ubuntu SSH path (`ssh.socket` or
   `ssh.service`).
5. Tests a new `support` connection and `sudo -n` from the controller.
6. Switches Ansible to the target port.
7. Disables root and password authentication.
8. Tests the final connection again.
9. Removes every managed allowance for the discovered, current and bootstrap
   ports except the final target port.
10. Verifies the live IPv4 and IPv6 chains end in `DROP`.

If migration or hardening validation fails, the previous SSH files are restored,
SSH is restarted, the old firewall allowance remains, and execution stops.

If a run is interrupted after opening the migration window, rerun the same
playbook. Finalization removes stale Ansible-managed transition and former
target rules for all known old ports. It never flushes Docker-owned chains.

## Firewall and Docker ownership

UFW is not used. Docker publishes traffic through forwarding/NAT chains, which
can bypass host INPUT-oriented firewall tools.

The playbook separates ownership:

- Ansible owns `ANSIBLE_INPUT`, the host policies, and the static
  `/etc/iptables/rules.v4` and `rules.v6` files.
- Docker owns its generated NAT and forwarding chains.
- Ansible owns the contents of Docker's `DOCKER-USER` chain after Docker starts.

Docker repository normalization runs before the first apt update, allowing the
playbook to repair servers that already contain conflicting `.gpg` and `.asc`
Docker source definitions.

The Docker controls are independent:

- Keep all three Docker variables `true` to install and fully manage Docker.
- Set `install_docker: false` on a host with Docker already installed while
  leaving `manage_docker_firewall: true` to retain published-port protection.
- Set `manage_docker_repository: false` only when another tool owns the apt
  source. A requested Docker installation requires repository management.
- Set `manage_docker_firewall: false` only when another reviewed policy owns
  `DOCKER-USER`.

The persistent host files deliberately exclude Docker-generated chains. A
systemd oneshot service reapplies `DOCKER-USER` after Docker starts at boot.
New published-port traffic from outside is dropped; established traffic and
container-originated traffic remain allowed.

Run Docker commands through sudo:

```bash
sudo docker ps
sudo docker compose version
```

## After a successful run

Connect with:

```bash
ssh -i ~/.ssh/id_ed25519 -p 38575 support@SERVER_IP
```

If `/var/run/reboot-required` exists, the final recap reports it. Schedule the
reboot separately after checking the server workload.
