# 🛠️ KeyDB Incident & Troubleshooting Report

**Date:** July 23, 2026  
**Server IP:** `10.0.1.78`  
**Status:** Resolved ✅

---

## 📌 Executive Summary

During the deployment/testing phase, the Node.js application failed to connect to the KeyDB server, resulting in connection errors:
1. **KeyDB Crash (`Segmentation Fault`):** KeyDB failed to start due to corruption/conflict between the `RediSearch` module, persistence files (`.rdb`/`.aof`), and multi-threading.
2. **Connection Refused (`ECONNREFUSED`):** Node.js was attempting to connect to **Port 6400**, whereas KeyDB was configured and listening on **Port 6378**.

---

## 🔍 Root Cause Analysis

### 1. KeyDB Crash (`Signal 11 / Segmentation Fault`)
* **Symptom:** Executing `keydb-server` crashed immediately with exit code failure (`code=dumped, signal=SEGV`).
* **Root Cause:** The `redisearch.so` module crashed during index initialization when starting with multi-threaded options (`server-threads > 1`) on pre-existing data files.

### 2. Node.js `ECONNREFUSED 10.0.1.78:6400`
* **Symptom:** Node.js application threw `Error: connect ECONNREFUSED 10.0.1.78:6400`.
* **Root Cause:** 
  * Port mismatch: Application requested port `6400`, but KeyDB was running on port `6378`.
  * Authentication: KeyDB required a password (`requirepass`), which caused `NOAUTH` errors when unauthenticated connections were attempted.

---

## 🚀 Step-by-Step Resolution

### Step 1: Disable Unused Modules & Set Single Threading
To prevent the `RediSearch` module crash, we disabled the external module and restricted threading to `1`.

```bash
# Comment out external modules loading (if not required)
sudo sed -i 's/^loadmodule /# loadmodule /g' /etc/keydb/keydb.conf

# Set server-threads to 1 to avoid multi-threading conflicts
sudo sed -i 's/^server-threads .*/server-threads 1/g' /etc/keydb/keydb.conf
```

---

### Step 2: Backup and Clear Corrupted Persistence Files
KeyDB failed to start because it attempted to load corrupted RDB/AOF files created before module removal.

```bash
# 1. Stop KeyDB service
sudo systemctl stop keydb-server

# 2. Create a backup directory
sudo mkdir -p /var/lib/keydb/backup_uat_$(date +%Y%m%d)

# 3. Move old .rdb and .aof files to the backup folder
sudo mv /var/lib/keydb/*.rdb /var/lib/keydb/*.aof /var/lib/keydb/backup_uat_$(date +%Y%m%d)/ 2>/dev/null

# 4. Fix folder permissions
sudo chown -R keydb:keydb /var/lib/keydb/
```

---

### Step 3: Restart KeyDB Service
After clearing corrupted state files and fixing configuration parameters:

```bash
# Reset systemd failure flags
sudo systemctl reset-failed keydb-server

# Start KeyDB service
sudo systemctl start keydb-server

# Check service status
sudo systemctl status keydb-server
```
> **Result:** `Active: active (running)`

---

### Step 4: Verification & Authentication Test
KeyDB was configured with a password (`requirepass x-x-e`). We verified connection on port `6378`:

```bash
keydb-cli -p 6378 -a "x-x-e" ping
```
> **Output:** `PONG`

---

## ⚙️ Final Application Configuration

Update your Node.js application environment variables (`.env`) with the following values:

```env
# KeyDB / Redis Connection Details
REDIS_HOST=10.0.1.78
REDIS_PORT=6378
REDIS_PASSWORD=x-x-e
```

---

## 📋 Summary Checklist

- [x] Unused modules disabled and single-threaded mode configured in `keydb.conf`.
- [x] Legacy/corrupted `.rdb` and `.aof` files backed up and removed.
- [x] KeyDB service started successfully on port `6378`.
- [x] Authentication verified via `keydb-cli` returning `PONG`.
- [x] Node.js `.env` configuration aligned with host, port, and password.
