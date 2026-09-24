# ansible-hosting

Ansible playbooks for deploying sites on VPS.

## Setup

Install required collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

Sensitive variables (SMTP credentials, etc.) are encrypted with Ansible Vault, one file per host under `playbooks/host_vars/<host>/vault.yml`. Note: `host_vars`/`group_vars` must live next to the playbook or next to the inventory file, never at the project root.

Set a vault password file once, so you don't have to type it on every run:

```bash
echo "your_vault_password" > ~/.ansible_vault_pass
chmod 600 ~/.ansible_vault_pass
```

`ansible.cfg` already points to it via `vault_password_file`.

To view or edit an encrypted file:

```bash
ansible-vault edit playbooks/host_vars/<host>/vault.yml
```

## Structure

- `roles/bootstrap` — base packages, ufw, unattended-upgrades
- `roles/deploy-user` — dedicated system user + `/srv/<user>` directory
- `roles/nodejs` — Node.js via NodeSource
- `roles/app-deploy` — git clone/update, `.env` templating, build commands
- `roles/systemd-service` — generic systemd service (with optional socket)
- `roles/caddy` — Caddy install + per-site reverse proxy / static file config
- `roles/python` — build specific Python versions from source
- `roles/postgresql` — PostgreSQL via the official PGDG repository

## Running a playbook

Each site has its own playbook under `playbooks/`. The target host must already exist in your `~/.ssh/config` with the same alias used in `inventory/hosts.yml`, including `ForwardAgent yes` (needed for git clone of private repos).

```bash
ansible-playbook playbooks/mang.yml --ask-vault-pass
```

Each role in a playbook is tagged with its own name, so you can run a subset without touching the file:

```bash
ansible-playbook playbooks/mang.yml --tags bootstrap
ansible-playbook playbooks/mang.yml --tags app-deploy,systemd-service
ansible-playbook playbooks/mang.yml --skip-tags caddy
```
