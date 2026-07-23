# 🛠️ KeyDB Incident & Troubleshooting Report

**Date:** July 23, 2026  
**Environment:** UAT  
**Server IP:** `10.0.1.78`  
**Service:** KeyDB  
**Status:** ✅ Resolved

---

# 📌 Executive Summary

During application deployment, the Node.js application was unable to connect to the KeyDB server.

Two issues were identified:

1. **KeyDB service crashed** with a Segmentation Fault while starting.
2. **Node.js application attempted to connect to the wrong port (6400)** whereas KeyDB was configured to listen on **6378**.

After analyzing the service logs and KeyDB configuration, the issue was resolved by modifying the configuration, removing corrupted persistence files, restarting the service, and updating the application configuration.

---

# 🔍 Root Cause Analysis

## Issue 1: KeyDB Service Crash (Segmentation Fault)

### Symptom

The KeyDB service failed to start.

```bash
sudo systemctl status keydb-server
```

Output showed:

```
code=dumped, signal=SEGV
```

This indicates that the KeyDB process crashed because of a segmentation fault.

### Root Cause

The crash occurred because:

- The **RediSearch module (`redisearch.so`)** was loading during startup.
- Existing RDB/AOF persistence files were created while the module was enabled.
- KeyDB was configured with multiple worker threads (`server-threads > 1`).
- During startup, RediSearch attempted to rebuild indexes from the persistence files, which resulted in a segmentation fault.

---

## Issue 2: Node.js Connection Refused

### Symptom

The application displayed:

```
Error: connect ECONNREFUSED 10.0.1.78:6400
```

### Root Cause

The application configuration did not match the KeyDB server configuration.

| Application | KeyDB |
|-------------|--------|
| Port 6400 | Port 6378 |

Since KeyDB was not listening on port **6400**, every connection request was refused.

Additionally, KeyDB authentication was enabled using **requirepass**, so unauthenticated requests returned:

```
NOAUTH Authentication required.
```

---

# 🚀 Resolution Steps

---

# Step 1 – Disable the RediSearch Module

### Command

```bash
sudo sed -i 's/^loadmodule /# loadmodule /g' /etc/keydb/keydb.conf
```

### Explanation

This command edits the KeyDB configuration file.

It comments every line beginning with:

```
loadmodule
```

For example,

Before:

```
loadmodule /usr/lib/keydb/modules/redisearch.so
```

After:

```
# loadmodule /usr/lib/keydb/modules/redisearch.so
```

### Why?

Since RediSearch was causing the crash during startup and was not required for this deployment, disabling it allowed KeyDB to start without loading the problematic module.

---

# Step 2 – Configure KeyDB to Use a Single Thread

### Command

```bash
sudo sed -i 's/^server-threads .*/server-threads 1/g' /etc/keydb/keydb.conf
```

### Explanation

This command updates the KeyDB configuration.

Before:

```
server-threads 4
```

After:

```
server-threads 1
```

### Why?

The crash occurred while running multiple server threads.

Running with a single thread avoids thread synchronization issues during database loading.

---

# Step 3 – Stop the KeyDB Service

### Command

```bash
sudo systemctl stop keydb-server
```

### Why?

The service must be stopped before modifying or removing database files.

---

# Step 4 – Create a Backup

### Command

```bash
sudo mkdir -p /var/lib/keydb/backup_uat_$(date +%Y%m%d)
```

### Explanation

Creates a backup directory with today's date.

Example:

```
/var/lib/keydb/backup_uat_20260723
```

### Why?

Always create a backup before deleting database files.

---

# Step 5 – Backup Existing Database Files

### Command

```bash
sudo mv /var/lib/keydb/*.rdb /var/lib/keydb/*.aof /var/lib/keydb/backup_uat_$(date +%Y%m%d)/ 2>/dev/null
```

### Explanation

Moves:

- dump.rdb
- appendonly.aof

to the backup folder.

```
2>/dev/null
```

suppresses errors if no files exist.

### Why?

Old persistence files were incompatible after disabling RediSearch.

Removing them forces KeyDB to start with a clean database.

---

# Step 6 – Fix Ownership

### Command

```bash
sudo chown -R keydb:keydb /var/lib/keydb/
```

### Explanation

Changes ownership of the KeyDB data directory.

Owner:

```
keydb
```

Group:

```
keydb
```

### Why?

Without correct permissions, KeyDB cannot read or write its database files.

---

# Step 7 – Reset Failed State

### Command

```bash
sudo systemctl reset-failed keydb-server
```

### Explanation

Clears the failed state stored by systemd.

### Why?

If a service repeatedly fails, systemd remembers the failure.

Resetting clears that history.

---

# Step 8 – Start the Service

### Command

```bash
sudo systemctl start keydb-server
```

### Why?

Starts KeyDB using the updated configuration.

---

# Step 9 – Verify Service Status

### Command

```bash
sudo systemctl status keydb-server
```

Expected output:

```
Active: active (running)
```

This confirms that the service is running successfully.

---

# Step 10 – Test KeyDB Connectivity

### Command

```bash
keydb-cli -p 6378 -a "x-x-e" ping
```

### Explanation

- `-p 6378` → Connect to port 6378
- `-a` → Authenticate using the configured password
- `ping` → Sends a health check command

Expected Output:

```
PONG
```

### Why?

A `PONG` response confirms that:

- KeyDB is running.
- Authentication is working.
- The server is accepting client connections.

---

# Step 11 – Update the Node.js Application

Update the application `.env` file.

```env
REDIS_HOST=10.0.1.78
REDIS_PORT=6378
REDIS_PASSWORD=x-x-e
```

### Why?

The application was previously trying to connect to:

```
10.0.1.78:6400
```

After updating the configuration, it connects to the correct KeyDB port:

```
10.0.1.78:6378
```

---

# ✅ Final Verification

The following checks were successfully completed:

- KeyDB service started successfully.
- RediSearch module disabled.
- Corrupted persistence files backed up.
- Database ownership corrected.
- KeyDB responding with `PONG`.
- Node.js application connected successfully.
- Redis host, port, and password verified.

---

# 📌 Final Outcome

The issue was caused by two independent problems:

1. The **RediSearch module** crashed while loading existing persistence files in multi-threaded mode.
2. The **Node.js application** was configured with an incorrect KeyDB port.

After disabling the unnecessary module, backing up corrupted database files, restarting KeyDB, and updating the application's Redis configuration, the service became stable and the application connected successfully without any further errors.
