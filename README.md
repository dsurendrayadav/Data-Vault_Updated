# DataVault

A multi-database operations dashboard with query optimizer and
replication-health monitor across PostgreSQL + MongoDB.

- **3-node PostgreSQL** cluster: 1 primary + 2 streaming-replication replicas
- **3-shard MongoDB** cluster: config server + 3 shards + 1 mongos router
- **Python/Streamlit dashboard**: slow-query identifier, index recommendations,
  replication lag graphs, shard-balance graphs
- **Backup & PITR runbook** + **scripted failover demo** for both databases

---

## Requirements

- Docker Desktop (or Docker Engine) **20.10+**, with **Docker Compose v2**
  (the `docker compose` command, not the old standalone `docker-compose`)
- ~4 GB of free RAM for the containers
- Ports free on your machine: `5432`, `5433`, `5434`, `27017`, `8501`
- `psql` and `mongosh` installed locally if you want to run the helper
  scripts (`scripts/generate_load.sh`, manual queries) from your host
  rather than through `docker exec`

Verify Docker Compose v2 is available:
```bash
docker compose version
```

---

## 1. Unzip and enter the project

```bash
unzip datavault.zip
cd datavault
```

## 2. Start everything with one command

```bash
chmod +x scripts/*.sh
./scripts/setup.sh
```

This script, in order:
1. Builds the custom Postgres primary/replica images and the dashboard image.
2. Starts `pg-primary` and waits until it reports `healthy` (this is when
   the seed data — 5,000 customers / 150,000 orders — has finished loading).
3. Starts `pg-replica1` and `pg-replica2`, which each run `pg_basebackup`
   against the primary and come up in standby/streaming mode automatically.
4. Starts the Mongo config server, all 3 shards, and the `mongos` router.
5. Runs `mongo-setup`, a one-shot container that initializes all replica
   sets, registers the 3 shards with the router, enables sharding on the
   `datavault` database, and seeds ~20k products / ~100k events.
6. Starts the Streamlit dashboard.

**First run takes a few minutes** (image builds + seeding 150k Postgres
rows + 120k Mongo docs). You can watch progress with:

```bash
docker compose logs -f
```

If you'd rather not use the helper script, the equivalent manual sequence is:

```bash
docker compose build
docker compose up -d pg-primary
# wait for it to become healthy:
docker inspect -f '{{.State.Health.Status}}' dv-pg-primary
docker compose up -d pg-replica1 pg-replica2
docker compose up -d mongo-configsvr mongo-shard1 mongo-shard2 mongo-shard3 mongo-router
docker compose up -d mongo-setup
docker compose up -d dashboard
```

## 3. Open the dashboard

```
http://localhost:8501
```

Tabs available:
- **Overview** — cluster topology + live health checks for both databases
- **PostgreSQL Slow Queries** — ranked `pg_stat_statements` output, plus a
  box to paste any SQL and get an `EXPLAIN`-based index recommendation
- **Index Recommendations** — cluster-wide scan of tables with a bad
  seq-scan/index-scan ratio
- **Replication Lag** — live WAL lag (bytes + seconds) per replica
- **MongoDB Shard Balance** — chunk distribution per shard, per collection
- **MongoDB Server Status** — `serverStatus()` metrics + per-shard replica
  set health
- **Runbook** — the backup/PITR and failover docs, rendered in-app

## 4. Generate live traffic (recommended)

The dashboard is much more interesting with real, ongoing load. In a
separate terminal:

```bash
./scripts/generate_load.sh
```

This continuously inserts orders and runs a couple of deliberately
un-indexed lookup queries (`WHERE customer_email = ...`, `WHERE status =
...`) against the primary, so the **Slow Queries** and **Replication Lag**
tabs have real, moving data. Leave it running while you explore the
dashboard; stop it with `Ctrl+C` any time.

## 5. Try the query optimizer

In the **PostgreSQL Slow Queries** tab, paste (or use the pre-filled):
```sql
SELECT * FROM orders WHERE customer_email = 'user42@example.com' AND status = 'shipped';
```
Click **Explain query** — you'll get the raw JSON plan plus a concrete
`CREATE INDEX ...` recommendation, because `orders` intentionally ships
without indexes on `customer_email` or `status`. Try running the suggested
`CREATE INDEX` directly against Postgres and re-running `EXPLAIN ANALYZE`
to see the plan switch from `Seq Scan` to `Index Scan`:

```bash
docker exec -it dv-pg-primary psql -U datavault -d datavault \
  -c "CREATE INDEX idx_orders_email_status ON orders (customer_email, status);"
```

## 6. Run the backup / PITR runbook

```bash
./scripts/backup.sh
# ... make some changes / let generate_load.sh run for a bit ...
./scripts/restore_pitr.sh ./backups/base_<timestamp> "2026-07-14 09:15:00"
docker logs -f dv-pg-restore
```

Full explanation: `runbook/BACKUP_PITR_RUNBOOK.md`.

## 7. Run the failover demo

```bash
./scripts/failover_demo.sh
```

This stops `pg-primary`, promotes `pg-replica1` to primary, then steps down
the MongoDB `shard1RS` primary and shows the re-election. Full narrated
walkthrough (and how to restore the 3-node Postgres topology afterwards):
`runbook/FAILOVER_DEMO.md`.

## 8. Tear down

```bash
docker compose down          # stop + remove containers, keep data volumes
docker compose down -v       # also wipe all data volumes (full reset)
```

---

## Project layout

```
datavault/
├── docker-compose.yml          # orchestrates the whole stack
├── postgres/
│   ├── primary/                # Dockerfile, postgresql.conf, pg_hba.conf,
│   │                           # init script (replication role + extension),
│   │                           # seed-data.sql (customers/orders schema)
│   └── replica/                # Dockerfile + entrypoint that runs
│                                # pg_basebackup and starts in standby mode
├── mongo/
│   ├── init-cluster.sh          # master script: initiates all replica sets,
│   │                            # registers shards, enables sharding, seeds data
│   ├── init-configsvr.js / init-shard{1,2,3}.js / init-mongos.js
│   └── seed-data.js
├── dashboard/
│   ├── app.py                   # Streamlit UI (7 tabs)
│   ├── pg_monitor.py             # pg_stat_statements / pg_stat_replication queries
│   ├── mongo_monitor.py          # serverStatus / chunk-balance / replSet status
│   ├── query_optimizer.py        # EXPLAIN-plan-driven CREATE INDEX recommender
│   ├── requirements.txt
│   └── Dockerfile
├── scripts/
│   ├── setup.sh                  # one-command bring-up
│   ├── generate_load.sh          # continuous write/read traffic generator
│   ├── backup.sh / restore_pitr.sh
│   └── failover_demo.sh
└── runbook/
    ├── BACKUP_PITR_RUNBOOK.md
    └── FAILOVER_DEMO.md
```

## Design notes / simplifications

- Each MongoDB shard is a **single-node** replica set to keep the demo's
  resource footprint small. `runbook/FAILOVER_DEMO.md` shows exactly how to
  extend each shard to a real 3-node replica set for true no-downtime
  failover.
- PostgreSQL replica promotion is **manual** (`pg_ctl promote`), by design,
  so the failover mechanics are visible for the demo. Production setups
  typically add Patroni, repmgr, or pg_auto_failover for automatic,
  fenced failover.
- The `orders` table ships without indexes on `customer_email` or `status`
  on purpose, so the slow-query and index-recommendation panels have real
  problems to find.

## Troubleshooting

- **Dashboard shows "Cannot reach primary"**: `docker compose ps` to check
  `dv-pg-primary` is `healthy`; `docker compose logs pg-primary`.
- **Replicas never show as streaming**: check
  `docker compose logs pg-replica1` — most common cause is the primary not
  being healthy yet when the replica tried its `pg_basebackup`; restart the
  replica: `docker compose restart pg-replica1`.
- **Mongo shard balance tab is empty**: `mongo-setup` is a one-shot
  container — check it completed successfully with
  `docker compose logs mongo-setup`; re-run with `docker compose up mongo-setup`
  if it exited early.
- **Port already in use**: another Postgres/Mongo instance is likely
  running locally — stop it, or edit the host-side port mappings in
  `docker-compose.yml`.
