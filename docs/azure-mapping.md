# Azure mapping

This article maps each part of the homelab to an Azure service, as the plan for rebuilding it in Azure. It is a design note, not a deployed result.

## 1. Mapping

| Homelab | Azure |
|---|---|
| Proxmox VMs and CTs, Terraform `bpg/proxmox` | `azurerm` resources; start from `infrastructure/terraform-azure/` |
| PostgreSQL host | Azure Database for PostgreSQL Flexible Server |
| MySQL hosts | Azure Database for MySQL Flexible Server |
| MariaDB hosts | No managed MariaDB. Use MySQL Flexible Server, or a VM |
| k3s plus ArgoCD | AKS plus ArgoCD (the `k8s/` manifests mostly carry over) or Container Apps |
| Docker Hub | Azure Container Registry |
| Vault plus Agent injection | Key Vault with Workload Identity |
| Tailscale mesh | VNet with private endpoints and NSGs |
| Cloudflare Tunnel | Application Gateway or Front Door, or keep the tunnel |
| Redis pod | Azure Cache for Redis |
| MinIO Terraform state | `azurerm` backend on Blob Storage |
| Self-hosted runner | GitHub Actions with OIDC to Azure |
| Prometheus, Grafana, Loki | Azure Monitor and Log Analytics |
| Nightly dump sync to Blob | Already uses Blob Storage with a SAS token |

## 2. Hard parts

- **MariaDB.** The `booking` module relies on MariaDB stored procedures and triggers. Moving to MySQL means checking them one by one.
- **One database per machine.** Five managed servers is the main cost. One PostgreSQL server and one MySQL server, each holding several databases, would loosen the isolation rule but cut the bill.
- **Cross-database consistency.** Nothing in Azure changes this; it stays in the application.
- **Secrets.** The Vault Agent annotations in `k8s/` would be replaced by Workload Identity and the Key Vault CSI driver.

## 3. Cost and capacity

- Azure Firewall, managed databases and AKS nodes bill while they exist, so build in small stages and destroy each one.
- Capacity is a per-region, per-SKU limit. The stretch config already hit `SkuNotAvailable` in `southeastasia` ([Troubleshooting](troubleshooting.md#5-azure-sku-not-available)). Check quotas before choosing a region.

## 4. Suggested stages

1. Networking and a resource group, with a remote Blob state backend.
2. One PostgreSQL and one MySQL Flexible Server on private endpoints.
3. Container registry and the app on AKS or Container Apps.
4. Key Vault and Workload Identity.
5. CI with OIDC, then monitoring.

Each stage should end with `terraform destroy`, as the existing stretch config did.
