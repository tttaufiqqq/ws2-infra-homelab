# Cutover checklist

The ASW repo is untouched. Nothing below has been done. Do the steps in order.

## 1. Publish
- [ ] Create empty GitHub repo `ws2-infra`, then
      `git remote add origin <url> && git push -u origin main`.
- [ ] Remove `infrastructure/terraform-azure/terraform.tfstate*` from the ASW working tree
      only if you no longer need it (it is gitignored, so it was never moved).

## 2. Runner and secrets
- [ ] Register `linux-gh-runner` against `ws2-infra` (it is registered to the ASW repo today).
- [ ] `terraform-drift.yml` reads Vault `secret/asw-terraform-cd` via `TF_VAULT_TOKEN` in the
      runner's `.env`. Confirm the runner service can read it for the new repo.
- [ ] Run the drift workflow once with `workflow_dispatch` and confirm exit code 0.

## 3. ArgoCD (k8s/)
- [ ] Re-point the ArgoCD Application `repoURL` to `ws2-infra`, path `k8s`.
- [ ] Move the instant-sync webhook from the ASW repo to `ws2-infra`.
- [ ] Confirm the app syncs with no diff before touching ASW.

## 4. Fix the image-tag loop (the real coupling)
`deploy.yml` in ASW builds images and runs `sed` + `git commit` on `k8s/deployment.yaml` and
`k8s/nginx-deployment.yaml`. After the move it must commit to `ws2-infra` instead:
- [ ] Create a fine-grained PAT (or deploy key) with write access to `ws2-infra` only; store it
      in Vault, not in the repo.
- [ ] Change the job to check out `ws2-infra`, bump the tags there, and push.
- [ ] Its Ansible steps use `working-directory: infrastructure/ansible`; check out `ws2-infra`
      for those too, or move those jobs.

## 5. Remove from ASW (last, one commit)
- [ ] Delete `infrastructure/`, `k8s/`, `terraform-drift.yml`, and the five moved docs.
- [ ] Fix about 20 doc/comment references to `infrastructure/...` and `k8s/...`, plus
      `docs/README.md` links and `CLAUDE.md`.
- [ ] Update `tests.yml` comments that point at `infrastructure/ansible/playbooks/app-server.yml`.
- [ ] Push, confirm Tests still green.

## Rollback
Until step 5 the ASW repo is unchanged, so repointing ArgoCD and the runner back is enough.
