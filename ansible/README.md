# EpicBook deployment (Ansible + Azure DevOps)

DMI Week 10, Assignment 04 — **Kuntal Tarwatkar**. Infrastructure comes from the separate
[`infra-epicbook`](https://github.com/tarwatkarkuntal315/infra-epicbook) repository (Terraform).

```text
Browser → frontend VM (Nginx :80) → backend VM private IP (EpicBook :8080) → Azure Database for MySQL (private, TLS)
```

## Layout

```text
ansible/
  ansible.cfg
  inventory.ini            frontend + backend hosts (Terraform handoff: public management addresses)
  group_vars/all.yml       backend_private_ip, mysql_fqdn (handoff) + non-secret app settings
  requirements.yml         collections (none needed — builtin modules only)
  site.yml                 handoff/secret checks → common → database → backend → frontend
  verify.yml               service, port, Nginx, public HTTP 200, tables + seed data
  roles/
    common/                apt cache, baseline packages, world-writable check
    database/              MySQL client, root-only option file, DNS + port checks, idempotent schema/seed import
    backend/               Node.js, epicbook system user, git deploy at the pipeline commit, npm ci, env file, systemd unit
    frontend/              Nginx reverse proxy to the backend PRIVATE IP, config test before reload
    verify/                non-sensitive post-deployment checks
pipeline/ansible-setup.yml shared job setup: Ansible venv, Secure File key, job-scoped known_hosts
azure-pipelines.yml        Prepare → Validate → Deploy → Verify
```

## Manual handoff (from the infra pipeline's Outputs stage)

| Terraform output | Where it goes |
|------------------|---------------|
| `app_public_ip` | `inventory.ini` (frontend `ansible_host`) and `group_vars/all.yml` |
| `backend_ansible_host` | `inventory.ini` (backend `ansible_host`) |
| `backend_private_ip` | `group_vars/all.yml` — Nginx upstream |
| `mysql_fqdn` | `group_vars/all.yml` — database host |

The playbook refuses to run while any `REPLACE_WITH_…` placeholder remains.

## Secrets

| Secret | Stored in | How it reaches the servers |
|--------|-----------|----------------------------|
| SSH private key | Azure DevOps **Secure Files** (`epicbook_id_rsa`) | `DownloadSecureFile@1` to the job temp dir, `chmod 600`, passed with `--private-key` |
| MySQL admin username/password | Variable group **`epicbook-secrets`** (password secret) | mapped to env vars → `lookup('env')` → root-only files on the backend, tasks use `no_log` |

On the backend, `/etc/epicbook/mysql-client.cnf` (0600 root) is used for database tasks and
`/etc/epicbook/epicbook.env` (0640 root:epicbook) holds `JAWSDB_URL` for the systemd service.
EpicBook connects to MySQL over TLS (`config/config.json` → `dialectOptions.ssl`).
