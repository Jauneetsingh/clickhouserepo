# ClickHouse Keeper — Configuration Fix

## The Problem

The customer provided a Keeper configuration that was not working. After fixing the PDF copy-paste formatting issues (broken XML tags), the Keeper started but was **unreachable** from the ClickHouse server.

**Error from ClickHouse server:**
```
Code: 999. DB::Exception: All connection tries failed while connecting to ZooKeeper.
Poco::Exception. Code: 1000, e.code() = 111, Connection refused
```

## Root Cause

`<listen_host>0.0.0.0</listen_host>` was placed **inside** `<keeper_server>`.

```xml
<clickhouse>
    <keeper_server>
        <listen_host>0.0.0.0</listen_host>   <!-- WRONG LOCATION -->
        <tcp_port>9181</tcp_port>
        ...
    </keeper_server>
</clickhouse>
```

When placed inside `<keeper_server>`, Keeper **ignores** the setting and defaults to listening on `127.0.0.1` (localhost only). Since the ClickHouse server runs in a separate Docker container with a different IP, it cannot connect.

**Keeper log confirmed the issue:**
```
Application: Listening for Keeper (tcp): [::1]:9181
Application: Listening for Keeper (tcp): 127.0.0.1:9181
```

## The Fix

Move `<listen_host>` to the root `<clickhouse>` level:

```xml
<clickhouse>
    <listen_host>0.0.0.0</listen_host>       <!-- CORRECT LOCATION -->
    <keeper_server>
        <tcp_port>9181</tcp_port>
        ...
    </keeper_server>
</clickhouse>
```

## How We Verified It Worked

**1. Keeper now listens on all interfaces:**
```bash
docker exec fixed-keeper grep -i "Listening" /var/log/clickhouse-keeper/clickhouse-keeper.log
```
```
Application: Listening for Keeper (tcp): 0.0.0.0:9181
```

**2. ClickHouse server connects successfully:**
```bash
docker exec fixed-server clickhouse-client --query "SELECT host, port, is_expired FROM system.zookeeper_connection"
```
```
fixed-keeper    9181    0
```

`is_expired = 0` confirms a healthy, active session between the server and Keeper.

## Files

| File | Description |
|------|-------------|
| `partial-fix-keeper-config/keeper_config.xml` | Wrong config — `listen_host` inside `keeper_server` |
| `clickhouse-keeper/keeper_config.xml` | Correct config — `listen_host` at root level |
