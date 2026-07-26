# Repository guidance for coding agents

## Purpose and structure

This repository provides one idempotent Ansible playbook for preparing Ubuntu
24.04 LTS (Noble) servers. `site.yml` discovers either the managed `support`
path or a fresh root-key path, bootstraps `support` when needed, then converges
hosts one at a time. Reusable steps live in `tasks/`, rendered configuration in
`templates/`, safe defaults in `vars/defaults.yml`, and operator instructions
in `README.md`.

## Secret and inventory safety

- Never read, print, copy, decrypt, or stage `vars/local.yml`.
- Never read, print, copy, or stage `inventory/hosts.ini`.
- Never inspect or output private SSH keys, plaintext passwords, Vault
  passwords, or console password hashes.
- Use only `inventory/hosts.example.ini` for static validation. Its host lines
  must remain commented and use documentation-only addresses.
- Keep `vars/local.yml` and `inventory/hosts.ini` ignored.

## Remote-change boundary

- Repository review and local validation do not authorize server changes.
- Do not run the playbook, SSH mutation commands, reboots, package upgrades, or
  firewall changes against a real host without explicit approval for that run.
- For approved rollouts, start with one named canary using `--limit`; preserve
  `serial: 1` and `any_errors_fatal: true`.
- Never request or handle the user's Ansible Vault password. Give the operator
  the `--ask-vault-pass` command to run locally.

## Security invariants

- The managed account is `support`, uses the controller key, and has
  passwordless sudo. Root and password SSH authentication remain disabled.
- SSH migration must independently prove the new key-based path before closing
  an old port. Roll back SSH configuration on failed verification.
- UFW is not the firewall owner. Ansible owns `ANSIBLE_INPUT`, INPUT/OUTPUT host
  policies, and `/etc/iptables/rules.v4` and `rules.v6`.
- Do not flush or persist Docker-generated chains. Docker owns its NAT and
  forwarding chains; Ansible owns only the contents of `DOCKER-USER`.
- Keep the live target SSH allowance before a terminal `DROP` in both IPv4 and
  IPv6. Interrupted runs must remain recoverable.
- Do not disable kernel IP forwarding because Docker requires it.

## Implementation expectations

- Use fully qualified Ansible collection names.
- Keep tasks idempotent and give read-only checks `changed_when: false`.
- Validate generated SSH, sudoers, and iptables files before activation.
- Keep defaults, assertions, effective-state verification, and README
  documentation aligned when adding a variable or security control.
- Preserve unrelated worktree changes. Use `apply_patch` for manual edits.
- Do not stage, commit, push, or create a pull request unless the user asks.
  Even when staging is requested, never stage the real inventory or local Vault.

## Local validation

Run these from the repository root:

```bash
ansible-playbook -i inventory/hosts.example.ini --syntax-check site.yml
ansible-lint --profile production
ansible-inventory -i inventory/hosts.example.ini --graph
git diff --check
```

Warnings that `vars/local.yml` cannot be decrypted during lint are expected;
do not supply a Vault password to suppress them.
