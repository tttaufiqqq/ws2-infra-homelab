# Hardening

This article explains how each machine was hardened so services survive reboots and power cuts, and so each database accepts connections only from the application tier over Tailscale.

## 1. Approach

The guiding principle is defence in depth. Network binding, the OS firewall, database-level access control and process auto-start each block unauthorized access on their own, so one misconfigured layer does not expose the database.

| Goal | How it is met |
|---|---|
| Availability | Services are enabled in systemd, and Proxmox starts every VM and CT at boot |
| Network isolation | The database listens only on its Tailscale address, not on the LAN |
| Access control | UFW allows the database port only from the app server's Tailscale address |
| Startup ordering | Database hosts start before the app server, which waits 30 seconds |

### 1.1 Machine reference

| Hostname | Type | DBMS | Port |
|---|---|---|---|
| `app-server` | VM | Laravel (nginx, PHP-FPM) | n/a |
| `linux-mariadb` | VM, later CT | MariaDB 10.11 | 3306 |
| `linux-mariadb-2` | CT | MariaDB 10.11 | 3306 |
| `linux-postgres` | VM, later CT | PostgreSQL 16 | 5432 |
| `linux-mysql` | VM, later CT | MySQL 8.0 | 3306 |
| `linux-mysql-2` | CT | MySQL 8.0 | 3306 |

`shelter` and `animals` previously ran on a Windows host and moved to `linux-mysql` and `linux-mysql-2` on 2026-07-20. `booking` was split from `linux-mariadb` onto `linux-mariadb-2` the same day.

## 2. Application server

### 2.1 Auto-start the web server

A VM that reboots after a power cut does not start services unless systemd enables them.

```bash
sudo systemctl enable nginx
sudo systemctl enable php8.3-fpm
systemctl is-enabled nginx php8.3-fpm   # both print: enabled
```

### 2.2 Firewall

UFW enforces an allowlist: SSH, HTTP and HTTPS only. SSH is bound to the `tailscale0` interface, so a spoofed source address on `eth0` cannot match the rule. There is no global `allow 22/tcp`.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on tailscale0 from <admin tailscale ip> to any port 22 proto tcp comment 'SSH from admin'
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
sudo ufw status verbose
```

### 2.3 Storage permissions after a reboot

Laravel writes to `storage/` and `bootstrap/cache/`. If ownership reverts, the app returns 500 errors. Restore it with:

```bash
sudo chown -R workshop:www-data <app>/storage <app>/bootstrap/cache
chmod -R 775 <app>/storage <app>/bootstrap/cache
```

## 3. Database hosts

Every database host follows the same four steps. Replace the port and address per host.

### 3.1 Bind to the Tailscale address

A default `bind-address = 0.0.0.0` listens on every interface. Binding to the Tailscale address hides the port from the LAN.

**MariaDB**

```bash
grep -r "bind-address" /etc/mysql/        # usually /etc/mysql/mariadb.conf.d/50-server.cnf
```

```ini
[mysqld]
bind-address = <tailscale ip of this host>
```

**PostgreSQL**

```bash
sudo find /etc/postgresql -name postgresql.conf   # usually /etc/postgresql/16/main/
```

```ini
listen_addresses = '<tailscale ip of this host>'
```

### 3.2 Restrict `pg_hba.conf` (PostgreSQL only)

`pg_hba.conf` is evaluated before any SQL runs, so it is a third independent check. Use `scram-sha-256`, not `md5`: `md5` exposes a hash that can be cracked offline if the exchange is captured.

```
# TYPE  DATABASE  USER             ADDRESS                METHOD
local   all       all                                     peer
host    all       workshop_2_prod  <app-server ip>/32     scram-sha-256
```

Remove any broad rule such as `host all all 0.0.0.0/0 md5`.

### 3.3 Enable auto-start

```bash
sudo systemctl enable mariadb      # or mysql, postgresql
sudo systemctl is-enabled mariadb  # prints: enabled
```

### 3.4 Firewall

The firewall is a second layer: if a package update reverts `bind-address`, UFW still blocks unauthorized peers at the kernel.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on tailscale0 from <admin tailscale ip> to any port 22 proto tcp comment 'SSH from admin'
sudo ufw allow in on tailscale0 from <app-server ip> to any port 3306 proto tcp comment 'DB from app-server'
sudo ufw --force enable
```

Use port 5432 for PostgreSQL.

> [!WARNING]
> Never use a blanket `ufw allow in on tailscale0`. It lets every Tailscale peer reach the database port and removes the source-address restriction. The interface and source address together provide the restriction.

### 3.5 Restart and verify

```bash
sudo systemctl restart mariadb
ss -tlnp | grep 3306      # expect <tailscale ip>:3306, not 0.0.0.0
```

## 4. Ansible-provisioned hosts

`linux-mysql`, `linux-mysql-2` and `linux-mariadb-2` were provisioned by Ansible from the start (`linux-mysql.yml`, `linux-mysql-2.yml`, `linux-mariadb-2.yml`). The playbooks enable the service and apply the UFW rules above.

One difference is deliberate: `bind-address` stays at `0.0.0.0` on these three hosts. UFW is therefore the only network-layer restriction, not a second layer. Re-running the playbook with a Tailscale-bound address would close that gap.

```bash
ss -tlnp | grep 3306     # expect 0.0.0.0:3306 on these three hosts
sudo ufw status verbose
```

## 5. Proxmox startup order

If the node reboots and every machine starts at once, Laravel may connect before a database finishes starting. Laravel fails fast on startup and does not retry, so the app serves 500 errors until restarted.

Set it under each machine's **Options**, **Start/Shutdown order**:

| Machine | Start at boot | Order | Delay |
|---|---|---|---|
| All five database hosts | Yes | 1 | 0s |
| `app-server` | Yes | 2 | 30s |

The 30 second delay is a buffer for slow disk recovery after a hard shutdown. The databases usually start in under 10 seconds.

## 6. Verification checklist

Run these from the app server after hardening. Enter the database password when prompted.

```bash
mysql -h <reporting ip> -u workshop_2_prod -p -e "SELECT 'MariaDB (reporting) ok';"
mysql -h <booking ip>   -u workshop_2_prod -p -e "SELECT 'MariaDB (booking) ok';"
mysql -h <shelter ip>   -u workshop_2_prod -p -e "SELECT 'MySQL (shelter) ok';"
mysql -h <animals ip>   -u workshop_2_prod -p -e "SELECT 'MySQL (animals) ok';"
psql  -h <users ip> -U workshop_2_prod -d workshop_2_prod -c "SELECT 'PostgreSQL ok';"

curl -s -o /dev/null -w "%{http_code}" http://localhost/    # expect 200
sudo ufw status verbose
```

To confirm the firewall blocks others, connect to a database port from a machine that is not the app server. The attempt should time out or be refused.

## 7. What each measure protects against

| Measure | Threat mitigated |
|---|---|
| `bind-address` set to the Tailscale address | Database port is invisible on LAN and public interfaces |
| UFW allowlist on all five hosts | Blocks every Tailscale peer except the app server |
| `pg_hba.conf` limited to the app server | Rejects other addresses even if they pass the firewall |
| `scram-sha-256` | Prevents offline cracking of captured authentication |
| `systemctl enable` | Services restart after a reboot or power cut |
| Proxmox start at boot | Machines return when the node recovers |
| Startup order and delay | Stops the app connecting before the databases are ready |
