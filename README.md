# ws2-infra

Infrastructure for the Animal Shelter Workshop (Workshop II, UTeM) app, split out of
[`Animal-Shelter-Workshop`](https://github.com/tttaufiqqq/Animal-Shelter-Workshop).
History for every moved path was preserved with `git filter-repo`.

| Path | What it is |
|---|---|
| `infrastructure/terraform/` | Proxmox VMs/CTs (state in MinIO on `linux-mini-io`) |
| `infrastructure/terraform-azure/` | Azure `azurerm` proof-then-destroy stretch config |
| `infrastructure/ansible/` | Host configuration for the DB fleet, Vault, runner, observability |
| `infrastructure/*.sh`, `*.py` | Disposable test-loop scripts, Tailscale IP resolver |
| `k8s/` | k3s manifests synced by ArgoCD |
| `.github/workflows/terraform-drift.yml` | Daily read-only drift check |
| `docs/` | Ansible, Terraform, hardening, backups write-ups |

## Not here (stays in the app repo)

Application code, `Dockerfile`, `docker-compose.yml`, `tests.yml`, and the CI/CD docs
(`11-ci.md`, `12-cd.md`). `deploy.yml` also stays there until the cutover is done.

## Secrets

No credentials live in this repo. `terraform.tfvars` and `*.tfstate` are gitignored; use the
`*.example` files. Real values come from Vault or the environment.

## Status

Prepared locally, not yet pushed or wired up. See [`docs/CUTOVER.md`](docs/CUTOVER.md).
