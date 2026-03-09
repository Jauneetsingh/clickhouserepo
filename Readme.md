# ClickHouse Support Case: ACME Company Troubleshooting

## Table of Contents

1. [Case Summary](#case-summary)
2. [Research & Analysis](#research--analysis)
3. [Requirement 1: Database Limit](#requirement-1-limit-databases-to-3)
4. [Requirement 2: User Configuration](#requirement-2-user-configuration-admin--developer)
5. [Requirement 3: Keeper Configuration](#requirement-3-keeper-configuration)
6. [Deployment & Testing](#deployment--testing)
7. [Draft Customer Response](#draft-customer-response)

---

## Case Summary

**Customer:** ACME Company  
**Issue:** Deploying ClickHouse 25.8.1 with Docker, needing:
1. Limit databases to 3
2. Create two users (admin, developer) with specific permissions and limits
3. Deploy ClickHouse Keeper in a separate container (broken configuration provided)

---

## Research & Analysis

### Things That Stand Out

1. **Version 25.8.1**: The customer requested `25.8.1`, but this specific tag does not exist on Docker Hub. ClickHouse 25.8 is the LTS release, with the latest patch being `25.8.16.34`. The correct Docker tag to use is `25.8` or `25.8.16`.

2. **Database Limit Setting**: ClickHouse does not have a commonly documented setting for this. Through research of the ClickHouse source code (`src/Core/ServerSettings.cpp`), the setting `max_database_num_to_throw` was found. It defaults to `0` (no limit) and throws `TOO_MANY_DATABASES` (error code 725) when the limit is exceeded.

3. **Keeper Config XML Issues**: The provided Keeper configuration has **severe XML formatting issues** — tag names are broken across lines with underscores and spaces, making the XML unparseable. This is the primary reason the Keeper deployment fails.

4. **Keeper `listen_host` Placement**: Even after fixing the broken XML tags, the `<listen_host>` setting was placed inside `<keeper_server>`. In ClickHouse Keeper, `listen_host` must be at the `<clickhouse>` root level for Keeper to bind to external interfaces. Without this, Keeper only listens on `127.0.0.1` (localhost) and is unreachable from the ClickHouse server container.

### What I Understood From the Issue

- The customer is new to ClickHouse deployment and has a configuration that is syntactically broken.
- The XML was likely copy-pasted from a source that introduced line breaks and formatting artifacts into the tag names.
- The customer needs guidance on proper XML structure, user access control, and Keeper architecture.

### Replication

- All issues were **successfully replicated** using Docker containers.
- The broken Keeper config produces: `SAXParseException: Invalid token in '/etc/clickhouse-keeper/keeper_config.xml', line 10 column 10` (exit code 232).
- After fixing XML tags but keeping `listen_host` inside `<keeper_server>`, Keeper starts but only listens on `127.0.0.1:9181`, causing `Connection refused` from the ClickHouse server.

---

## Requirement 1: Limit Databases to 3

### Setting Used

```xml
<clickhouse>
    <max_database_num_to_throw>3</max_database_num_to_throw>
</clickhouse>
```

**File:** `clickhouse-server/config.d/custom.xml`

### How It Works

- This server setting enforces a hard limit on the number of non-system databases.
- When the count is exceeded, ClickHouse throws error `725 (TOO_MANY_DATABASES)`.
- System databases (`system`, `INFORMATION_SCHEMA`, `information_schema`) are NOT counted towards this limit.
- The `default` database IS counted, so with a limit of 3, the user can create 2 additional databases.
- There is also `max_database_num_to_warn` (default: 1000) which only generates a warning in `system.warnings`.

### Test Results

```
Creating db1... OK
Creating db2... OK
Creating db3... FAILED
  Code: 725. DB::Exception: Too many databases.
  The limit (server configuration parameter `max_database_num_to_throw`) is set to 3,
  the current number of databases is 3. (TOO_MANY_DATABASES)
```

---

## Requirement 2: User Configuration (admin & developer)

### 2a. Admin User

**Configuration** (`clickhouse-server/users.d/custom_users.xml`):

```xml
<admin>
    <password>admin_password</password>
    <networks>
        <ip>::/0</ip>
    </networks>
    <profile>default</profile>
    <quota>default</quota>
    <access_management>1</access_management>
</admin>
```

- Uses the `default` profile (no limits).
- `access_management>1` grants the ability to manage users, roles, row policies, etc.
- Same permission level as the `default` user.

### 2b. Developer User

**Configuration**:

```xml
<developer>
    <password>developer_password</password>
    <networks>
        <ip>::/0</ip>
    </networks>
    <profile>developer_profile</profile>
    <quota>default</quota>
</developer>
```

**Profile** with limits:

```xml
<developer_profile>
    <!-- Queries can't run more than 500ms -->
    <max_execution_time>0.5</max_execution_time>
    <!-- Queries can't consume more than 100MB of memory -->
    <max_memory_usage>104857600</max_memory_usage>
</developer_profile>
```

### 2b-i. Demonstration of Limits

**Query Timeout (500ms):**
```
SELECT sleep(2)
→ Code: 159. DB::Exception: Timeout exceeded: elapsed 1005.356584 ms, maximum: 500 ms.
  (TIMEOUT_EXCEEDED)
```

**Memory Limit (100MB):**
```
SELECT count() FROM (SELECT arrayJoin(range(50000000)) AS x ORDER BY x)
→ Code: 241. DB::Exception: Query memory limit exceeded: would use 190.82 MiB,
  maximum: 100.00 MiB. (MEMORY_LIMIT_EXCEEDED)
```

### 2b-ii. Row Policy on system.tables

The developer user should only see rows where `database != 'system'` in `system.tables`.

**Implementation** (via init script at container startup):

```sql
-- Developer can only see non-system databases
CREATE ROW POLICY IF NOT EXISTS policy_hide_system_tables
    ON system.tables
    FOR SELECT
    USING database != 'system'
    TO developer;

-- Admin and default users retain full visibility
CREATE ROW POLICY IF NOT EXISTS policy_allow_all_system_tables
    ON system.tables
    FOR SELECT
    USING 1
    TO admin, default;
```

**Important**: When ANY row policy is defined on a table, ALL users are affected. Users without an explicit permissive policy will see NO rows. This is why we must create the `policy_allow_all_system_tables` policy for admin/default.

**Test Results:**

| User      | Databases visible in `system.tables` |
|-----------|--------------------------------------|
| admin     | INFORMATION_SCHEMA, information_schema, system |
| developer | INFORMATION_SCHEMA, information_schema |

---

## Requirement 3: Keeper Configuration

### 3a. Configuration Issues Found

The customer-provided Keeper configuration has **two categories of issues**:

#### Issue 1: Broken XML Tags (Critical)

All compound tag names containing underscores are broken across multiple lines, making the XML completely unparseable. Examples:

| Broken Tag | Correct Tag |
|-----------|------------|
| `<keeper\nserver>` | `<keeper_server>` |
| `<listen\nhost>` | `<listen_host>` |
| `<server\nid>` | `<server_id>` |
| `<log_\nstorage\n_path>` | `<log_storage_path>` |
| `<snapshot\n_\nstorage\n_path>` | `<snapshot_storage_path>` |
| `<coordination\n_\nsettings>` | `<coordination_settings>` |
| `<operation\ntimeout\n_\nms>` | `<operation_timeout_ms>` |
| `<session\ntimeout\nms>` | `<session_timeout_ms>` |
| `<raft\n_\nlogs\nlevel>` | `<raft_logs_level>` |
| `<raft\n_\nconfiguration>` | `<raft_configuration>` |

Additionally, several **closing tags don't match their opening tags** (e.g., `</operation\ntimeout\nms>` instead of `</operation_timeout_ms>`).

**Error produced:**
```
SAXParseException: Invalid token in '/etc/clickhouse-keeper/keeper_config.xml', line 10 column 10
Exit code: 232
```

#### Issue 2: `listen_host` Placement (Logical)

Even after fixing the XML tags, `<listen_host>0.0.0.0</listen_host>` is placed **inside** the `<keeper_server>` block. For ClickHouse Keeper, `listen_host` must be at the **root `<clickhouse>` level** to take effect.

**Symptom:** Keeper starts successfully but only listens on `127.0.0.1:9181`, making it unreachable from other containers.

**Fix:** Move `<listen_host>` outside `<keeper_server>`:

```xml
<clickhouse>
    <listen_host>0.0.0.0</listen_host>  <!-- Must be at root level -->
    <logger>...</logger>
    <keeper_server>
        <!-- listen_host should NOT be here -->
        <tcp_port>9181</tcp_port>
        ...
    </keeper_server>
</clickhouse>
```

### 3b. What is ClickHouse Keeper and Why is it Required?

#### Purpose

ClickHouse Keeper is a **coordination service** (similar to Apache ZooKeeper) that provides:

1. **Distributed Coordination**: Manages consensus among ClickHouse replicas and shards in a cluster.
2. **Metadata Storage**: Stores metadata for ReplicatedMergeTree tables, including partition information, mutation logs, and replication queues.
3. **Leader Election**: Handles leader election for distributed DDL operations.
4. **Configuration Synchronization**: Ensures consistent configuration across cluster nodes.

#### When is Keeper Required?

- **ReplicatedMergeTree tables**: Any table using replication engines requires Keeper.
- **Distributed DDL**: Operations like `ON CLUSTER` require coordination.
- **Multi-node clusters**: Any setup with more than one ClickHouse server node.

#### Production Best Practices

1. **Odd Number of Nodes**: Deploy 3 or 5 Keeper nodes to maintain quorum. A 3-node cluster tolerates 1 failure; a 5-node cluster tolerates 2 failures.

2. **Dedicated Servers**: Run Keeper on dedicated machines, separate from ClickHouse servers, to avoid resource contention. Keeper is latency-sensitive and CPU/IO competition can cause session timeouts.

3. **Low-Latency Storage**: Use fast SSDs for Keeper's log and snapshot storage paths. Keeper performs synchronous disk writes for each operation.

4. **Network Proximity**: Place Keeper nodes in the same datacenter or with low-latency network connections. High latency between Keeper nodes increases commit times.

5. **Resource Allocation**: Keeper is lightweight but needs consistent resources. Allocate at least 2 CPU cores and 4GB RAM per Keeper node.

6. **Monitoring**: Monitor Keeper health using the four-letter commands (`ruok`, `stat`, `mntr`) or the ClickHouse system table `system.zookeeper_connection`.

7. **Separate from ZooKeeper**: If migrating from Apache ZooKeeper, ClickHouse Keeper is recommended as a drop-in replacement with better performance and tighter integration.

8. **Backup Snapshots**: Regularly back up the snapshot directory. Snapshots can be used to restore Keeper state.

9. **Session Timeouts**: Configure appropriate `session_timeout_ms` (default: 30000ms). Too low causes false disconnections under load; too high delays failure detection.

10. **Version Consistency**: Keep all Keeper nodes on the same ClickHouse version to avoid protocol incompatibilities.

---

## Deployment & Testing

### Project Structure

```
clickhouse1/
├── docker-compose.yml                    # Main deployment (working)
├── docker-compose-broken-keeper.yml      # For demonstrating the broken config
├── clickhouse-server/
│   ├── config.d/
│   │   └── custom.xml                    # Database limit + Keeper connection
│   ├── users.d/
│   │   └── custom_users.xml              # Admin + Developer users & profiles
│   └── init-db.sh                        # Row policy creation script
├── clickhouse-keeper/
│   └── keeper_config.xml                 # Fixed Keeper configuration
├── broken-keeper-config/
│   └── keeper_config.xml                 # Original broken config (for reference)
└── README.md                             # This file
```

### How to Run

```bash
# Start the full stack
docker compose up -d

# Wait ~10 seconds for initialization, then verify
docker exec clickhouse-server clickhouse-client --query "SELECT 1"

# Connect as admin
docker exec clickhouse-server clickhouse-client --user admin --password admin_password

# Connect as developer
docker exec clickhouse-server clickhouse-client --user developer --password developer_password

# Stop everything
docker compose down -v
```

### To Demonstrate the Broken Config

```bash
docker compose -f docker-compose-broken-keeper.yml up
# Observe: SAXParseException: Invalid token ... line 10 column 10
# Exit code: 232
```

---
