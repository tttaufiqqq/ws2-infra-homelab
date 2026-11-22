# Ansible

This article explains how Terraform and Ansible split the work of building the fleet, and how to run the Ansible playbooks.

## 1. Why two tools

Setting up six machines by hand means clicking through Proxmox, SSHing into each one and hoping every step is remembered on the second build. Infrastructure as code removes that.

| Tool | Owns | Example |
|---|---|---|
| Terraform | What exists | VM ID, CPU, RAM, disk, network, cloud-init |
| Ansible | What runs inside | Packages, databases, config files, firewall |
| Tailscale | The network | Stable hostnames such as `linux-mysql`, with no port forwarding |

- Change a database config: run Ansible. Terraform is not involved.
- Add a Terraform-managed VM: Ansible picks it up from the inventory.
- Running the same playbook twice gives the same result (idempotent).

### 1.1 Which machines Terraform manages

The fleet has six machines: the app server, two MySQL hosts (`shelter`, `animals`), two MariaDB hosts (`reporting`, `booking`) and one PostgreSQL host (`users`).

- Terraform created four of them as VMs: `app-server`, `linux-mysql`, `linux-mariadb` and `linux-postgres`.
- `linux-mysql-2` and `linux-mariadb-2` are containers (CTs) created directly with Proxmox `pct` on 2026-07-20, when each shared host was split in two.
- The CTs are configured by Ansible like every other host. They need a playbook and an inventory entry, and Terraform is never involved.

The Terraform VM count is correct for what Terraform manages. It is not the total machine count. Later, the real fleet was imported into Terraform ([Terraform](07-terraform.md)).

## 2. Setup

Ansible runs from a WSL Ubuntu instance. It needs no agent and connects over SSH through Tailscale.

```bash
sudo apt update && sudo apt install -y ansible python3-pip

cd infrastructure/ansible
ansible-galaxy collection install -r requirements.yml
pip3 install hvac        # Vault client used by the hashi_vault collection

ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519
```

The public key in `~/.ssh/id_ed25519.pub` must match `ssh_public_key` in `terraform.tfvars`. If they differ, Ansible cannot connect.

### 2.1 SSH usernames differ per host

Cloud-init created a `workshop` user, but each machine later ended up with its own convention.

| Host | SSH user |
|---|---|
| `linux-mysql-2`, `linux-mariadb-2` | Matches the hostname |
| `app-server` | `taufiq` |
| Other hosts | Varies; see below |

`ssh workshop@<host>` fails on most hosts. The real username is the `User` line in `~/.ssh/config`. Check that file before guessing.

## 3. Run the playbooks

After `terraform apply`, wait about 60 seconds for cloud-init to finish, then confirm connectivity.

```bash
tailscale status              # all six machines should appear
ssh linux-mysql "echo ok"     # uses the user from ~/.ssh/config
```

Every playbook that reads `asw_secrets` needs a scoped Vault AppRole in the environment. Never use the Vault root token.

```bash
export VAULT_ROLE_ID="<asw-deploy role id>"
export VAULT_SECRET_ID="<asw-deploy secret id>"

cd infrastructure/ansible
ansible-playbook playbooks/site.yml          # all six machines, in dependency order
ansible-playbook playbooks/linux-mysql.yml   # or a single host
```

`app-server.yml` deploys as `taufiq`, because it targets the hand-configured box. A fresh Terraform VM only has `workshop`, so the playbook does not complete there until a `taufiq` user exists. See [Production hardening](09-production-hardening.md).

### 3.1 Update Tailscale addresses

If MagicDNS is on, hostnames resolve and `inventory.yml` works as is. If not, read the addresses with `tailscale status` and set `ansible_host` in `inventory.yml`.

```yaml
linux-mysql:
  ansible_host: 100.x.x.x   # from tailscale status
```

## 4. What each playbook does

| Playbook | Host | Result |
|---|---|---|
| `linux-mysql.yml`, `linux-mysql-2.yml` | MySQL hosts | MySQL 8.0, `workshop_2_prod` database and user, remote access, triggers, UFW |
| `linux-mariadb.yml`, `linux-mariadb-2.yml` | MariaDB hosts | MariaDB, `workshop_2_prod` database and user, remote access, UFW |
| `linux-postgres.yml` | PostgreSQL host | PostgreSQL, database and user, `pg_hba`, schema grants, UFW |
| `app-server.yml` | App server | PHP 8.3 and extensions, nginx, Node 20, Composer, UFW, then the deploy below |
| `site.yml` | All six | Imports the five database playbooks, then `app-server.yml` |

A `workshop_2_dev` database and user also exist on each server for local development. Ansible does not manage them.

### 4.1 What `app-server.yml` deploys

1. Clones the repository to `/home/taufiq/Animal-Shelter-Workshop`.
2. Renders `.env` from `templates/env-app.j2`, taking every credential from Vault through `asw_secrets`. It re-renders on every run.
3. Runs `composer install` and `npm ci && npm run build`.
4. Installs the nginx site from `templates/nginx-app.conf.j2`. It serves plain HTTP on port 80, because the Cloudflare Tunnel terminates TLS unless `-e use_certbot=true` is passed.
5. Runs `migrate --force` every time, and `db:seed --force` only on the first deploy, gated on `storage/.provisioned`.

The playbook never runs `db:fresh-all` ([Production hardening](09-production-hardening.md)). Re-running it is safe, because every task is idempotent.

## 5. Verify

Check usernames against `~/.ssh/config` first. Enter the database password when prompted.

```bash
ssh linux-mysql   "mysql -u workshop_2_prod -p -e 'SHOW DATABASES;'"
ssh linux-mariadb "mysql -u workshop_2_prod -p -e 'SHOW DATABASES;'"
ssh linux-postgres "psql -U workshop_2_prod -d workshop_2_prod -c '\l'"
ssh linux-app-server "php -v && nginx -v && composer --version"
```

## 6. Full flow

1. `terraform init && terraform apply` creates the Terraform-managed VMs on Proxmox. The two CTs already exist.
2. The machines boot, cloud-init joins them to Tailscale, and all six appear in `tailscale status`.
3. `ansible-playbook playbooks/site.yml` installs software, configures the databases and deploys the app.
