# PostgreSQL HA Cluster

> 🇬🇧 [English version](README.md)

Production-grade кластер високої доступності PostgreSQL (HA), який розгортається за допомогою Ansible.
Це лабораторний проєкт, що показує реальний підхід до автоматизації інфраструктури.
Усе оточення піднімається локально через **Vagrant + VirtualBox** — без потреби в хмарному акаунті.

## Архітектура

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

## Стек

| Компонент | Версія | Роль |
|---|---|---|
| PostgreSQL | 16 | Основна база даних |
| Patroni | latest | HA менеджер з авто-фейловером |
| etcd | 3 вузли | Розподілене сховище конфігурації / DCS |
| pgBouncer | latest | Пулінг зʼєднань (transaction mode) |
| HAProxy | latest | Балансувальник навантаження, R/W split |
| WAL-G | v3.0.0 | Бекапи та WAL-архівація |
| MinIO | latest | S3-сумісне сховище для бекапів |
| Prometheus | latest | Збір метрик + alert rules |
| Grafana | latest | Візуалізація метрик |

## Ключові можливості

- **Zero data loss** — `synchronous_mode_strict: true` у Patroni DCS
- **Авто-фейловер** — Patroni + etcd з кворумом при виборі лідера
- **Пулінг зʼєднань** — pgBouncer у transaction mode, `scram-sha-256` авторизація, PG `max_connections=50`
- **R/W розділення** — HAProxy віддає записи на :5000, читання — на :5001
- **WAL archiving** — безперервна відправка WAL у MinIO через WAL-G
- **Щоденні бекапи** — повний base backup о 02:00, зберігаються останні 7, стиснення brotli
- **PITR-ready** — `restore_command` налаштовано для Point-in-Time Recovery через WAL-G
- **Алерти** — правила Prometheus для Patroni, PostgreSQL і збоїв WAL-G бекапів
- **Управління секретами** — Ansible Vault для всіх паролів

## Вимоги

### Хост-машина

- [VirtualBox](https://www.virtualbox.org/) 6.1+
- [Vagrant](https://www.vagrantup.com/) 2.3+
- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/) 2.14+
- ~6 GB вільної RAM (haproxy: 1 GB, pg1+pg2: по 2 GB)
- ~15 GB вільного дискового простору

```bash
# Перевірка версій
virtualbox --help | head -1
vagrant --version
ansible --version
```

## Швидкий старт

### 1. Клонувати репозиторій

```bash
git clone https://github.com/samarets-vlad/postgres-cluster
cd postgres-cluster
```

### 2. Налаштувати секрети

```bash
# Скопіювати приклад vault і заповнити паролі
cp group_vars/vault_example.yml group_vars/vault.yml
nano group_vars/vault.yml

# Зашифрувати через ansible-vault і зберегти пароль від vault
ansible-vault encrypt group_vars/vault.yml
echo "your_vault_password" > .vault_pass
chmod 600 .vault_pass
```

> ⚠️ `.vault_pass` та `group_vars/vault.yml` вже внесені до `.gitignore`.

### 3. Підняти кластер

```bash
vagrant up
```

Vagrant зробить:
1. Створить 3 VM з Ubuntu 22.04 у мережі `192.168.56.0/24` (host-only)
2. Створить користувача `vlad` із passwordless sudo на кожній VM
3. Дочекається підняття всіх VM і запустить `ansible-playbook site.yml` з хоста

Перший запуск займає **~15–20 хвилин** (встановлення пакетів, завантаження WAL-G, початковий бекап).

### 4. Перевірити стан кластера

```bash
# Зайти на pg1 і подивитися статус кластера
vagrant ssh pg1
sudo -u postgres patronictl -c /etc/patroni/patroni.yml list
# Очікується: pg1=Leader, pg2=Sync Standby, Lag=0

# Точка запису (тільки primary)
psql -h 192.168.56.10 -p 5000 -U admin -d postgres -c "SELECT inet_server_addr();"

# Точка читання (репліка)
psql -h 192.168.56.10 -p 5001 -U admin -d postgres -c "SELECT inet_server_addr();"

# Подивитися список бекапів
vagrant ssh pg1 -c "sudo -u postgres wal-g --config /etc/wal-g/walg.json backup-list"
```

## Повторний запуск Ansible без пересоздання VM

Якщо змінюєш ролі й хочеш пере застосувати їх без видалення VM:

```bash
# Повторно запустити provision усіх VM
vagrant provision

# Або запустити ansible-playbook напряму (VM вже мають бути підняті)
ansible-playbook site.yml --vault-password-file .vault_pass

# Запустити лише одну роль (наприклад, WAL-G)
ansible-playbook site.yml --vault-password-file .vault_pass --tags walg

# Запуск лише для одного хоста
ansible-playbook site.yml --vault-password-file .vault_pass --limit pg1
```

## Керування VM

```bash
vagrant status          # показати стан VM
vagrant halt            # акуратно вимкнути всі VM
vagrant up              # підняти існуючі VM (без reprovision)
vagrant destroy -f      # видалити всі VM (дані будуть втрачені)
vagrant ssh haproxy     # зайти на конкретну VM
```

## Тест фейловеру

```bash
# Дивитися за кластером у реальному часі (в одному терміналі)
vagrant ssh pg1 -c "watch -n2 'sudo -u postgres patronictl -c /etc/patroni/patroni.yml list'"

# Імітувати падіння primary (інший термінал)
vagrant ssh pg1 -c "sudo systemctl stop patroni"
# pg2 стає Leader за ~10-15 секунд

# Повернути pg1 в кластер
vagrant ssh pg1 -c "sudo systemctl start patroni"
# pg1 повертається як Sync Standby
```

## Доступ до сервісів

| Сервіс | URL | Креденшіали |
|---|---|---|
| Grafana | http://192.168.56.10:3000 | admin / див. vault.yml |
| HAProxy Stats | http://192.168.56.10:7000 | admin / див. vault.yml |
| MinIO Console | http://192.168.56.10:9001 | див. vault.yml |
| Prometheus | http://192.168.56.10:9090 | без авторизації |
| Alertmanager | — | не налаштовано (алерти залишаються в Prometheus) |

## Алерти

Файли з правилами Prometheus розгортаються в `/etc/prometheus/rules/` і містять:

| Файл правил | Алерти |
|---|---|
| `patroni.yml` | LeaderMissing, ClusterUnhealthy, StandbyNotInSync, ReplicationLag, EtcdUnhealthy |
| `postgresql.yml` | PostgresDown, ConnectionsNearLimit (>80%), ReplicationLagCritical, Deadlocks, TxIDWraparound, TableBloat |
| `walg.yml` | BackupMissing (>25h), BackupJobFailed |

Щоб додати Alertmanager (Slack, PagerDuty тощо) — можна створити роль `roles/alertmanager` і налаштувати `alerting.alertmanagers` у `prometheus.yml`.

## Безпека

> ⚠️ **Lab note:** `group_vars/vault_example.yml` містить приклади креденшіалів у відкритому вигляді.
> Використовувати тільки для локальної лабораторії.

- `group_vars/vault.yml` обовʼязково має бути зашифрований через `ansible-vault` і **ніколи не комітитись в git**
- `.gitignore` вже виключає `vault.yml` та `.vault_pass`
- WAL-G використовує `.pgpass` замість `PGPASSWORD` у конфігураційному JSON
- Перед будь-яким умовно-продакшен використанням потрібно задати унікальні та сильні паролі

```bash
# Робота з vault
ansible-vault encrypt group_vars/vault.yml
ansible-vault view    group_vars/vault.yml
ansible-vault edit    group_vars/vault.yml
```

## Структура проєкту

```
postgres-cluster/
├── Vagrantfile                  # Опис 3-вузлової лабораторії VirtualBox
├── ansible.cfg                  # Налаштування Ansible (vault_password_file, без become_ask_pass)
├── inventory.ini                # Опис хостів (192.168.56.10-12)
├── site.yml                     # Головний playbook (9 кроків)
├── .vault_pass                  # Пароль до vault (не в git, створюється локально)
├── group_vars/
│   ├── all.yml                  # Звичайні (не секретні) змінні
│   ├── vault.yml                # Зашифровані секрети (не в git)
│   └── vault_example.yml        # Приклад структури vault
└── roles/
    ├── common/                  # Базові пакети, /etc/hosts
    ├── etcd/                    # Кластер etcd (усі 3 вузли)
    ├── patroni/                 # PostgreSQL 16 + Patroni + restore_command
    ├── pgbouncer/               # Пулінг зʼєднань (transaction mode)
    ├── haproxy/                 # Балансувальник + R/W split
    ├── postgres_exporter/       # Prometheus exporter
    ├── prometheus/              # Метрики + alert rules (patroni/postgresql/walg)
    ├── grafana/                 # Дашборди + provisioning
    ├── minio/                   # S3-сумісне сховище для бекапів
    └── walg/                    # Backup agent + .pgpass + cron розклад
```

## Author

Vlad Samarets — DevOps Engineer
