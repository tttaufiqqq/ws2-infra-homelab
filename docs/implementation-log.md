# Implementation log

This log records how the infrastructure was planned, built and changed. Dates come from the commit history of the moved paths (2026-06-29 to 2026-09-01). It lists what happened and why each decision was made.

## 1. Goal

Run a five-database, three-engine Laravel app on a personal Proxmox homelab as if it were production, with these requirements:

- Every database on its own machine, reachable only over a private network.
- Every machine rebuildable from code.
- Deploys by commit, secrets out of files, and backups that are proven to restore.

## 2. Decisions

| Decision | Choice | Reason |
|---|---|---|
| IaC | Terraform (`bpg/proxmox`) plus Ansible | Terraform makes machines, Ansible configures them |
| Network | Tailscale, replacing SSH tunnels | No database port on the public internet |
| Terraform state | Self-hosted MinIO | Shared state without a cloud account |
| Existing machines | `terraform import` | They held real data |
| Database hosts | VMs, then containers | VMs here pinned all declared RAM |
| App tier | Single VM, then k3s with ArgoCD | Rolling deploys, replicas, GitOps |
| Secrets | `.env`, then Vault AppRole, then Vault Agent | Each step removed a place a secret could leak |

## 3. Timeline

| Date | Phase | What happened |
|---|---|---|
| 2026-06-29 | Scaffold | Terraform and Ansible for 4 VMs; fixed storage, disk format, VM IDs and names; UFW enforced by Ansible |
| 2026-07-19 | Production hardening | Deploy playbook stopped wiping data; sessions, cache and queue moved off the `users` database; scheduler on a systemd timer; HTTPS via Let's Encrypt; real SMTP; log rotation; idle Postgres sessions auto-terminated |
| 2026-07-20 | Backups and Vault | Per-host dumps replaced by coordinated backup and restore, with a completed restore drill; secrets moved to a scoped Vault AppRole, then to a Vault Agent |
| 2026-07-20 | Five-way split | Every connection given its own physical host; prod and dev databases split; a stored-routine `DEFINER` outage found and fixed |
| 2026-07-21 | CD | Automated deploy on green tests through a self-hosted runner |
| 2026-07-26 | Terraform and Ansible stages | First real apply unblocked; state moved to MinIO; two containers adopted; Ansible refactored into roles with idempotency checks; drift check added; offsite backups to Azure Blob |
| 2026-07-26 | k3s and observability | First k3s manifests; node exporters, Promtail, Prometheus, Grafana, Loki and Alertmanager |
| 2026-07-27 | GitOps and Azure | ArgoCD sync tested by bumping replicas to 3; Azure stretch VM created, proven and destroyed |
| 2026-07-28 | Fleet import | The rest of the live fleet (5 VMs, 5 CTs) imported; scope narrowed back to this app; container creation proven with 2 disposable containers |
| 2026-08-01 | Test loop | Provision and teardown scripts for the scratch loop; a hostname-collision bug in the IP resolver fixed |
| 2026-08-02 | VM to CT and k3s cutover | `linux-mysql`, `linux-mariadb`, `linux-postgres` moved from VMs to CTs; Tailscale netfilter found bypassing UFW and fixed; app split into app and nginx Deployments; Cloudflare Tunnel, Redis and Alloy added |
| 2026-08-30 | Monitoring drills | Kubernetes object state added to Prometheus; real database connectivity probes; nginx config deliberately broken and reverted to prove alerts |
| 2026-09-01 | Alert tuning | `HighDiskUsage` split into warning and critical tiers |
| 2026-11 | Archive | Infrastructure extracted from the application repository, with history; homelab taken down; documentation restyled |

## 4. Problems found, by phase

### 4.1 Adopting running machines

Provider defaults for `on_boot`, `started`, `scsi_hardware` and `cdrom` silently differ from a real host, so a first `plan` wanted to change healthy machines. Each was declared explicitly to match reality. Full detail in [Terraform](07-terraform.md#importing-the-real-production-vmscts-three-proxmox_virtual_environment_vm-defaults-that-silently-diverge-from-reality).

### 4.2 Secrets

The first Vault integration still left passwords in `.env` and cached config. Four real bugs surfaced while deploying it: mismatched env names for the payment gateway, two DB hosts pointing at a host with no database, a missing filesystem disk setting, and a missing marker file that would have reseeded demo data over live data. Detail in [Production hardening](09-production-hardening.md#secrets-moved-to-vault).

### 4.3 Firewalls

After the VM-to-CT move, Tailscale's own netfilter rules bypassed UFW on the database hosts. A hardening fix then broke CI's database access, which was restored the same day. The lesson: test the deny rule, not just the allow rule.

### 4.4 Rollouts on a small node

With three replicas and a Vault injector sidecar already using about 95% of one CPU, a default rolling update could never schedule its surge pod. Setting `maxSurge: 0` made it retire one pod before creating the next.

### 4.5 Monitoring

The alerts were validated by deliberately breaking an nginx config, watching them fire, and reverting it, twice. A real early warning came from the CI runner: Docker build caches piled up on its small root disk and tripped the disk alert, which led to pruning after every build and the two-tier disk alert.

## 5. Outcome

The environment ran the app on k3s behind ArgoCD, with five isolated databases, Vault-injected secrets, verified backups and working alerts. It was then taken down, and the code and write-ups were kept here.
