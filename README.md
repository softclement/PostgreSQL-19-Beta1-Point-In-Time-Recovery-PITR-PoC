# 🐘 PostgreSQL 19 Beta1 — Point-In-Time Recovery (PITR) PoC

> **Platform:** WSL2 + Docker Desktop
> **PostgreSQL Version:** 19 Beta1
> **Goal:** A clean, end-to-end PITR walkthrough anyone can follow — from zero to recovery to cleanup.

---

## 📋 Table of Contents

1. [What is PITR?](#1-what-is-pitr)
2. [What's New in PostgreSQL 19 Beta1](#2-whats-new-in-postgresql-19-beta1)
3. [Architecture](#3-architecture)
4. [Prerequisites](#4-prerequisites)
5. [Project Setup](#5-project-setup)
6. [Phase 1 — Start Primary Container](#6-phase-1--start-primary-container)
7. [Phase 2 — Verify WAL Archiving](#7-phase-2--verify-wal-archiving)
8. [Phase 3 — Generate Data & Base Backup](#8-phase-3--generate-data--base-backup)
9. [Phase 4 — Simulate Disaster](#9-phase-4--simulate-disaster)
10. [Phase 5 — Perform PITR Recovery](#10-phase-5--perform-pitr-recovery)
11. [Phase 6 — Validate Recovery](#11-phase-6--validate-recovery)
12. [Phase 7 — Explore PostgreSQL 19 New Features](#12-phase-7--explore-postgresql-19-new-features)
13. [Cleanup](#13-cleanup)
14. [PITR Quick Reference](#14-pitr-quick-reference)
15. [Troubleshooting](#15-troubleshooting)

---

## 1. What is PITR?

**Point-In-Time Recovery (PITR)** lets you restore a PostgreSQL database to any specific moment in history using:

| Component | Description |
|-----------|-------------|
| **Base Backup** | A full snapshot of the data directory at a point in time |
| **WAL Archive** | A continuous stream of change files recorded after the backup |
| **recovery_target_time** | The exact timestamp you want to restore to |

```
Timeline:
  [Insert Data]──►[Base Backup]──►[More WAL]──►[Disaster!]
                        │                           │
                        └──── PITR Recovery ────────►
                           (replay WAL to target time,
                            target MUST be after backup)
```

> ⚠️ **Key Rule:** The recovery target time must always be **after** the base backup completed, not before it.

---

## 2. What's New in PostgreSQL 19 Beta1

### `pg_stat_recovery` — Real-time Recovery Monitoring

A new system view that exposes live recovery progress:

```sql
SELECT * FROM pg_stat_recovery;
```

| Column | Description |
|--------|-------------|
| `recovery_start_time` | When recovery began |
| `recovery_target` | The configured target |
| `last_xact_replay_timestamp` | Timestamp of last replayed transaction |
| `recovery_state` | `streaming`, `waiting`, or `paused` |

### Async I/O Workers — `io_min_workers` / `io_max_workers`

Auto-scaling I/O workers make WAL replay during recovery faster:

```
io_method = worker
io_min_workers = 2
io_max_workers = 8
```

### Autovacuum Parallel Workers

```
autovacuum_max_parallel_workers = 4
```

### `WAIT FOR LSN`

Synchronize replica reads after recovery:

```sql
WAIT FOR LSN '0/3000000';
SELECT * FROM my_table;
```

### Password Expiration Warning

```
password_expiration_warning_threshold = 7d
```

> ⚠️ **Beta1 gotcha:** Use `7d` not `'7 days'` — spelled-out unit names cause a startup FATAL.

---

## 3. Architecture

```
WSL2 Host
│
└── Docker
    ├── pg19_primary   (always running, archives WAL)
    │   ├── Port 5432
    │   ├── Volume: pg19_data         → /var/lib/postgresql/data
    │   └── Volume: pg19_wal_archive  → /mnt/wal_archive
    │
    └── pg19_recovery  (started only during PITR exercise)
        ├── Port 5433
        ├── Volume: pg19_recovery_data → /var/lib/postgresql/data
        └── Volume: pg19_wal_archive   → /mnt/wal_archive (read-only, shared)

~/pitr-poc/
└── docker-compose.yml
```

---

## 4. Prerequisites

| Tool | Check |
|------|-------|
| WSL2 | `wsl --version` |
| Docker Desktop (27+) | `docker --version` |
| Docker Compose v2 | `docker compose version` |
| Python 3 | `python3 --version` |

Enable WSL2 integration: Docker Desktop → Settings → Resources → WSL Integration → enable your distro.

---

## 5. Project Setup

### 5.1 — Create project directory

```bash
mkdir -p ~/pitr-poc
cd ~/pitr-poc
```

### 5.2 — Create `docker-compose.yml`

Copy and paste this entire block exactly as-is:

```bash
python3 - << 'PYEOF'
content = """version: "3.9"

networks:
  pitr_network:
    driver: bridge

volumes:
  pg19_data:
  pg19_wal_archive:
  pg19_recovery_data:

services:

  pg19_primary:
    image: postgres:19beta1
    container_name: pg19_primary
    restart: unless-stopped
    networks:
      - pitr_network
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres123
      POSTGRES_DB: pitrdb
      PGDATA: /var/lib/postgresql/data
    volumes:
      - pg19_data:/var/lib/postgresql/data
      - pg19_wal_archive:/mnt/wal_archive
    entrypoint:
      - bash
      - -c
      - |
        chown postgres:postgres /mnt/wal_archive
        chmod 750 /mnt/wal_archive
        exec docker-entrypoint.sh "$$@"
      - --
    command:
      - postgres
      - -c
      - wal_level=replica
      - -c
      - archive_mode=on
      - -c
      - archive_command=cp %p /mnt/wal_archive/%f
      - -c
      - archive_timeout=60
      - -c
      - max_wal_senders=3
      - -c
      - wal_keep_size=128
      - -c
      - checkpoint_timeout=300
      - -c
      - io_method=worker
      - -c
      - io_min_workers=2
      - -c
      - io_max_workers=8
      - -c
      - autovacuum_max_parallel_workers=2
      - -c
      - password_expiration_warning_threshold=7d
      - -c
      - log_connections=on
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d pitrdb"]
      interval: 10s
      timeout: 5s
      retries: 10

  pg19_recovery:
    image: postgres:19beta1
    container_name: pg19_recovery
    restart: "no"
    profiles:
      - recovery
    networks:
      - pitr_network
    ports:
      - "5433:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres123
      PGDATA: /var/lib/postgresql/data
    volumes:
      - pg19_recovery_data:/var/lib/postgresql/data
      - pg19_wal_archive:/mnt/wal_archive:ro
    entrypoint:
      - bash
      - -c
      - |
        exec docker-entrypoint.sh "$$@"
      - --
    command:
      - postgres
"""
with open("/root/pitr-poc/docker-compose.yml" if __import__('os').path.exists('/root/pitr-poc') else
          __import__('os').path.expanduser("~/pitr-poc/docker-compose.yml"), "w") as f:
    f.write(content)
print("✅ docker-compose.yml written")
PYEOF
```

Verify it was written:

```bash
cat ~/pitr-poc/docker-compose.yml | head -5
```

---

## 6. Phase 1 — Start Primary Container

```bash
cd ~/pitr-poc

# Pull image and start
docker pull postgres:19beta1
docker compose up -d pg19_primary

# Wait for healthy status (up to 60s)
echo "Waiting for PostgreSQL to be ready..."
until docker exec pg19_primary pg_isready -U postgres -d pitrdb -q; do
  sleep 2
done
echo "✅ PostgreSQL is ready"

# Verify all PITR-critical settings are active
docker exec pg19_primary psql -U postgres -d pitrdb -c \
  "SHOW archive_mode; SHOW archive_command; SHOW wal_level;"
```

Expected output:
```
 archive_mode | on
 archive_command | cp %p /mnt/wal_archive/%f
 wal_level | replica
```

Create the replication user:

```bash
docker exec pg19_primary psql -U postgres -d pitrdb -c \
  "CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'repl123';"
```

---

## 7. Phase 2 — Verify WAL Archiving

```bash
# Force a WAL segment to be archived
docker exec pg19_primary psql -U postgres -d pitrdb -c "SELECT pg_switch_wal();"

sleep 5

# Check archive directory — should show at least one WAL file
docker exec pg19_primary ls -lh /mnt/wal_archive/

# Check archiver stats — archived_count should be >= 1, failed_count = 0
docker exec pg19_primary psql -U postgres -d pitrdb -c \
  "SELECT archived_count, last_archived_wal, last_archived_time, failed_count FROM pg_stat_archiver;"
```

Expected:
```
 archived_count | last_archived_wal        | failed_count
----------------+--------------------------+--------------
              1 | 000000010000000000000001 |            0
```

---

## 8. Phase 3 — Generate Data & Base Backup

> ⚠️ **Order matters:** Insert data FIRST, take backup SECOND, record safe_point THIRD.
> The recovery target must fall between the backup and the disaster — never before the backup.

### 8.1 — Insert initial data

```bash
docker exec pg19_primary psql -U postgres -d pitrdb -c "
  CREATE TABLE orders (
      id         SERIAL PRIMARY KEY,
      item       TEXT        NOT NULL,
      amount     NUMERIC     NOT NULL,
      created_at TIMESTAMPTZ DEFAULT now()
  );
  INSERT INTO orders (item, amount)
  SELECT 'Product-' || g, round((random() * 1000)::numeric, 2)
  FROM generate_series(1, 20) g;
  SELECT count(*) AS row_count FROM orders;
"
```

### 8.2 — Take base backup

```bash
docker exec pg19_primary pg_basebackup \
  -h localhost \
  -U replicator \
  -D /var/lib/postgresql/base_backup \
  -Fp -Xs \
  --checkpoint=fast \
  -v

echo "✅ Base backup complete"
docker exec pg19_primary du -sh /var/lib/postgresql/base_backup
```

### 8.3 — Record safe point (AFTER backup completes)

```bash
docker exec pg19_primary psql -U postgres -d pitrdb -c \
  "SELECT pg_create_restore_point('before_disaster'), now() AS safe_point;"
```

**⚠️ Copy the `safe_point` timestamp — you need it in Phase 5.**

Example: `2026-06-16 01:08:15.123456+00`

### 8.4 — Insert data that will be lost in the disaster

```bash
docker exec pg19_primary psql -U postgres -d pitrdb -c "
  INSERT INTO orders (item, amount) VALUES
    ('AFTER-BACKUP-1', 9999.99),
    ('AFTER-BACKUP-2', 8888.88);
  SELECT count(*) AS total_rows FROM orders;
"
```

---

## 9. Phase 4 — Simulate Disaster

```bash
# Note the disaster time
docker exec pg19_primary psql -U postgres -d pitrdb -c "SELECT now() AS disaster_time;"

# DROP the table — this is the disaster
docker exec pg19_primary psql -U postgres -d pitrdb -c "DROP TABLE orders;"

# Confirm it is gone
docker exec pg19_primary psql -U postgres -d pitrdb -c "\dt"

# Archive the WAL containing the DROP
docker exec pg19_primary psql -U postgres -d pitrdb -c "SELECT pg_switch_wal();"
sleep 5
docker exec pg19_primary ls -lh /mnt/wal_archive/
```

---

## 10. Phase 5 — Perform PITR Recovery

### 10.1 — Copy base backup into recovery volume

```bash
# Copy from primary's base_backup into the recovery volume
docker run --rm \
  --volumes-from pg19_primary \
  -v pitr-poc_pg19_recovery_data:/recovery_data \
  postgres:19beta1 \
  bash -c "
    cp -a /var/lib/postgresql/base_backup/. /recovery_data/
    echo '✅ Base backup copied'
    ls /recovery_data | head -5
  "
```

### 10.2 — Write recovery configuration

Replace `<YOUR_SAFE_POINT>` with the timestamp saved in Phase 3 Step 8.3:

```bash
RECOVERY_TARGET="2026-06-16 01:08:15.123456+00"   # ← paste your safe_point here

docker run --rm \
  -v pitr-poc_pg19_recovery_data:/recovery_data \
  postgres:19beta1 \
  bash -c "
    touch /recovery_data/recovery.signal
    cat >> /recovery_data/postgresql.auto.conf << EOF
restore_command = 'cp /mnt/wal_archive/%f %p'
recovery_target_time = '${RECOVERY_TARGET}'
recovery_target_action = 'promote'
recovery_target_inclusive = true
EOF
    echo '✅ Recovery config written'
    cat /recovery_data/postgresql.auto.conf
  "
```

Verify the config looks correct before proceeding:

```bash
docker run --rm \
  -v pitr-poc_pg19_recovery_data:/recovery_data \
  postgres:19beta1 \
  cat /recovery_data/postgresql.auto.conf
```

You should see all four lines: `restore_command`, `recovery_target_time`, `recovery_target_action`, `recovery_target_inclusive`.

### 10.3 — Start recovery container

```bash
docker compose --profile recovery up -d pg19_recovery

# Watch recovery logs
docker logs pg19_recovery --follow
```

Expected log sequence:
```
LOG:  starting point-in-time recovery to 2026-06-16 01:08:15+00
LOG:  restored log file "000000010000000000000006" from archive
LOG:  redo starts at 0/06000028
LOG:  consistent recovery state reached at 0/06000130
LOG:  database system is ready to accept read-only connections
LOG:  recovery stopping before commit of transaction ...
LOG:  pausing at the end of recovery
...
LOG:  database system is ready to accept connections   ← promoted!
```

Press `Ctrl+C` once you see "ready to accept connections".

The `cp: cannot stat '.../00000002.history': No such file or directory` messages are **normal** — PostgreSQL checks for a timeline history file that doesn't exist yet, then continues.

---

## 11. Phase 6 — Validate Recovery

```bash
# Connect to the recovered database on port 5433
docker exec pg19_recovery psql -U postgres -d pitrdb -c "
  -- 1. Table should exist again
  SELECT schemaname, tablename FROM pg_tables WHERE tablename = 'orders';

  -- 2. Row count (should be 20 — the 2 AFTER-BACKUP rows may or may not appear
  --    depending on whether your safe_point captured them)
  SELECT count(*) AS recovered_rows FROM orders;

  -- 3. Spot check data
  SELECT * FROM orders ORDER BY id DESC LIMIT 5;

  -- 4. Confirm promoted (not in recovery)
  SELECT pg_is_in_recovery() AS still_in_recovery;
"
```

Expected:
```
 tablename
-----------
 orders

 recovered_rows
----------------
             20

 still_in_recovery
-------------------
 f
```

🎉 **Recovery successful** — the `orders` table is back, the disaster DROP is reversed.

---

## 12. Phase 7 — Explore PostgreSQL 19 New Features

### `pg_stat_recovery` — New in PG19

To observe this view during recovery, repeat Phase 10 with `recovery_target_action = 'pause'` instead of `'promote'`, then query before promoting:

```sql
-- Connect to recovery container while paused (port 5433, read-only mode)
SELECT
    recovery_start_time,
    recovery_target,
    last_xact_replay_timestamp,
    recovery_state
FROM pg_stat_recovery;

-- Then promote manually when ready
SELECT pg_promote();
```

### Async I/O Settings

```bash
docker exec pg19_primary psql -U postgres -d pitrdb -c \
  "SHOW io_method; SHOW io_min_workers; SHOW io_max_workers;"
```

### WAL and LSN Queries

```bash
docker exec pg19_primary psql -U postgres -d pitrdb -c "
  SELECT
    pg_current_wal_lsn()                          AS current_lsn,
    pg_walfile_name(pg_current_wal_lsn())         AS current_wal_file,
    pg_size_pretty(pg_wal_lsn_diff(
      pg_current_wal_lsn(), '0/0'))               AS wal_generated;
"
```

### Named Restore Points

```sql
-- Create a restore point before any risky operation
SELECT pg_create_restore_point('before_migration_v2');

-- Then use it in recovery config instead of a timestamp:
-- recovery_target_name = 'before_migration_v2'
```

### Archiver Monitoring

```bash
docker exec pg19_primary psql -U postgres -d pitrdb -c "
  SELECT
    archived_count,
    last_archived_wal,
    last_archived_time,
    failed_count,
    last_failed_wal,
    last_failed_time
  FROM pg_stat_archiver;
"
```

---

## 13. Cleanup

```bash
cd ~/pitr-poc

# Stop and remove all containers, volumes, and networks
docker compose --profile recovery down -v 2>/dev/null || docker compose down -v

# Remove the image (optional)
docker rmi postgres:19beta1 2>/dev/null || true

# Remove project files
rm -rf ~/pitr-poc/

# Verify clean state
docker ps
docker volume ls | grep pg19
echo "✅ Cleanup complete"
```

---

## 14. PITR Quick Reference

### Correct Sequence (critical — do not reorder)

```
1. Insert initial data
2. Take base backup          ← pg_basebackup
3. Record safe_point         ← pg_create_restore_point + now()
4. Insert data that will be lost
5. Trigger disaster          ← DROP TABLE / DELETE / etc.
6. Archive final WAL         ← pg_switch_wal()
7. Copy backup to recovery volume
8. Write recovery.signal + postgresql.auto.conf
9. Start recovery container
10. Validate
```

### Key Configuration

```ini
# On primary (passed as -c flags to postgres process)
wal_level = replica
archive_mode = on
archive_command = cp %p /mnt/wal_archive/%f
archive_timeout = 60

# In recovery volume's postgresql.auto.conf
restore_command = cp /mnt/wal_archive/%f %p
recovery_target_time = '2026-06-16 01:08:15+00'
recovery_target_action = promote
recovery_target_inclusive = true
```

### Recovery Target Options

```ini
# By time (most common)
recovery_target_time = '2026-06-16 01:08:15+00'

# By named restore point
recovery_target_name = 'before_disaster'

# By transaction ID
recovery_target_xid = '500'

# By LSN
recovery_target_lsn = '0/3000000'
```

### Key Files in `$PGDATA`

| File | Purpose |
|------|---------|
| `recovery.signal` | Presence triggers recovery mode on startup |
| `postgresql.auto.conf` | Where recovery settings are written |
| `pg_wal/` | Live WAL (not the archive) |
| `backup_manifest` | Created by pg_basebackup, used for verification |

### Useful Queries

```sql
-- Force WAL archiving now
SELECT pg_switch_wal();

-- Archiver health
SELECT archived_count, failed_count, last_archived_wal FROM pg_stat_archiver;

-- Create named restore point
SELECT pg_create_restore_point('label');

-- Check if in recovery
SELECT pg_is_in_recovery();

-- Current WAL position
SELECT pg_current_wal_lsn(), pg_walfile_name(pg_current_wal_lsn());

-- Recovery progress (PG19 new view)
SELECT * FROM pg_stat_recovery;
```

---

## 15. Troubleshooting

### Container exits immediately on startup

```bash
docker logs pg19_primary --tail 30
```

**Cause:** Bad parameter in config.
**Common PG19 Beta1 issue:**

```
# ❌ causes FATAL
password_expiration_warning_threshold = '7 days'

# ✅ correct
password_expiration_warning_threshold = 7d
```

After any config fix, do a full clean restart:
```bash
docker compose down -v
docker compose up -d pg19_primary
```

### `archive_mode = off` despite config file existing

PostgreSQL ignores a mounted config file if `$PGDATA/postgresql.conf` exists (initdb generates one). **Solution:** pass all settings as `-c` flags in the `command:` list in `docker-compose.yml` — this always wins over any config file.

### WAL archive permission denied

```
cp: cannot create regular file '/mnt/wal_archive/...': Permission denied
```

**Cause:** Docker creates volumes as root; postgres user can't write.
**Fix:** The `entrypoint` in this guide runs `chown` before postgres starts. If you hit this on an existing container:

```bash
docker exec -u root pg19_primary chown postgres:postgres /mnt/wal_archive
docker exec -u root pg19_primary chmod 750 /mnt/wal_archive
docker exec pg19_primary psql -U postgres -c "SELECT pg_switch_wal();"
```

### `database "pitrdb" does not exist`

**Cause:** Container crashed before initdb completed, leaving a corrupt volume.
**Fix:**
```bash
docker compose down -v   # -v wipes the volume
docker compose up -d pg19_primary
```

### `FATAL: recovery ended before configured recovery target was reached`

**Cause:** Your `recovery_target_time` is set to a time **before** the base backup was taken.
**Fix:** The target must be **after** the backup checkpoint and **before** the disaster:

```
backup checkpoint time  <  recovery_target_time  <  disaster time
```

Check the backup checkpoint time in the logs:
```
LOG:  database system was interrupted; last known up at 2026-06-16 01:06:44 UTC
                                                                    ^^^^^^^^
                                                    target must be after this
```

### Recovery container runs `initdb` instead of recovering

**Cause:** `POSTGRES_DB` is set on the recovery container, which triggers the entrypoint to run `initdb` when it sees a non-empty data directory.
**Fix:** Do not set `POSTGRES_DB` on `pg19_recovery` — only `POSTGRES_USER`, `POSTGRES_PASSWORD`, and `PGDATA` are needed. The database already exists in the copied backup.

### `cp: cannot stat '.../00000002.history'` during recovery

This is **normal**. PostgreSQL checks for a timeline history file before falling back to the primary timeline. It is not an error — recovery continues successfully.

---

## References

- [PostgreSQL 19 Beta1 Release Announcement](https://www.postgresql.org/about/news/postgresql-19-beta-1-released-3313/)
- [Continuous Archiving and PITR — PG Docs](https://www.postgresql.org/docs/19/continuous-archiving.html)
- [pg_basebackup](https://www.postgresql.org/docs/19/app-pgbasebackup.html)
- [Recovery Configuration](https://www.postgresql.org/docs/19/recovery-config.html)
- [pg_stat_recovery View](https://www.postgresql.org/docs/19/monitoring-stats.html)

---

> ⚠️ PostgreSQL 19 Beta1 is pre-release software. Do not use in production.

*Last updated: June 2026 | PostgreSQL 19 Beta1 | WSL2 + Docker*
