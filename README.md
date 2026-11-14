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

## report_hub (`playbooks/report_hub.yml`)

report_hub (github.com/mang-srl/report-hub) on the host `mang`, beside mang's site:
https://report.mangsrl.com (app) and https://content.report.mangsrl.com (report files). Its own
user `report_hub`, everything under `/srv/report_hub` (`app/` the code, `venv/` Python 3.14.7
from `/opt/python`, `storage/` the reports), its own database, two services
(`report_hub` = gunicorn on 127.0.0.1:8000, `report_hub-notifications` = the emails), one Caddy
file (`/etc/caddy/sites/report.mangsrl.com.caddy`, template `roles/caddy/templates/report_hub.caddy.j2`).

### Before the first run

1. The realm `mang` on Keycloak: `~/Systemadmin/keycloak/setup_mang.py` (its README). It writes
   `~/Systemadmin/keycloak/mang-secrets.json`, which this playbook reads.
2. DNS: `report` and `content.report` in the zone `mangsrl.com`, CNAMEs to `vps.mangsrl.com`.
3. In the vault of `mang` (`ansible-vault edit playbooks/host_vars/mang/vault.yml`):
   `vault_report_hub_db_password`. The SMTP keys (`vault_smtp_*`) are mang's site's, reused:
   report_hub logs in as `contact@` and sends as `noreply@mangsrl.com`, an alias.
4. The code pushed to the branch `develop` (the server clones it from GitHub).

Django's `SECRET_KEY` is generated at the first run into `.secrets/report_hub_secret_key`
(git-ignored) and reused afterwards: keep that file (and `mang-secrets.json`) with the vault's
password.

### First installation, in this order

```bash
ansible-playbook playbooks/report_hub.yml --tags deploy-user --ask-vault-pass
ansible-playbook playbooks/report_hub.yml --tags python --ask-vault-pass        # ~15 min, also upgrades the system's packages
ansible-playbook playbooks/report_hub.yml --tags postgresql,postgresql-db --ask-vault-pass
ansible-playbook playbooks/report_hub.yml --tags report-hub-config --ask-vault-pass   # restarts Caddy (mang's site too, ~2 s)
ansible-playbook playbooks/report_hub.yml --tags app-deploy --ask-vault-pass
ansible-playbook playbooks/report_hub.yml --tags systemd-service --ask-vault-pass
ansible-playbook playbooks/report_hub.yml --tags caddy --ask-vault-pass
```

Or all at once: `ansible-playbook playbooks/report_hub.yml --ask-vault-pass`.

`report-hub-config` also puts the user `caddy` in the group `report_hub`: Caddy serves the
report files itself (X-Accel-Redirect) and must read them (folders 750, files 640).

### Once, by hand (data of the app, not configuration)

The commands run on the server as `report_hub`, with the production settings:

```bash
ssh mang 'cd /srv/report_hub/app/backend && sudo -u report_hub env DJANGO_SETTINGS_MODULE=backend.settings_prod /srv/report_hub/venv/bin/python manage.py COMMAND'
```

with COMMAND, in order:

1. `create_organization mang MANG --domain report.mangsrl.com --realm mang --sender-email noreply@mangsrl.com`
2. Log in once at https://report.mangsrl.com (the Keycloak role `django-superuser` makes the
   account a Django superuser), then
   `add_organization_admin mang boleo88@gmail.com`
3. On https://report.mangsrl.com/organization: logo `~/report_hub-assets/mang/mang-full.png`,
   icon `~/report_hub-assets/mang/mang-m-circle-512.png`, and the colour (the logo's blue).
4. The public demos (https://report.mangsrl.com/demo), after copying the zips:
   `scp ~/Projects/research/demo_report/demo_report_v{1,2}_{single,multi}.zip mang:/tmp/`
   - `prepare_demo mang --staff boleo88@gmail.com --title "Demo single page" --zip /tmp/demo_report_v1_single.zip --zip /tmp/demo_report_v2_single.zip --public`
   - `prepare_demo mang --staff boleo88@gmail.com --title "Demo multi-page" --zip /tmp/demo_report_v1_multi.zip --zip /tmp/demo_report_v2_multi.zip --public`

### A new version of report_hub

Push to `develop`, then:

```bash
ansible-playbook playbooks/report_hub.yml --tags app-deploy --ask-vault-pass
```

It pulls, rebuilds (venv, frontend, annotator), collects the static files, migrates, checks
the production settings and restarts both services.
