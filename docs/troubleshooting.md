# Troubleshooting

This article lists problems met while building the environment and how each was resolved. Fuller write-ups are in [Terraform lessons](terraform-lessons.md) and [Production hardening](09-production-hardening.md).

## 1. Read the plan before applying

```bash
terraform plan
terraform state list
terraform state show <address>
```

On an imported machine, an unexpected change in the first `plan` usually means a provider default differs from the real host. Fix the declaration; do not apply to "see what happens".

## 2. First plan after import wants to change healthy machines

**Symptom:** after `terraform import`, `plan` shows `on_boot`, `started`, `scsi_hardware` or `cdrom` changing.

**Fix:** the VM resource defaults these away from reality. Declare them to match the host: `on_boot` and `started` to the real state, `scsi_hardware = "virtio-scsi-single"`, and `cdrom` with `interface = "ide2"`. A machine that was powered off must be imported with `started = false`, or the first apply boots it.

## 3. Proxmox lock timeout on apply

**Symptom:** applying VMs and containers together fails with `can't lock file '/run/lock/lxc/pve-config-<id>.lock' - got timeout`, and Terraform marks the containers tainted.

**Fix:** the containers were created fine; only the status polling hit lock contention while four VM clones saturated the host. Run `terraform untaint` on them instead of re-applying, and apply VMs and containers as two separate commands.

## 4. A plan diff that never goes away

**Symptom:** `disk.file_format: "raw" -> "qcow2"` appears on every plan, even right after applying it.

**Fix:** Proxmox's clone does not preserve that attribute. Add `disk[0].file_format` to the module's `lifecycle.ignore_changes`.

## 5. Azure SKU not available

**Symptom:** the Azure stretch VM fails with `SkuNotAvailable` or a capacity restriction for `Standard_B1s`, `B1ms` and `B2s` in `southeastasia`.

**Fix:** capacity is a regional, per-SKU limit on the subscription. The config uses `Standard_D2s_v3`, which applied. It costs more per hour, so the VM was meant to be destroyed straight after the proof. Check availability first with `az vm list-skus`.

## 6. Storage, datastore and node names

**Symptom:** `qm importdisk` or `terraform apply` fails on storage.

**Fix:** the node had a single Directory storage called `local`, not `local-lvm`. Set `storage_pool = "local"`, `file_format = "qcow2"` (Directory storage needs it), and `datastore_id` in the cloud-init `initialization` block. The node name was the host's real name, not the example's `pve`.

## 7. Terraform will not install on a new Ubuntu

**Symptom:** the HashiCorp apt install script fails silently on Ubuntu 25.04.

**Fix:** `sudo snap install terraform --classic`.

## 8. Ansible cannot connect with the Terraform key

**Symptom:** Terraform succeeds but Ansible is refused.

**Fix:** the `ssh_public_key` in `terraform.tfvars` must match the private key Ansible actually uses, which is whatever sits at `~/.ssh/id_ed25519` on the control node. SSH usernames also differ per host: `~/.ssh/config` is the source of truth, not the docs.

## 9. Database reachable when it should not be

**Symptom:** a database port answers from a host that is not the app tier.

**Fix:** Tailscale's netfilter rules can bypass UFW. After changing a firewall, test both the allow and the deny from a machine that should be refused.

## 10. Stored routines fail after splitting prod and dev users

**Symptom:** actions fail with `execute command denied to user 'workshop_2'@'%'`, even though the app connects as `workshop_2_prod`.

**Fix:** the prod database was populated by dump-restoring the old shared one, which kept every routine's `DEFINER=workshop_2@%`. Under the default `SQL SECURITY DEFINER` the privilege check runs as that old user, which had no access, so all 121 routines on the four MySQL and MariaDB hosts failed. Procedures and functions were switched to `SQL SECURITY INVOKER`, and triggers were re-created with the new definer. See [Production hardening](09-production-hardening.md#29-stored-routine-privileges-after-the-proddev-split).

## 11. Rolling update stuck pending

**Symptom:** a Deployment update leaves a new pod `Pending` on a single-CPU node.

**Fix:** the default surge pod can never fit. Use `maxSurge: 0` and `maxUnavailable: 1`, so one pod retires before the next starts.

## 12. Disk alert on the CI runner

**Symptom:** `HighDiskUsage` fires on the runner after a handful of pushes.

**Fix:** every build created two fresh images that were never reused. Prune images and the build cache after each build, and alert on two tiers, warning and critical.
