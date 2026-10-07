# eXamc — Ansible + Vault environment management

This project uses **Ansible** to manage all environment configuration and deployment of [eXamc](https://github.com/EPFL-CePro/eXamc) securely.  
It replaces manual `.env.*` editing while keeping `.env.example` (in the main eXamc repository) as a public reference.

> [!NOTE]
> For the general project setup (Docker, Makefile, OIDC, DB, etc.), see the `README.md` of the [main eXamc repository](https://github.com/EPFL-CePro/eXamc).

---

## Overview

- 🔐 **Secrets** are stored encrypted with **Ansible Vault**.
- 🧩 **Environment files** (`.env.dev`, `.env.staging`, `.env.prod`) are **generated automatically** from a Jinja2 template.
- 🚀 **Local dev**: run `make env ENV=dev` to create `.env.dev`, then `make up ENV=dev`.
- 🌍 **Staging / Prod**: deployed with an Ansible playbook (see below).

The manual approach (copying `.env.example`, described in the *Environment files* section of the `README.md` in the [main eXamc repository](https://github.com/EPFL-CePro/eXamc)) remains available; Ansible is the **preferred** workflow. The generated `.env.*` files must contain the same variables listed there, including the Entra ID / OIDC settings.

---

## Configuration files

| Purpose                       | Path                         |
|-------------------------------|------------------------------|
| Env template (Jinja2)         | `templates/.env.j2`          |
| Non-sensitive per-env vars    | `group_vars/<env>/app.yml`   |
| **Secrets (Vault-encrypted)** | `group_vars/<env>/vault.yml` |
| Inventory                     | `inventory/<env>/hosts.ini`  |
| Deployment playbook           | `playbook.yml`               |

---

## Local development

From the root of the [main eXamc repository](https://github.com/EPFL-CePro/eXamc), generate your local `.env.dev`:

```bash
make env ENV=dev
```

Then start the stack:

```bash
make up ENV=dev
```

> If `.env.dev` is missing, `make up` will prompt you to run `make env` first.  
> Real `.env.*` files remain **untracked** (ignored by Git). Keep `.env.example` for reference.

---

## Deployment

Three deployment targets are supported:

`dev`, for local development:

```bash
ansible-playbook -i inventory/dev/hosts.ini playbook.yml -e env=dev --vault-id @prompt
```

`staging`, for the staging server:

```bash
ansible-playbook -i inventory/staging/hosts.ini playbook.yml -e env=staging --vault-id @prompt
```

`prod`, for the production server:

```bash
ansible-playbook -i inventory/prod/hosts.ini playbook.yml -e env=prod --vault-id @prompt
```

The playbook (`playbook.yml`):

1. Renders `.env` from `templates/.env.j2`,
2. Decrypts Vault values,
3. Runs `docker compose up -d --build` on the target host.

> [!IMPORTANT]
> Migrations are **not** applied automatically at boot on staging/prod — see the *Migrations & updates* section of the `README.md` in the [main eXamc repository](https://github.com/EPFL-CePro/eXamc).