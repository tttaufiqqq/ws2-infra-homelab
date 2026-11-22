# Production hardening

This article records what changed to make the app deployable for unattended, single-user homelab operation, and which items were left as deliberate follow-ups.

## 1. Scope

The original deploy could wipe production data, password reset was silently broken, `users` timestamps were off by about 8 hours, and there was no TLS, no backups and no scheduler.

"Production" here means a portfolio deployment. The goal is to show the engineering work, not to serve real users, and there is no real data to protect. That caps some choices: the payment gateway stays on ToyyibPay's sandbox permanently, and "production-grade" means "handled the way a real deploy would be".

## 2. What changed

### 2.1 The deploy no longer destroys data

**Problem:** `app-server.yml` ran `php artisan db:fresh-all --seed` on every deploy. That command drops every table on all five databases, so any redeploy after go-live would have wiped real data.

**Fix:** the deploy now runs these steps in order:

1. `optimize:clear`
2. `migrate --force` (always, idempotent)
3. `db:seed --force`, gated on a `storage/.provisioned` flag file created once
4. `config:cache`, `route:cache`, `view:cache`, `event:cache`
5. `storage:link`

`db:fresh-all` is no longer called anywhere in the playbook. The `git` task uses `update: true`, so a redeploy pulls new commits.

A `--check --diff` run against the live box found a gap in that gate. `storage/.provisioned` was missing, because the box had been configured by hand and never deployed by the playbook. The first real run would have seeded demo records on top of live data (102 real animal rows and the user accounts). An empty `storage/.provisioned` was created on the box first.

Two migrations that had only run against the `_test` databases were applied to production as a one-time catch-up: `create_password_reset_tokens_table` and `add_remember_token_to_users_table`. Password reset was broken until then.

### 2.2 Timezone alignment

**Problem:** `config/app.php` uses `Asia/Kuala_Lumpur`, but no connection set a session timezone. PHP-side writes disagreed with the database by about 8 hours (`audit_logs`, `updated_at`, `User::isLocked()`).

**Fix:** `config/database.php` now sets `timezone` on all five connections.

| Connection | Value | Reason |
|---|---|---|
| `users` (PostgreSQL) | `Asia/Kuala_Lumpur` | Has its own tzdata |
| The four MySQL and MariaDB connections | `+08:00` | The named zone is not loaded, and Malaysia has no DST, so a fixed offset is exact |

This also fixed a time-of-day flake in `tests/Procedures/BookingTriggersTest.php`.

### 2.3 Sessions, cache and queue off the `users` database

**Problem:** `CACHE_STORE=database` and `SESSION_DRIVER=database` both resolved to the PostgreSQL `users` connection, so one Postgres outage would have broken every page.

**Fix:** both are now `file`. `QUEUE_CONNECTION` is `sync`, because the app has no queued jobs, mail or notifications.

### 2.4 Scheduler on a systemd timer

`routes/console.php` schedules `db:refresh-status` (every minute) and `taufiq:refresh-stats` (every 5 minutes), but nothing ran `schedule:run`, so both were dead. A systemd timer (`laravel-scheduler.timer.j2`) now fires every minute and runs a oneshot service. No queue worker was added.

### 2.5 TLS

nginx used to listen on port 80 only. The playbook now requires `app_domain` and `certbot_email`, and fails early if either is missing. It gets a certificate by certbot's webroot challenge and serves port 443 with HSTS, `X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy` and `X-XSS-Protection`. Port 80 redirects once a certificate exists.

`APP_URL` is `https://{{ app_domain }}`, `SESSION_SECURE_COOKIE` and `SESSION_ENCRYPT` are `true`, and `AppServiceProvider::boot()` calls `URL::forceScheme('https')` in production, because nginx terminates TLS.

A Content-Security-Policy was deliberately not added. The app loads several CDNs (Tailwind, Font Awesome, Leaflet, Nominatim), and a CSP needs its own pass so it does not break the map.

### 2.6 Trusted proxies

**Problem:** the request path is Cloudflare, then the `cloudflared` tunnel, then nginx on port 80, then php-fpm. Every request arrives from `127.0.0.1`, and Laravel trusts no forwarded headers by default. `$request->ip()` returned the proxy and `isSecure()` returned `false` on real HTTPS visits.

**Fix:** `$middleware->trustProxies(at: ['127.0.0.1'])` in `bootstrap/app.php`. It trusts `127.0.0.1` and not `'*'`, because that is the only known hop.

### 2.7 Mail

**Problem:** production had no `MAIL_*` settings, so it used the `log` mailer and password reset emails went to `storage/logs`.

**Fix:** `env-app.j2` has a full SMTP block (`MAIL_MAILER`, `MAIL_HOST`, `MAIL_PORT`, `MAIL_USERNAME`, `MAIL_PASSWORD`, `MAIL_SCHEME`, `MAIL_FROM_ADDRESS`, `MAIL_FROM_NAME`). The provider is Resend, with the domain verified for SPF, DKIM and DMARC. The key lives in Vault.

### 2.8 Backups

The first design was a per-host dump timer. It had no off-VM copy, no restore drill and no consistency check across databases. It was replaced by one coordinated backup run from the app server (`php artisan db:backup`). See [Backups](10-backups.md).

### 2.9 Stored routine privileges after the prod/dev split

**Symptom:** "Failed to delete report" and "Failed to assign caretaker" on `reporting`, with `execute command denied to user 'workshop_2'@'%'`, although the app connects as `workshop_2_prod`.

**Cause:** `workshop_2_prod` was filled by dump-restoring the old database, which kept every routine's `DEFINER=workshop_2@%`. Under the default `SQL SECURITY DEFINER`, privilege checks run as the definer, and `workshop_2` has no rights on `workshop_2_prod`. Nobody could call the routines, not even `root`. All 121 procedures and triggers on the four MySQL and MariaDB hosts failed. `users` (PostgreSQL) was unaffected.

**Fix:** done live in SQL, with no code or Ansible change. Procedures and functions were switched to `SQL SECURITY INVOKER`. Triggers have no such option, so they were recreated with `DEFINER=workshop_2_prod@%`. The full write-up is in the homelab repository, `docs/16-stored-routine-definer-privilege-split/`.

### 2.10 Animal delete checks real references

**Problem:** `ManagesAnimals::destroy()` did not check the `booking` connection (`adoption`, `animal_booking`). Those live on another server, so no foreign key could stop the delete, and adopted or booked animals orphaned rows.

**Fix:** the controller now looks for referencing rows first and blocks the delete. It fails closed if `booking` is unreachable. `OrphanRiskTest.php` was inverted to assert the block, and verified red then green.

### 2.11 Logging

`LOG_STACK` was unset, so logging fell back to one unbounded `laravel.log`. It is now `daily`, with the 14-day default retention. nginx logs already rotate through `nginx-common`.

Two endpoints are worth monitoring: Laravel's `/up` and the app's `/api/database-status`, which reports per-connection reachability.

### 2.12 Secrets moved to Vault

**Problem:** credentials lived in three places. `env-app.j2` hardcoded all five `DBn_PASSWORD` values, the three database playbooks hardcoded the same password, and the Resend key existed only in the live `.env`.

**Fix:** everything now comes from one Vault path, `secret/animal-shelter-workshop` on `linux-vault`, read at deploy time with the `vault_kv2_get` lookup (`group_vars/all.yml`). A scoped, read-only AppRole (`asw-deploy`) authenticates, not the root token. `APP_KEY` moved too, so `.env` is re-rendered from Vault on every run.

The preparation found four bugs that predate Vault:

| Bug | Effect |
|---|---|
| `TOYYIBPAY_SECRET_KEY` and `_CATEGORY_CODE` did not match `config/toyyibpay.php` (`TOYYIBPAY_KEY`, `_CATEGORY`) | Payments misconfigured. Fixed |
| `DB2_HOST` and `DB3_HOST` pointed at `linux-mysql`, which had no `workshop_2` database | Real shelter and animal data was on another host. Fixed |
| `FILESYSTEM_DISK` was never set | Uploads would have moved off Cloudinary to local disk. Set to `cloudinary` |
| `BOOKING_PREFER_SQLSRV`, `VITE_APP_NAME`, `BROADCAST_CONNECTION` in the live `.env` only | Confirmed unused by searching `app/`, `config/` and `resources/`. Left out on purpose |

Consequences:

- Deploys depend on Vault being unsealed.
- `key:generate` no longer runs, because regenerating a key that comes from Vault would make the two diverge.

### 2.13 Secrets never touch disk (Vault Agent)

**Problem:** every deploy still wrote all secrets to `.env`, and `config:cache` baked them into `bootstrap/cache/config.php`. That left two plaintext copies readable by anyone with file access.

**Fix:** `playbooks/tasks/vault-agent.yml` wraps php-fpm (a systemd `ExecStart` override) and the migrate and seed commands in `vault agent`. It injects secrets into the process environment from `env_template` stanzas, never to a file. `env-app.j2` blanks every Vault field when `vault_agent_enabled` is on, and `config:cache` is skipped. Setting it to `false` reverts to the plain flow, including removing the override. The revert was tested.

Six bugs surfaced getting it running:

- An invalid CLI flag on this Vault version
- Two file permission mismatches between `root` and `taufiq`
- `needrestart` hanging unattended `apt` installs
- A recursive `chmod` that made tracked files executable
- Vault Agent deduplicating identical `env_template` bodies, so 4 of 5 identical database passwords never reached the app

The tradeoff is that php-fpm now needs Vault reachable on every restart. The full design is in the homelab repository, `docs/11-vault-approle-app-integration/`.

### 2.14 Removed email-verification scaffolding

Routes, controllers and a view for Laravel's verification flow existed but were never wired: `User` did not implement `MustVerifyEmail` and there was no `email_verified_at` column. A single-user deployment gets no value from it, so three controllers, the routes, a dead form and the cast were removed.

## 3. Follow-ups

| Item | Status |
|---|---|
| Content-Security-Policy | Open. Needs a pass over the CDN origins |
| Off-VM backups and a restore drill | Closed by [Backups](10-backups.md) |
| ToyyibPay on the sandbox (`dev.toyyibpay.com`) | By design, not a pending decision |
| `ansible-playbook --syntax-check` | Open. The pass was written on a machine without Ansible |
| Playbook path (`/home/taufiq/...`, user `taufiq`) | Closed. The playbook now matches the real box |
| Fresh VM lacks a `taufiq` user | Closed. Fixed in `cloud-init.yml.tftpl`, and the real fleet is imported into Terraform ([Terraform](07-terraform.md)) |
| SMTP credentials | Closed, and stored in Vault |
| Confirm delivery to a real inbox | Open. The test hit a seeded demo address, which bounced. That proves the relay works, but a real address still needs testing |
| Vault runs with `tls_disable = true` | Open. Tailscale encrypts the transport, which is acceptable for a lab but a real gap in production |

## 4. Verification

- The full Pest suite was re-run after every code change, with **364 tests passing** against real test databases.
- The two migrations were confirmed live with `hasTable('password_reset_tokens')` and `hasColumn('users', 'remember_token')`.
- The timezone fix was confirmed by comparing PHP's `now()` with each connection's session clock. The skew dropped from about 8 hours to under a second.
- The animal-delete guard was checked red then green with `git stash` on the controller change.
- The `mysqldump` command ran against the real local MySQL and produced a valid dump. `pg_dump` was reviewed by hand, because the dev machine had no PostgreSQL client.
- **Not verified** from the dev environment: a live systemd timer firing, a real TLS handshake, real SMTP delivery and a restore drill. They need the real hardware, domains and secrets.
