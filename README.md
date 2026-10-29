# eXamc.ops

Ansible + Vault deployment of [eXamc](https://github.com/EPFL-CePro/eXamc).

This project generates each environment's configuration and secrets, and deploys the Docker Compose stack, for **dev** (your machine), **staging** and **prod**.

> [!NOTE]
> For the application itself (Docker images, Compose files, Makefile, OIDC, database), see the `README.md` of the [main eXamc repository](https://github.com/EPFL-CePro/eXamc).

## Overview

- 🔐 **Secrets** live in an encrypted **Ansible Vault**, one per environment, and are deployed as **files** (Compose secrets, mounted at `/run/secrets/<name>`). They are never written to a `.env`.
- 🧩 **The `.env`** only holds what differs from the defaults in `compose/compose.base.yml`. It's rendered from a Jinja2 template.
- 👤 **On servers**, the app runs as a dedicated `examc` user (UID/GID 1500, no login), which owns the private media.
- 🚀 **One playbook** for all three environments: it writes the secrets and the `.env`, logs in to Docker Hardened Images, and starts the stack, waiting until every service is healthy.

## Layout

| Purpose                             | Path                                                                            |
|-------------------------------------|---------------------------------------------------------------------------------|
| Playbook (variables + steps)        | `playbook.yml`                                                                  |
| Steps                               | `tasks/app_user.yml`, `tasks/secrets.yml`, `tasks/code.yml`, `tasks/docker.yml` |
| `.env` template                     | `templates/.env.j2`                                                             |
| Redis ACL template                  | `templates/redis_acl.j2`                                                        |
| Non-secret settings per environment | `group_vars/<env>/app.yml`                                                      |
| Secrets (Vault-encrypted)           | `group_vars/<env>/vault.yml`                                                    |
| Inventory (host + `env`)            | `inventory/<env>/hosts.ini`                                                     |

## Secrets

Each environment's `vault.yml` contains:

```yaml
vault_secrets:            # one file per key in SECRETS_DIR
  secret_key: "..."
  mysql_root_password: "..."
  mysql_password: "..."
  redis_password: "..."   # hex only: it goes inside URLs
  oidc_rp_client_id: "..."
  oidc_rp_client_secret: "..."
  tenant_id: "..."
  oasis_base_url: "..."
  oasis_bearer: "..."
  tls_crt: |              # prod: certificate + intermediates; staging/dev: placeholder
    -----BEGIN CERTIFICATE-----
    ...
  tls_key: |
    -----BEGIN PRIVATE KEY-----
    ...
vault_dockerhub_username: "..."   # Docker Hub account for dhi.io
vault_dockerhub_token: "..."
```

```bash
ansible-vault edit group_vars/<env>/vault.yml --vault-id @prompt
```

- **Adding a secret** accessible to services means adding a key to `vault_secrets`; removing or renaming one deletes the old file on the next run.
- **On servers**, files are in `/srv/examc/secrets` (root only), each `0400` and owned by the service that reads it (`secret_owners` in `playbook.yml`). In dev, they're in `compose/secrets/` of your checkout, owned by you.
- **With `caddy_tls_mode: certs`** (prod), the playbook checks the certificate and key with `openssl` before writing anything, so a broken or mismatched pair stops the run.

## Running

Always start with a dry run. It shows what would change, without changing anything (secret contents are never printed):

```bash
ansible-playbook playbook.yml -i inventory/<env>/hosts.ini --check --diff --vault-id @prompt
```

Then the real run:

| Target             | Command                                                                           |
|--------------------|-----------------------------------------------------------------------------------|
| dev (your machine) | `ansible-playbook playbook.yml -i inventory/dev/hosts.ini --vault-id @prompt`     |
| staging            | `ansible-playbook playbook.yml -i inventory/staging/hosts.ini --vault-id @prompt` |
| prod               | `ansible-playbook playbook.yml -i inventory/prod/hosts.ini --vault-id @prompt`    |

Each inventory sets `env` (`dev`, `staging` or `prod`), which selects the matching `group_vars/<env>/`. On staging and prod, Ansible connects over SSH as `root`; dev runs locally as you.

What the playbook does:

1. **App user** (servers only): creates the `examc` group and user, and gives `/srv/examc/private_media` and its contents to them.
2. **Secrets:** checks the TLS pair (certs mode), writes one file per vault secret plus the Redis ACL, and removes files no longer in the vault.
3. **Code and `.env`:** checks out `repo_version` (servers only; your dev checkout is never touched) and renders the `.env`.
4. **Stack:** logs in to `dhi.io`, then runs `docker compose up` (`--build` in dev, `--pull always` on servers) with `--wait`. If a service doesn't become healthy, the playbook prints the Compose output, the status of every service and their recent logs, then fails.

On servers, the images come from the [packages of the eXamc repository](https://github.com/orgs/EPFL-CePro/packages?repo_name=eXamc) (`staging` and `stable` tags).

## Dev specifics

`group_vars/dev/app.yml` points everything at your checkout (`~/Documents/Projets/eXamc`): secrets in `compose/secrets/`, private media in `compose/data/private_media/`, your own user instead of `examc`, and Caddy on <https://localhost:8443>. Adjust `app_dir` if your checkout lives elsewhere.

After the first run, you can start and stop the stack from the main repository as usual. Rerun the playbook when secrets or settings change.