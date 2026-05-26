# PostgreSQL HA Cluster

Production-grade PostgreSQL High Availability cluster deployed with Ansible.
Built as a hands-on lab project demonstrating real-world infrastructure automation skills.
The entire environment is provisioned locally with **Vagrant + VirtualBox** — no cloud account needed.

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
| pgBouncer | latest | Connection pooling (transaction mode) |
| HAProxy | latest | Load balancer, R/W split |
| WAL-G | v3.0.0 | Backup & WAL archiving |
| MinIO | latest | S3-compatible backup storage |
| Prometheus | latest | Metrics collection + alert rules |
| Grafana | latest | Metrics visualization |

## Key Features

- **Zero data loss** — `synchronous_mode_strict: true` in Patroni DCS
- **Automatic failover** — Patroni + etcd quorum-based leader election
- **Connection pooling** — pgBouncer in transaction mode, scram-sha-256 auth, PG `max_connections=50`
- **R/W split** — HAProxy routes writes to :5000, reads to :5001
- **WAL archiving** — continuous WAL shipping to MinIO via WAL-G
- **Daily backups** — full base backup at 02:00, retain last 7, brotli compression
- **PITR-ready** — `restore_command` configured for Point-in-Time Recovery via WAL-G
- **Alerting** — Prometheus alert rules for Patroni, PostgreSQL, and WAL-G backup failures
- **Secret management** — Ansible Vault for all passwords

## Requirements

### Host machine

- [VirtualBox](https://www.virtualbox.org/) 6.1+
- [Vagrant](https://www.vagrantup.com/) 2.3+
- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/) 2.14+
- ~6 GB free RAM (haproxy: 1 GB, pg1+pg2: 2 GB each)
- ~15 GB free disk

```bash
# Verify versions
virtualbox --help | head -1
vagrant --version
ansible --version
```

## Quick Start

### 1. Clone the repository

```bash
git clone https://github.com/samarets-vlad/postgres-cluster
cd postgres-cluster
```

### 2. Set up secrets

```bash
# Copy the example vault and fill in your passwords
cp group_vars/vault_example.yml group_vars/vault.yml
nano group_vars/vault.yml

# Encrypt with ansible-vault and save the vault password
ansible-vault encrypt group_vars/vault.yml
echo "your_vault_password" > .vault_pass
chmod 600 .vault_pass
```

> ⚠️ `.vault_pass` and `group_vars/vault.yml` are already in `.gitignore`.

### 3. Bring up the cluster

```bash
vagrant up
```

Vagrant will:
1. Create 3 Ubuntu 22.04 VMs on the `192.168.56.0/24` host-only network
2. Create the `vlad` user with passwordless sudo on each VM
3. Wait for all VMs to be up, then run `ansible-playbook site.yml` automatically from the host

First run takes **~15–20 minutes** (apt installs + WAL-G download + initial backup).

### 4. Verify the cluster

```bash
# SSH into pg1 and check cluster status
vagrant ssh pg1
sudo -u postgres patronictl -c /etc/patroni/patroni.yml list
# Expected: pg1=Leader, pg2=Sync Standby, Lag=0

# Write endpoint (primary only)
psql -h 192.168.56.10 -p 5000 -U admin -d postgres -c "SELECT inet_server_addr();"

# Read endpoint (replica)
psql -h 192.168.56.10 -p 5001 -U admin -d postgres -c "SELECT inet_server_addr();"

# Check backup list
vagrant ssh pg1 -c "sudo -u postgres wal-g --config /etc/wal-g/walg.json backup-list"
```

## Re-running Ansible Without Recreating VMs

If you change a role and want to re-apply without destroying the VMs:

```bash
# Re-provision all
vagrant provision

# Or run ansible-playbook directly (VMs must already be up)
ansible-playbook site.yml --vault-password-file .vault_pass

# Run a single step (e.g. only WAL-G role)
ansible-playbook site.yml --vault-password-file .vault_pass --tags walg

# Run against one host
ansible-playbook site.yml --vault-password-file .vault_pass --limit pg1
```

## VM Management

```bash
vagrant status          # show VM states
vagrant halt            # graceful shutdown of all VMs
vagrant up              # start existing VMs (no re-provision)
vagrant destroy -f      # delete all VMs (data is lost)
vagrant ssh haproxy     # SSH into a specific VM
```

## Failover Test

```bash
# Watch the cluster in real time (run in one terminal)
vagrant ssh pg1 -c "watch -n2 'sudo -u postgres patronictl -c /etc/patroni/patroni.yml list'"

# Simulate primary failure (another terminal)
vagrant ssh pg1 -c "sudo systemctl stop patroni"
# pg2 becomes Leader in ~10-15 seconds

# Restore pg1
vagrant ssh pg1 -c "sudo systemctl start patroni"
# pg1 rejoins as Sync Standby
```

## Access

| Service | URL | Credentials |
|---|---|---|
| Grafana | http://192.168.56.10:3000 | admin / see vault.yml |
| HAProxy Stats | http://192.168.56.10:7000 | admin / see vault.yml |
| MinIO Console | http://192.168.56.10:9001 | see vault.yml |
| Prometheus | http://192.168.56.10:9090 | no auth |
| Alertmanager | — | not included (alert rules fire to Prometheus only) |

## Alerting

Prometheus alert rules are deployed to `/etc/prometheus/rules/` and cover:

| Rule file | Alerts |
|---|---|
| `patroni.yml` | LeaderMissing, ClusterUnhealthy, StandbyNotInSync, ReplicationLag, EtcdUnhealthy |
| `postgresql.yml` | PostgresDown, ConnectionsNearLimit (>80%), ReplicationLagCritical, Deadlocks, TxIDWraparound, TableBloat |
| `walg.yml` | BackupMissing (>25h), BackupJobFailed |

To add Alertmanager notifications (Slack, PagerDuty, etc.) — add a `roles/alertmanager` role and point `alerting.alertmanagers` in `prometheus.yml`.

## Security

> ⚠️ **Lab note:** `group_vars/vault_example.yml` contains plain-text example
> credentials for demonstration purposes only.

- `group_vars/vault.yml` must be encrypted with `ansible-vault` and **never committed to git**
- `.gitignore` already excludes `vault.yml` and `.vault_pass`
- WAL-G credentials use `.pgpass` instead of `PGPASSWORD` in the config JSON
- Use strong unique passwords for all services before any production use

```bash
# Vault operations
ansible-vault encrypt group_vars/vault.yml
ansible-vault view    group_vars/vault.yml
ansible-vault edit    group_vars/vault.yml
```

## Project Structure

```
postgres-cluster/
├── Vagrantfile                  # 3-node VirtualBox lab definition
├── ansible.cfg                  # Ansible defaults (vault_password_file, no become_ask_pass)
├── inventory.ini                # Host definitions (192.168.56.10-12)
├── site.yml                     # Main playbook (9 steps)
├── .vault_pass                  # Vault password (git-ignored, create manually)
├── group_vars/
│   ├── all.yml                  # Non-secret variables
│   ├── vault.yml                # Encrypted secrets (git-ignored)
│   └── vault_example.yml        # Example vault structure
└── roles/
    ├── common/                  # Base packages, /etc/hosts
    ├── etcd/                    # etcd cluster (all 3 nodes)
    ├── patroni/                 # PostgreSQL 16 + Patroni + restore_command
    ├── pgbouncer/               # Connection pooler (transaction mode)
    ├── haproxy/                 # Load balancer + R/W split
    ├── postgres_exporter/       # Prometheus exporter
    ├── prometheus/              # Metrics + alert rules (patroni/postgresql/walg)
    ├── grafana/                 # Dashboards + provisioning
    ├── minio/                   # S3-compatible backup storage
    └── walg/                    # Backup agent + .pgpass + cron schedule
```

## Author

Vlad Samarets — DevOps Engineer
