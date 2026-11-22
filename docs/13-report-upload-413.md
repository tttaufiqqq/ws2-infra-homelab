# Report upload 413

This article explains why report submissions hung on "Submitting Report...", how the cause was found, and which three upload limits were raised. The incident was on 24 July 2026.

## 1. Symptom

Submitting a stray report with photos hung on "Submitting Report..." indefinitely, and the site felt slower overall. It was reported right after the homelab network segmentation work (VLANs and a new OPNsense router), so the network was the first suspect.

## 2. What was ruled out

Each suspect was tested directly and was healthy.

| Suspect | Result |
|---|---|
| DNS and database reachability | Database connections under 70 ms |
| Vault secret injection | The Vault Agent rotated every credential (`DB1_PASSWORD` to `DB5_PASSWORD`, `CLOUDINARY_*`, `APP_KEY`) every few minutes with no errors |
| Outbound bandwidth | About 180 Mbps |
| Cloudinary | Reachable |

The network work was not the cause. Ruling it out properly led to the real bug.

## 3. Cause

With `access.log` and `error.log` tailed live, reproducing the submit showed:

```
[error] client intended to send too large body: 3929516 bytes ...
request: "POST /reports HTTP/1.1"
"POST /reports HTTP/1.1" 413 594
```

nginx rejected the roughly 3.9 MB image almost instantly with `413 Payload Too Large`. The server was not slow. The frontend never handled a 413, so the submit button stayed in its loading state.

Three limits were still at their defaults, and none matched the report form's own rule of up to 5 images at 5 MB each:

| Setting | Where | Before | After |
|---|---|---|---|
| `client_max_body_size` | `infrastructure/ansible/templates/nginx-locations.conf.j2` | Unset (nginx default 1 MB) | 30M |
| `upload_max_filesize` | `/etc/php/8.3/fpm/php.ini`, set by a new task in `app-server.yml` | 2M (package default) | 8M |
| `post_max_size` | Same file | 8M (package default) | 30M |

The form had presumably been broken since it was built. It surfaced when someone first uploaded a photo larger than 1 MB.

## 4. Fix

Raise all three limits on the live VM and in the Ansible source, so a later `ansible-playbook app-server.yml` does not undo them.

```bash
sudo nginx -t && sudo systemctl reload nginx
sudo systemctl restart php8.3-fpm
```

**Verify:** a real 4 MB `POST` to `/reports` changed from `413` to `419`. The `419` is a CSRF mismatch, which is expected without a logged-in session. It confirms nginx and PHP now accept the payload and the request reaches Laravel.

## 5. Still open

The frontend should show an error message and re-enable the submit button when a submission fails (413, 419 or validation errors), not spin forever. That made a fast server-side rejection feel like a slow hang. It is a frontend code change, so it was not part of this fix.
