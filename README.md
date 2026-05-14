# PostgreSQL HA Cluster

Production-grade PostgreSQL High Availability cluster deployed with Ansible.
Built as a hands-on lab project demonstrating real-world infrastructure automation skills.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     Client Applications                  │
└────────────────────────┬────────────────────────────────┘
                         │
              ┌──────────▼──────────┐
              │   HAProxy :5000/5001 │  Write / Read split
              │   Prometheus :9090   │
              │   Grafana    :3000   │
              │   MinIO      :9000   │
              └──────────┬──────────┘
                192.168.56.10
                         │
          ┌──────────────┴──────────────┐
          │                             │
┌─────────▼─────────┐       ┌──────────▼────────┐
│   pg1 (Leader)    │  sync │  pg2 (Sync Standby)│
│   PostgreSQL 16   │◄─────►│   PostgreSQL 16    │
│   Patroni         │       │   Patroni          │
│   pgBouncer :6432 │       │   pgBouncer :6432  │
│   etcd            │       │   etcd             │
│   WAL-G           │       │   WAL-G            │
└───────────────────┘       └────────────────────┘
   192.168.56.11               192.168.56.12
```

## Stack

| Component | Version | Role |
|---|---|---|
| PostgreSQL | 16 | Primary database |
| Patroni | latest | Auto-failover HA manager |
| etcd | 3-node | Distributed config store / DCS |
| pgBouncer | latest | Connection pooling |
| HAProxy | latest | Load balancer, R/W split |
| WAL-G | v3.0.0 | Backup & WAL archiving |
| MinIO | latest | S3-compatible backup storage |
| Prometheus | latest | Metrics collection |
| Grafana | latest | Metrics visualization |

## Key Features

- **Zero data loss** — `synchronous_mode_strict: true` in Patroni DCS
- **Automatic failover** — Patroni + etcd quorum-based leader election
- **Connection pooling** — pgBouncer in transaction mode, scram-sha-256 auth
- **R/W split** — HAProxy routes writes to :5000, reads to :5001
- **WAL archiving** — continuous WAL shipping to MinIO via WAL-G
- **Daily backups** — full base backup at 02:00, retain last 7, brotli compression
- **Monitoring** — postgres_exporter → Prometheus → Grafana
- **Secret management** — Ansible Vault (see Security section)

## Requirements

- 3x Ubuntu 22.04 VMs (tested with VirtualBox + Host-Only network)
- Ansible 2.14+
- Python 3.10+
- SSH key access to all nodes

```
VM layout:
  192.168.56.10  haproxy   (HAProxy, Prometheus, Grafana, MinIO)
  192.168.56.11  pg1       (PostgreSQL, Patroni, pgBouncer, etcd, WAL-G)
  192.168.56.12  pg2       (PostgreSQL, Patroni, pgBouncer, etcd, WAL-G)
```

## Quick Start

### 1. Clone and prepare secrets

```bash
git clone https://github.com/samarets-vlad/postgres-cluster
cd postgres-cluster

# Copy vault example and set your passwords
cp group_vars/vault_example.yml group_vars/vault.yml
nano group_vars/vault.yml

# Encrypt with ansible-vault
ansible-vault encrypt group_vars/vault.yml

# Save vault password for convenience
echo "your_vault_password" > ~/.vault_pass
chmod 600 ~/.vault_pass
```

### 2. Deploy full stack

```bash
ansible-playbook site.yml -K --vault-password-file ~/.vault_pass
```

### 3. Verify cluster

```bash
# Cluster status
sudo -u postgres patronictl -c /etc/patroni/patroni.yml list
# Expected: pg1=Leader, pg2=Sync Standby, Lag=0

# Write endpoint (primary)
psql -h 192.168.56.10 -p 5000 -U admin -d postgres -c "SELECT inet_server_addr();"

# Read endpoint (replica)
psql -h 192.168.56.10 -p 5001 -U admin -d postgres -c "SELECT inet_server_addr();"

# Check backup
sudo -u postgres wal-g --config /etc/wal-g/walg.json backup-list
```

## Access

| Service | URL | Credentials |
|---|---|---|
| Grafana | http://192.168.56.10:3000 | admin / see vault.yml |
| HAProxy Stats | http://192.168.56.10:7000 | admin / see vault.yml |
| MinIO Console | http://192.168.56.10:9001 | see vault.yml |
| Prometheus | http://192.168.56.10:9090 | no auth |

## Failover Test

```bash
# Simulate primary failure
sudo systemctl stop patroni  # on pg1

# Watch failover happen (~10-15 seconds)
sudo -u postgres patronictl -c /etc/patroni/patroni.yml list
# pg2 becomes Leader automatically

# Restore pg1
sudo systemctl start patroni  # on pg1
# pg1 rejoins as Sync Standby
```

## Security

> ⚠️ **Lab note:** `group_vars/vault_example.yml` contains plain-text example
> credentials for demonstration purposes only.
>
> **In production:**
> - `group_vars/vault.yml` must be encrypted with `ansible-vault` and **never committed to git**
> - `.gitignore` already excludes `vault.yml` and `.vault_pass`
> - Use strong unique passwords for all services
> - Rotate `grafana_secret_key` and all passwords before production use
> - Consider using HashiCorp Vault or AWS Secrets Manager for secret management at scale

```bash
# Encrypt vault
ansible-vault encrypt group_vars/vault.yml

# View encrypted vault
ansible-vault view group_vars/vault.yml

# Edit encrypted vault
ansible-vault edit group_vars/vault.yml
```

## Project Structure

```
postgres-cluster/
├── ansible.cfg                  # Ansible defaults
├── inventory.ini                # Host definitions
├── site.yml                     # Main playbook (9 steps)
├── group_vars/
│   ├── all.yml                  # Non-secret variables
│   ├── vault.yml                # Encrypted secrets (git-ignored)
│   └── vault_example.yml        # Example vault structure
└── roles/
    ├── common/                  # Base packages, sysctl tuning
    ├── etcd/                    # etcd cluster
    ├── patroni/                 # PostgreSQL + Patroni + auto DCS config
    ├── pgbouncer/               # Connection pooler
    ├── haproxy/                 # Load balancer + R/W split
    ├── postgres_exporter/       # Prometheus exporter
    ├── prometheus/              # Metrics collection
    ├── grafana/                 # Dashboards + auto password reset
    ├── minio/                   # S3-compatible storage
    └── walg/                    # Backup agent + cron schedule
```

## Author

Vlad Samarets — DevOps Engineer
