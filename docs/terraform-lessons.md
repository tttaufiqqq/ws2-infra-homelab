# Terraform lessons

This article lists the problems met setting up and importing Proxmox machines with Terraform, and how each was fixed. See [Terraform](07-terraform.md) for the setup.

## 1. Storage is `local`, not `local-lvm`

**Symptom:** `qm importdisk` fails, and `apply` would fail too.

**Fix:** the node has one Directory storage called `local`. Set `storage_pool = "local"` and `file_format = "qcow2"`, which Directory storage requires. Also set `datastore_id = var.storage_pool` in the `initialization` block, because the provider defaults it to `local-lvm`.

## 2. Wrong node name

**Symptom:** the example's `pve` does not match the node.

**Fix:** use the name shown at the top of the Proxmox tree.

## 3. VM IDs already in use

**Symptom:** `VM ID already in use` for 101 and 104 to 106.

**Fix:** use 200-series IDs for Terraform-created VMs, and keep the 100-series as backups.

## 4. SSH key mismatch

**Symptom:** Ansible cannot connect after a successful apply.

**Fix:** `ssh_public_key` must match the private key at `~/.ssh/id_ed25519` on the control node. The key had been overwritten by copying the Windows key into WSL.

## 5. Imported VMs show changes on the first plan

**Symptom:** after importing `app-server` and the database VMs, `plan` wants to change healthy machines.

**Fix:** the VM resource defaults these attributes away from reality. Declare them explicitly.

- `on_boot` and `started` default to `true`, but none of the four VMs had `onboot` set.
- `scsi_hardware` defaults to `virtio-scsi-pci`, but every host uses `virtio-scsi-single`.
- `cdrom` cannot be read back on import, so it always shows as an add. Declare it with `interface = "ide2"`, the real one. The provider default `ide3` would leave a second device.

Confirmed with `qm config` before and after: the one-time apply changed nothing real.

## 6. A stopped host boots on the first apply

**Symptom:** a host that was deliberately off starts after import.

**Fix:** `linux-k3s` (CT 100) and `linux-mongodb` (CT 108) were off at import. Set `started = false` to match the real power state.

## 7. Proxmox lock timeout

**Symptom:** `can't lock file '/run/lock/lxc/pve-config-<id>.lock' - got timeout`, and Terraform marks the CTs tainted.

**Fix:** the CTs were created and running. Four concurrent VM clones starved the CT start tasks. Run `terraform untaint`, and apply VMs and CTs as two separate commands. Lowering `-parallelism` would only slow the VMs.

## 8. `file_format` diff never clears

**Symptom:** `disk.file_format: "raw" -> "qcow2"` shows on every plan, even after applying.

**Fix:** a Proxmox clone does not keep that attribute. Add `disk[0].file_format` to the module's `lifecycle.ignore_changes`.
