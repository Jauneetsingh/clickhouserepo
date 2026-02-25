# Step-by-Step Troubleshooting Guide

Run each command one at a time. Read the expected output and notes before moving to the next step.

Each phase uses **different container names and ports** so they don't conflict with each other.

| Phase | Keeper Container | Server Container | Ports |
|-------|-----------------|-----------------|-------|
| 1 (Broken) | `broken-keeper` | `broken-server` | 18123, 19000, 19181 |
| 2 (Partial Fix) | `partial-keeper` | `partial-server` | 28123, 29000, 29181 |
| 3 (Full Fix) | `fixed-keeper` | `fixed-server` | 38123, 39000, 39181 |

> All commands assume you are in the project root: `cd ~/clickhouse1`

---

## PHASE 1: Start with the Customer's Broken Keeper Config

### Step 1 — Start the broken stack

```bash
cd ~/clickhouse1
docker compose -f docker-compose-troubleshoot.yml up -d
```

**Expected:** Both containers get created and started without Docker errors.

---

### Step 2 — Check container status

```bash
docker ps -a --format "table {{.Names}}\t{{.Status}}" --filter name=broken
```

**Expected:**
```
NAMES            STATUS
broken-server    Up X seconds
broken-keeper    Exited (232) X seconds ago
```

**Observation:** Keeper has **crashed with exit code 232**. The server is running but Keeper is dead.

---

### Step 3 — Check Keeper logs to find the error

```bash
docker logs broken-keeper
```

**Expected output (key line):**
```
SAXParseException: Invalid token in '/etc/clickhouse-keeper/keeper_config.xml', line 10 column 10
```

**Observation:** The XML configuration file cannot be parsed. There is a syntax error at line 10.

---

### Step 4 — Look at the broken config file to understand what's wrong

```bash
cat broken-keeper-config/keeper_config.xml
```

**Observation:** Look at lines 9-10. The tag `<keeper_server>` is split across two lines as:
```xml
<keeper
server>
```
XML does not allow tag names to span multiple lines. The same problem repeats for every tag with underscores: `listen_host`, `server_id`, `log_storage_path`, `snapshot_storage_path`, `coordination_settings`, `operation_timeout_ms`, `session_timeout_ms`, `raft_logs_level`, `raft_configuration`.

---

### Step 5 — Confirm the server can't reach Keeper

```bash
docker exec broken-server clickhouse-client --query "SELECT * FROM system.zookeeper_connection"
```

**Expected:**
```
Code: 999. DB::Exception: Cannot use any of provided ZooKeeper nodes. (KEEPER_EXCEPTION)
```

**Observation:** Server is running but has no Keeper coordination.

---

### Step 6 — Stop the broken stack

```bash
docker compose -f docker-compose-troubleshoot.yml down -v
```

---

## PHASE 2: Fix #1 — Repair the XML Tags (Partial Fix)

We fixed all the broken XML tag names but kept `listen_host` inside `<keeper_server>` (exactly as the customer intended). Let's see if that's enough.

### Step 7 — Look at the partially fixed config

```bash
cat partial-fix-keeper-config/keeper_config.xml
```

**Observation:** All tags are now on single lines. But notice `<listen_host>0.0.0.0</listen_host>` is on line 10 inside `<keeper_server>`. Keep this in mind.

---

### Step 8 — Start with the partially fixed config

```bash
docker compose -f docker-compose-partial-fix.yml up -d
```

---

### Step 9 — Check container status (wait 5 seconds first)

```bash
sleep 5
docker ps -a --format "table {{.Names}}\t{{.Status}}" --filter name=partial
```

**Expected:**
```
NAMES             STATUS
partial-server    Up X seconds
partial-keeper    Up X seconds
```

**Observation:** Both containers are now running. The XML fix worked — Keeper no longer crashes.

---

### Step 10 — But can the server actually connect to Keeper?

```bash
docker exec partial-server clickhouse-client --query "SELECT * FROM system.zookeeper_connection"
```

**Expected:**
```
Code: 999. DB::Exception: All connection tries failed while connecting to ZooKeeper.
Poco::Exception. Code: 1000, e.code() = 111, Connection refused
```

**Observation:** Keeper is running but the server gets **Connection refused**. There's a second issue.

---

### Step 11 — Check what address Keeper is listening on

```bash
docker exec partial-keeper grep -i "Listening" /var/log/clickhouse-keeper/clickhouse-keeper.log
```

**Expected:**
```
Application: Listening for Keeper (tcp): [::1]:9181
Application: Listening for Keeper (tcp): 127.0.0.1:9181
```

**Observation:** Keeper is listening on `127.0.0.1` (localhost only). The server is in a different container with a different IP, so it cannot reach `127.0.0.1`. The `<listen_host>0.0.0.0</listen_host>` was **ignored** because it was inside `<keeper_server>` — it must be at the root `<clickhouse>` level.

---

### Step 12 — Verify by checking the Keeper container's network IP

```bash
docker exec partial-server ping -c 1 partial-keeper
```

**Observation:** The server resolves `partial-keeper` to something like `172.x.x.x`, not `127.0.0.1`. That confirms why it can't connect.

---

### Step 13 — Stop the partial-fix stack

```bash
docker compose -f docker-compose-partial-fix.yml down -v
```

---

## PHASE 3: Fix #2 — Move `listen_host` to Root Level (Full Fix)

### Step 14 — Look at the fully fixed Keeper config

```bash
cat clickhouse-keeper/keeper_config.xml
```

**Key difference:** Line 2 — `<listen_host>0.0.0.0</listen_host>` is now directly under `<clickhouse>`, NOT inside `<keeper_server>`.

---

### Step 15 — Start the fully fixed stack

```bash
docker compose up -d
```

---

### Step 16 — Wait for startup and check container status

```bash
sleep 10
docker ps -a --format "table {{.Names}}\t{{.Status}}" --filter name=fixed
```

**Expected:** Both `fixed-keeper` and `fixed-server` are `Up`.

---

### Step 17 — Verify Keeper is listening on 0.0.0.0

```bash
docker exec fixed-keeper grep -i "Listening" /var/log/clickhouse-keeper/clickhouse-keeper.log
```

**Expected:**
```
Application: Listening for Keeper (tcp): 0.0.0.0:9181
```

**Observation:** Now Keeper listens on ALL interfaces, making it reachable from the server container.

---

### Step 18 — Verify the server connects to Keeper successfully

```bash
docker exec fixed-server clickhouse-client --query "SELECT host, port, is_expired FROM system.zookeeper_connection"
```

**Expected:**
```
fixed-keeper    9181    0
```

**Observation:** Connection established. `is_expired = 0` means the session is healthy.

---

## PHASE 4: Verify All Customer Requirements

### Step 19 — Requirement 1: Database limit is set to 3

```bash
docker exec fixed-server clickhouse-client --query "SELECT name, value FROM system.server_settings WHERE name='max_database_num_to_throw'"
```

**Expected:**
```
max_database_num_to_throw    3
```

---

### Step 20 — Requirement 1: Prove the limit works

```bash
docker exec fixed-server clickhouse-client --query "CREATE DATABASE test_db1"
```
```bash
docker exec fixed-server clickhouse-client --query "CREATE DATABASE test_db2"
```
```bash
docker exec fixed-server clickhouse-client --query "CREATE DATABASE test_db3"
```

**Expected:** First two succeed. Third fails with:
```
Code: 725. DB::Exception: Too many databases. The limit is set to 3,
the current number of databases is 3. (TOO_MANY_DATABASES)
```

---

### Step 21 — Clean up test databases

```bash
docker exec fixed-server clickhouse-client --query "DROP DATABASE IF EXISTS test_db1"
docker exec fixed-server clickhouse-client --query "DROP DATABASE IF EXISTS test_db2"
```

---

### Step 22 — Requirement 2a: Admin user works with no limits

```bash
docker exec fixed-server clickhouse-client --user admin --password admin_password --query "SELECT currentUser(), 'connected'"
```

**Expected:**
```
admin    connected
```

---

### Step 23 — Requirement 2b: Developer user has correct settings

```bash
docker exec fixed-server clickhouse-client --user developer --password developer_password --query "SELECT name, value FROM system.settings WHERE name IN ('max_execution_time', 'max_memory_usage')"
```

**Expected:**
```
max_execution_time    0.5
max_memory_usage      104857600
```

---

### Step 24 — Requirement 2b-i: Prove 500ms timeout works

```bash
docker exec fixed-server clickhouse-client --user developer --password developer_password --query "SELECT sleep(2)"
```

**Expected:**
```
Code: 159. DB::Exception: Timeout exceeded: elapsed ... ms, maximum: 500 ms. (TIMEOUT_EXCEEDED)
```

---

### Step 25 — Requirement 2b-i: Prove 100MB memory limit works

```bash
docker exec fixed-server clickhouse-client --user developer --password developer_password --query "SELECT count() FROM (SELECT arrayJoin(range(50000000)) AS x ORDER BY x)"
```

**Expected:**
```
Code: 241. DB::Exception: Query memory limit exceeded: would use ~190 MiB,
maximum: 100.00 MiB. (MEMORY_LIMIT_EXCEEDED)
```

---

### Step 26 — Requirement 2b-ii: Admin sees system database in system.tables

```bash
docker exec fixed-server clickhouse-client --user admin --password admin_password --query "SELECT DISTINCT database FROM system.tables ORDER BY database"
```

**Expected:** Shows `INFORMATION_SCHEMA`, `information_schema`, `system`.

---

### Step 27 — Requirement 2b-ii: Developer does NOT see system database

```bash
docker exec fixed-server clickhouse-client --user developer --password developer_password --query "SELECT DISTINCT database FROM system.tables ORDER BY database"
```

**Expected:** Shows `INFORMATION_SCHEMA`, `information_schema` only. **No `system`.**

---

## PHASE 5: Cleanup

### Step 28 — Stop everything

```bash
docker compose down -v
```

---

## Summary of Issues Found

| # | Issue | Symptom | Fix |
|---|-------|---------|-----|
| 1 | XML tags split across lines | `SAXParseException`, exit code 232 | Put each tag name on a single line |
| 2 | `listen_host` inside `keeper_server` | Keeper listens on 127.0.0.1 only, connection refused | Move `listen_host` to root `<clickhouse>` level |
