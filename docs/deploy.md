# Rebuild

This article lists the order the environment was stood up in, what each step produced, and how it ended. The homelab is gone, so read it as a record of the sequence, not a runnable guide.

> [!NOTE]
> The moved reference docs ([Terraform](07-terraform.md), [Ansible](06-ansible.md)) still mention paths from the old application repository, such as `Animal-Shelter-Workshop/infrastructure/ansible`. The same folders now live at `infrastructure/` in this repository.

## 1. Prerequisites

- A Proxmox node, an API token, and an Ubuntu 24.04 cloud-init template (VM 9000)
- Terraform and Ansible on a control machine (it was WSL)
- A Tailscale reusable auth key
- The Terraform state backend reachable. State lived in MinIO on `linux-mini-io`, a VM that was not always running (`qm status 109`, then `qm start 109`)
- A Vault AppRole for Ansible. The Vault root token was never used for deploys

Details for each are in [Terraform](07-terraform.md#2-one-time-prerequisites).

## 2. Provision the machines

```bash
cd infrastructure/terraform
export AWS_ACCESS_KEY_ID=<MinIO user>
export AWS_SECRET_ACCESS_KEY=<MinIO secret>

terraform init
terraform plan      # read every line before applying
terraform apply
```

- Containers and VMs were applied as two separate commands. Together they hit a Proxmox lock timeout ([Troubleshooting](troubleshooting.md#3-proxmox-lock-timeout-on-apply)).
- The real fleet was **adopted with `terraform import`**, not created. Its resources are in `production-vms.tf` and `containers.tf`.
- The `test-` prefixed resources in `vms.tf` were a disposable loop, created and destroyed by `provision-test-loop.sh` and `destroy-test-loop.sh`.

## 3. Configure with Ansible

```bash
cd infrastructure/ansible
export VAULT_ROLE_ID=<AppRole role id>
export VAULT_SECRET_ID=<AppRole secret id>

ansible-playbook -i inventory-ip.yml playbooks/site.yml
```

- `site.yml` runs the five database playbooks and `app-server.yml` in dependency order.
- A single host can be run on its own, for example `playbooks/linux-mysql.yml`, or the whole database group with `--limit databases`.
- Tailscale addresses change when a machine is rebuilt, so `resolve_tailscale_ips.py` regenerates the inventory ([Ansible](06-ansible.md#31-update-tailscale-addresses)).

## 4. Harden and verify

Per host: bind the database to its mesh address, restrict the firewall to the app tier, enable auto-start, and set the Proxmox start order. The verification checklist is in [Hardening](02-hardening.md#6-verification-checklist).

## 5. Create the databases

Each of the five servers carries `workshop_2_prod` and `workshop_2_dev`, with matching users. The prod pair is created by the Ansible playbooks; the dev pair is a manual, local-development convenience. Passwords came from Vault, not from files in this repository.

## 6. Move the app to k3s

1. Apply the manifests in `k8s/` to the cluster.
2. Create the ArgoCD Application pointing at the `k8s/` folder, and add the webhook for instant sync.
3. A new image tag in `k8s/deployment.yaml` and `k8s/nginx-deployment.yaml` triggers a rollout.

The manifests inject secrets with the Vault Agent, so none are stored in the cluster's Secret objects. The one Secret in `k8s/app-secret.yaml` holds only a development app key.

## 7. Observability

`linux-observability.yml` installs Prometheus, Grafana, Loki and Alertmanager. `node-exporter-fleet.yml` and `promtail-fleet.yml` put the agents on every host, and the cluster runs kube-state-metrics and Grafana Alloy.

## 8. How it ended

The homelab was taken down, and this repository was kept as documentation. Before that, the old database VMs had already been left powered off, not destroyed, as a rollback net. The Terraform drift check that watched the fleet daily is kept as [`examples/terraform-drift.yml`](examples/terraform-drift.yml), which is not wired to any scheduler.

To rebuild on Azure instead, see [Azure mapping](azure-mapping.md).
