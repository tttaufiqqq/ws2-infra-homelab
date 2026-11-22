# Cutover checklist

The ASW repo is untouched. Nothing below has been done. Do the steps in order.

## 1. Publish
- [ ] Create empty GitHub repo `ws2-infra-homelab`, then
      `git remote add origin <url> && git push -u origin main`.
- [ ] Remove `infrastructure/terraform-azure/terraform.tfstate*` from the ASW working tree
      only if you no longer need it (it is gitignored, so it was never moved).

## 2. Runner and secrets
- [ ] Register `linux-gh-runner` against `ws2-infra-homelab` (it is registered to the ASW repo today).
- [ ] `terraform-drift.yml` reads Vault `secret/asw-terraform-cd` via `TF_VAULT_TOKEN` in the
      runner's `.env`. Confirm the runner service can read it for the new repo.
- [ ] Run the drift workflow once with `workflow_dispatch` and confirm exit code 0.

## 3. ArgoCD (k8s/)
- [ ] Re-point the ArgoCD Application `repoURL` to `ws2-infra-homelab`, path `k8s`.
- [ ] Move the instant-sync webhook from the ASW repo to `ws2-infra-homelab`.
- [ ] Confirm the app syncs with no diff before touching ASW.

## 4. Image-tag loop (workflows written, not yet live)
ASW `deploy.yml` now builds and pushes images, then sends `repository_dispatch`
(`asw-image-built`) to this repo. `bump-image-tag.yml` here commits the new tag to `k8s/`.
`deploy-db.yml` here replaces the old `deploy-db` job. Before the first run:
- [ ] Create a fine-grained PAT on this repo only (Contents: read/write; covers dispatch).
- [ ] `vault kv patch secret/asw-cd infra_repo_pat=<pat>` (new field, same path as the others).
- [ ] Register the self-hosted runner on this repo (needed by `deploy-db.yml`, drift check).
- [ ] **Push this repo first**, so the receiver exists, **then** push ASW.
- [ ] Check Actions: ASW build job, then `Bump image tag` here, then ArgoCD sync.

## 5. Remove from ASW (done locally in ASW, not yet pushed)
- [ ] Delete `infrastructure/`, `k8s/`, `terraform-drift.yml`, and the five moved docs.
- [ ] Fix about 20 doc/comment references to `infrastructure/...` and `k8s/...`, plus
      `docs/README.md` links and `CLAUDE.md`.
- [ ] Update `tests.yml` comments that point at `infrastructure/ansible/playbooks/app-server.yml`.
- [ ] Push, confirm Tests still green.

## Rollback
Until step 5 the ASW repo is unchanged, so repointing ArgoCD and the runner back is enough.
