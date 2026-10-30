# eXamc.ops

Ansible + Vault deployment of [eXamc](https://github.com/EPFL-CePro/eXamc).

This project generates each environment's configuration and secrets, and deploys the Docker Compose stack, for **dev** (your machine), **staging** and **prod**.

> [!NOTE]
> For the application itself (Docker images, Compose files, Makefile, OIDC, database), see the `README.md` of the [main eXamc repository](https://github.com/EPFL-CePro/eXamc).

## Overview

- 🔐 **Secrets** live in an encrypted **Ansible Vault**, one per environment, and are deployed as **files** (Compose secrets, mounted at `/run/secrets/<name>`). They are never written to a `.env`, and never printed: the run only shows their names.
- 🧩 **The `.env`** only holds what differs from the defaults in `compose/compose.base.yml`. It's rendered from a Jinja2 template.
- 👤 **On servers**, the app runs as a dedicated `examc` user (UID/GID 1500, no login), which owns the private media. In dev, it runs as you.
- 🗂️ **Each inventory brings its own variables**: choosing the inventory (`-i inventory/<env>`) selects the environment.
- 🚀 **One playbook** for all three environments: it writes the secrets and the `.env`, checks out the code, logs in to Docker Hardened Images, and starts the stack, waiting until every service is healthy.

## Layout

| Purpose                                 | Path                                                                            |
|-----------------------------------------|---------------------------------------------------------------------------------|
| Playbook (shared variables + steps)     | `playbook.yml`                                                                  |
| Steps                                   | `tasks/app_user.yml`, `tasks/secrets.yml`, `tasks/code.yml`, `tasks/docker.yml` |
| `.env` template                         | `templates/.env.j2`                                                             |
| Redis ACL template                      | `templates/redis_acl.j2`                                                        |
| Hosts (in an `[app]` group)             | `inventory/<env>/hosts.ini`                                                     |
| Settings per environment (incl. `env`)  | `inventory/<env>/group_vars/all/app.yml`                                        |
| Secrets per environment (Vault)         | `inventory/<env>/group_vars/all/vault.yml`                                      |

```
inventory/
├── dev/
│   ├── hosts.ini
│   └── group_vars/all/
│       ├── app.yml        # env: dev, branch and checkout from the command line, your user
│       └── vault.yml
├── staging/   (same layout)
└── prod/      (same layout)
```

Ansible loads `group_vars/` next to the inventory automatically, vault files included. Variables in `group_vars/all/` apply to every host of that inventory.

## Variables

**Shared** (same in every environment), in the `vars:` of `playbook.yml`: `repo_url`, and `env_path` (`<app_dir>/.env.<env>`).

**Per environment**, in `inventory/<env>/group_vars/all/app.yml`:

| Variable         | staging / prod                         | dev                                     |
|------------------|----------------------------------------|-----------------------------------------|
| `env`            | `staging` / `prod`                     | `dev`                                   |
| `repo_branch`    | `staging` / `main`                     | `-e branch=…`                           |
| `app_dir`        | `/opt/examc`                           | `-e checkout=…`                         |
| `media_dir`      | `/srv/examc/private_media`             | `<checkout>/compose/data/private_media` |
| `secrets_dir`    | `/srv/examc/secrets`                   | `<checkout>/compose/secrets`            |
| `app_user`       | `examc`                                | you (`$USER`)                           |
| `app_uid`        | `1500` (must match the images' UID)    | your UID, from Ansible's facts          |
| `compose_file`   | `compose.prod.yml`                     | `compose.dev.yml`                       |
| `caddy_tls_mode` | `internal` / `certs`                   | `internal`                              |

> [!IMPORTANT]
> Don't move per-environment settings into the playbook's `vars:`. Play variables take precedence over the inventory's `group_vars`, so an environment could no longer change them.

If `env` or one of the paths isn't defined, the playbook stops at its first task, before changing anything.

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
ansible-vault edit inventory/<env>/group_vars/all/vault.yml --vault-id @prompt
```

- **Adding a secret** accessible to services means adding a key to `vault_secrets`; removing or renaming one deletes the old file on the next run.
- **Every secret the Compose files mount must be in the vault.** A missing one makes `docker compose up` fail with `bind source path does not exist`, after the old containers have been stopped.
- **On servers**, files are in `/srv/examc/secrets`: the folder is root only (`0700`) and the files are `0444`, so each container can read its own whatever its user (1500 for the app, 65532 for MySQL and Redis). In dev, they're in `compose/secrets/` of your checkout, owned by you.
- **The run shows which secret files changed**, by name only. Their content is hidden, also on errors and with `-vvv`.
- **With `caddy_tls_mode: certs`** (prod), the playbook checks the certificate and key with `openssl` on the server before writing anything, so a broken or mismatched pair stops the run.

## Running

Always start with a dry run. It shows what would change, without changing anything:

```bash
ansible-playbook playbook.yml -i inventory/<env> --check --diff --vault-id @prompt      # + -e branch=… -e checkout=… in dev
```

Then the real run:

| Target             | Command                                                                                                                      |
|--------------------|------------------------------------------------------------------------------------------------------------------------------|
| dev (your machine) | `ansible-playbook playbook.yml -i inventory/dev --vault-id @prompt -e branch=staging -e checkout=~/Documents/Projets/eXamc` |
| staging            | `ansible-playbook playbook.yml -i inventory/staging --vault-id @prompt`                                                      |
| prod               | `ansible-playbook playbook.yml -i inventory/prod --vault-id @prompt`                                                         |

No `-e env=…`: the inventory's `app.yml` sets it. In dev, `branch` and `checkout` are required: without them, the playbook stops at once and shows the command. Staging and prod always deploy their own `repo_branch` and ignore both.

On staging and prod, Ansible connects over SSH as `root` and the play runs with `become`; dev runs locally as you.

To check what an environment loads, without running anything:

```bash
ansible-inventory -i inventory/<env> --graph                         # the hosts must be under @app
ansible-inventory -i inventory/<env> --vault-id @prompt --list | jq '._meta.hostvars[] | {env, app_dir, media_dir, secrets_dir}'
```

What the playbook does:

1. **Checks** that the environment's variables are loaded (and, in dev, that `branch` and `checkout` were given), and shows dev's paths for the run.
2. **App user** (servers only): creates the `examc` group and user, and gives `media_dir` and its contents to them (only the files that don't already belong to them).
3. **Secrets:** checks the TLS pair (certs mode), writes one file per vault secret plus the Redis ACL, lists the names of the files it changed, and removes files no longer in the vault.
4. **Code and `.env`:**
   - **Servers:** checks out `repo_branch`, discarding any local change.
   - **Dev:** clones the repository into `checkout` if it isn't a checkout yet (the folder must be missing or empty), otherwise switches your checkout to `branch` with `git switch`. Uncommitted changes are kept; if they conflict with the branch, the run stops without changing anything.
   - Then renders the `.env` at `<app_dir>/.env.<env>`.
5. **Stack:** logs in to `dhi.io`, then runs `docker compose up` (`--build` in dev, `--pull always` on servers) with `--force-recreate --wait`. If a service doesn't become healthy, the playbook prints the Compose output, the status of every service and their recent logs, then fails.

On servers, the images come from the [packages of the eXamc repository](https://github.com/orgs/EPFL-CePro/packages?repo_name=eXamc) (`staging` and `stable` tags).

## Managing the stack by hand

On a server, use the same project name, env file and Compose file as the playbook, or Compose creates a second set of containers:

```bash
cd /opt/examc
alias dc='docker compose -p examc-prod --env-file .env.prod -f compose/compose.prod.yml'
dc ps -a
dc logs --tail 50 django
```

`docker restart <container>` restarts a container without recreating it. Anything that recreates it (`dc up`, or a new deploy, which always uses `--force-recreate`) brings back the image's files, so a fix made inside a container must also be committed.

## Dev specifics

Everything points at the checkout you pass with `-e checkout=…`: secrets in `compose/secrets/`, private media in `compose/data/private_media/`, the `.env` at `.env.dev`. The app runs as you, with your own UID, and Caddy serves it on <https://localhost:8443>.

```bash
# Your usual checkout, on staging
ansible-playbook playbook.yml -i inventory/dev --vault-id @prompt -e branch=staging -e checkout=~/Documents/Projets/eXamc

# A fresh clone of a feature branch in a new folder
ansible-playbook playbook.yml -i inventory/dev --vault-id @prompt -e branch=my-feature -e checkout=~/code/eXamc-my-feature
```

After the first run, you can start and stop the stack from the main repository as usual. Rerun the playbook when secrets or settings change, or to switch branch.