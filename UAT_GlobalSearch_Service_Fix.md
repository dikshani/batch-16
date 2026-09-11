# UAT GlobalSearch API — Service Restart & Fix Documentation

## Issue Summary
`uat-globalsearch-api.service` was not running correctly (suspected wrong `prod` Redis config causing failure). Service was reset and restarted, and Redis connections were verified as healthy post-restart.

---

## Environment Details

| Field | Value |
|---|---|
| Server | `ip-10-0-0-83` (user: `ubuntu`) |
| Working Directory | `~/project-cash/src/api/search` |
| Service Name | `uat-globalsearch-api.service` |
| Service Description | Global Search |
| Node Path | `/home/ubuntu/.nvm/versions/node/v20.2.0/bin/node` |
| Entry File | `app.js` |
| App Port | `8011` |
| Systemd Unit Path | `/lib/systemd/system/uat-globalsearch-api.service` |

---

## Commands Executed (on `ip-10-0-0-83`)

### 1. Reset failed state of the service
```bash
sudo systemctl reset-failed uat-globalsearch-api.service
```
Clears any "failed" status flags from systemd so the service can be cleanly restarted.

### 2. Restart the service
```bash
sudo systemctl restart uat-globalsearch-api.service
```

### 3. Wait for service to stabilize
```bash
sleep 2
```

### 4. Check service status
```bash
sudo systemctl status uat-globalsearch-api.service --no-pager
```

**Output (confirmed healthy):**
```
● uat-globalsearch-api.service - Global Search
     Loaded: loaded (/lib/systemd/system/uat-globalsearch-api.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2026-09-11 12:18:21 IST; 2s ago
   Main PID: 3506147 (node)
      Tasks: 12 (limit: 18755)
     Memory: 84.1M (high: 2.0G max: 2.0G available: 1.9G)
        CPU: 784ms
     CGroup: /system.slice/uat-globalsearch-api.service
             └─3506147 /home/ubuntu/.nvm/versions/node/v20.2.0/bin/node app.js

Sep 11 12:18:21 ip-10-0-0-83 systemd[1]: Started Global Search.
Sep 11 12:18:21 ip-10-0-0-83 uat-globalsearch[3506147]: ◇ injected env (1) from .env
Sep 11 12:18:22 ip-10-0-0-83 uat-globalsearch[3506147]: ◇ injected env (0) from .env.local, .env
Sep 11 12:18:22 ip-10-0-0-83 uat-globalsearch[3506147]: 10.0.1.254, New Redis connection
Sep 11 12:18:22 ip-10-0-0-83 uat-globalsearch[3506147]: 10.0.1.182, New Redis connection
Sep 11 12:18:22 ip-10-0-0-83 uat-globalsearch[3506147]: 10.0.1.223, New Redis connection
Sep 11 12:18:22 ip-10-0-0-83 uat-globalsearch[3506147]: 10.0.1.254, Redis already connected
Sep 11 12:18:22 ip-10-0-0-83 uat-globalsearch[3506147]: 10.0.2.245, New Redis connection
Sep 11 12:18:22 ip-10-0-0-83 uat-globalsearch[3506147]: Server has been started at 8011
```

**Redis Connections Confirmed:**
| Redis Node IP | Status |
|---|---|
| 10.0.1.254 | Connected |
| 10.0.1.182 | Connected |
| 10.0.1.223 | Connected |
| 10.0.2.245 | Connected |

Conclusion: wrong `prod` Redis config issue resolved; service is active and listening on port `8011`.

---

## Next Step — Local API Test (on same server `ip-10-0-0-83`)

```bash
curl -v --max-time 10 "http://127.0.0.1:8011/globalsearch/v2/get?searchkey=sbi&page=0&size=15&type=0"
```

### Result Interpretation Guide

| Response | Meaning | Next Action |
|---|---|---|
| **200** | Backend completely healthy | Proceed to test public Nginx URL |
| **401 / 403 / 400** | Backend reachable; response is application/auth related | Not a 504 issue — no further backend action needed |
| **Timeout** | Backend/Redis request-level problem | Investigate Redis/backend internals |
| **Connection refused** | Service crashed again | Re-check service status, logs, and restart |

**Note:** Nginx configuration was intentionally **not** changed at this stage. Local backend response must be confirmed first before touching Nginx.

---

## Status
- [x] Service reset & restarted
- [x] Redis connections verified
- [ ] Local API (`127.0.0.1:8011`) response pending verification
- [ ] Public Nginx URL test pending
