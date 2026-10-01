# MF Watchlist API – 502 Bad Gateway
## RCA, Resolution, Fix & Prevention Documentation

**Environment:** UAT  
**Module:** Mutual Fund (MF)  
**API:** `POST /mf/watchlist/v1/getwatchlist`  
**Nginx Host:** `INVEST-UAT-NGINX`  
**Backend Host:** `10.0.3.213`  
**Backend Port:** `9001`  
**Date:** 01-Oct-2026

---

## 1. Incident Summary

The MF Watchlist API was returning:

```text
502 Bad Gateway
```

for:

```text
POST https://invest.venturasecurities.uat/mf/watchlist/v1/getwatchlist
```

Nginx was forwarding the request to the MF Transaction API backend:

```text
10.0.3.213:9001
```

The backend service was unavailable because the Gunicorn MF transaction service failed during worker startup.

---

## 2. Architecture / Request Flow

```text
Client
  |
  v
UAT Nginx
  |
  | /mf/watchlist/v1/getwatchlist
  v
mf-txn-apis
  |
  v
10.0.3.213:9001
  |
  v
Gunicorn
  |
  v
src.ec2.api.txn_server:api
```

Nginx upstream configuration:

```nginx
upstream mf-txn-apis {
    server 10.0.3.213:9001;
}
```

The Watchlist URL did not have a dedicated Nginx location and therefore used the generic proxy route to `mf-txn-apis`.

---

## 3. Initial Symptoms

The browser/API request returned:

```text
Status Code: 502 Bad Gateway
Remote Address: 10.0.0.149:443
```

Direct connectivity testing from the Nginx container to the backend returned:

```text
connect to 10.0.3.213 port 9001 failed: Connection refused
curl: (7) Failed to connect
```

This established that the issue was between Nginx and the backend service, rather than an Nginx configuration syntax issue.

---

## 4. Investigation

### 4.1 Nginx Validation

The Nginx configuration was tested inside the Docker container:

```bash
docker exec nginx nginx -t
```

Result:

```text
syntax is ok
test is successful
```

Therefore, no Nginx syntax/configuration error was identified.

### 4.2 Backend Port Check

On `10.0.3.213`:

```bash
sudo ss -tunlp | grep 9001
```

Initially there was no listener on port `9001`.

### 4.3 Service Identification

The configured systemd service was:

```text
gunicorn_mf_fastapi_gunicorn_txn.service
```

Its `ExecStart` was:

```bash
/home/ubuntu/.local/bin/gunicorn src.ec2.api.txn_server:api     --workers 4     --worker-class uvicorn.workers.UvicornWorker     --bind 0.0.0.0:9001     --timeout 120     --keep-alive 5
```

### 4.4 Gunicorn Failure

The service repeatedly failed with worker boot errors.

The key error was:

```text
ModuleNotFoundError: No module named 'src.ec2.api'
```

The Gunicorn service was therefore unable to load:

```text
src.ec2.api.txn_server:api
```

and exited with status `3`.

---

## 5. Root Cause

### Primary Root Cause

The deployed MF application workspace was incomplete and did not contain:

```text
src/ec2/api/
```

As a result, Gunicorn could not import:

```text
src.ec2.api.txn_server
```

The backend service on port `9001` therefore failed to start.

Because Nginx could not connect to port `9001`, it returned:

```text
502 Bad Gateway
```

### Secondary / Contributing Root Cause

The GitHub Actions UAT checkout was also failing because the existing workspace contained files with incorrect ownership/permissions.

The checkout action reported:

```text
Error: File was unable to be removed
EACCES: permission denied, unlink
.../src/ec2/jobs/send_camspay_nortified_data_to_admin/__pycache__/app_config.cpython-312.pyc
```

Because `actions/checkout@v4` could not clean the workspace, the repository checkout did not complete correctly.

This left the deployment workspace in an incomplete state.

---

## 6. Why the Deployment Checkout Failed

The UAT workflow uses:

```text
actions/checkout@v4
```

The workflow attempted to clean:

```text
/home/ubuntu/actions-runner/_work/proj_mf/proj_mf
```

before checking out the repository.

A file under `__pycache__` could not be deleted because of a permission/ownership mismatch.

Therefore:

```text
Incorrect ownership
        |
        v
Checkout cannot clean workspace
        |
        v
GitCheckout fails
        |
        v
Incomplete application source
        |
        v
src.ec2.api missing
        |
        v
Gunicorn worker boot failure
        |
        v
Port 9001 unavailable
        |
        v
Nginx 502
```

---

## 7. Resolution / Fix Performed

### Step 1 – Correct workspace ownership

The application workspace ownership was corrected:

```bash
sudo chown -R ubuntu:ubuntu /home/ubuntu/actions-runner/_work/proj_mf/proj_mf
```

### Step 2 – Remove problematic Python cache files

Problematic `__pycache__` directories were removed:

```bash
sudo find /home/ubuntu/actions-runner/_work/proj_mf/proj_mf -type d -name "__pycache__" -prune -exec rm -rf {} +
```

### Step 3 – Re-run UAT deployment

The UAT workflow:

```text
UAT-MF-Deployment(MF-QA)
```

was re-run.

After the workspace permission issue was resolved, the checkout completed successfully and the MF source was restored.

The following directory became available:

```text
/home/ubuntu/actions-runner/_work/proj_mf/proj_mf/src/ec2/api
```

including:

```text
txn_server.py
watchlist/
watchlist_v2/
```

### Step 4 – Verify backend service

After deployment, port `9001` was confirmed listening:

```bash
sudo ss -lntp | grep 9001
```

Result:

```text
LISTEN 0 2048 0.0.0.0:9001 ...
```

Gunicorn processes were running and listening on port `9001`.

---

## 8. Current Status

### Backend

```text
Gunicorn: RUNNING
Port 9001: LISTENING
txn_server.py: PRESENT
Watchlist module: PRESENT
```

### Nginx

Nginx configuration was already valid.

The original 502 was caused by backend unavailability, not by an Nginx syntax/configuration issue.

---

## 9. Permanent Prevention

### 9.1 Do not routinely fix ownership manually

`chown -R ubuntu:ubuntu` should **not** be required before every deployment.

If the issue repeats, identify which process/job is creating files with another user.

Check:

```bash
ps -eo user,pid,cmd | grep -E 'gunicorn|python' | grep -v grep
```

Check ownership:

```bash
find /home/ubuntu/actions-runner/_work/proj_mf/proj_mf -not -user ubuntu -ls | head -50
```

### 9.2 Keep application services running as `ubuntu`

The MF Gunicorn service should continue using:

```ini
User=ubuntu
Group=ubuntu
```

This prevents the application from creating root-owned runtime/cache files.

### 9.3 Avoid running application commands as root

Avoid commands such as:

```bash
sudo python ...
sudo pip ...
sudo gunicorn ...
```

inside the application workspace unless specifically required.

If a deployment step needs elevated privileges, use `sudo` only for the required system operation rather than running the application process as root.

### 9.4 Review deployment workflow

The UAT workflow should ensure:

1. Checkout runs successfully.
2. Application files remain owned by `ubuntu`.
3. Python cache files are not created by root.
4. Service restart uses the systemd unit configured with `User=ubuntu`.
5. Deployment validates port `9001` after restart.

Recommended post-deployment validation:

```bash
sudo systemctl is-active gunicorn_mf_fastapi_gunicorn_txn.service
sudo ss -lntp | grep 9001
```

### 9.5 Add deployment health check

A future deployment can include a simple validation after restarting the transaction service:

```bash
sudo systemctl is-active --quiet gunicorn_mf_fastapi_gunicorn_txn.service
sudo ss -lntp | grep -q ':9001'
```

If either check fails, the deployment should fail immediately instead of leaving the service unavailable.

---

## 10. Recommended Monitoring

Monitor:

```text
MF Gunicorn service
Port 9001
Nginx 502 responses
GitHub Actions GitCheckout failures
Workspace ownership
```

Useful commands:

```bash
sudo systemctl status gunicorn_mf_fastapi_gunicorn_txn.service --no-pager
```

```bash
sudo journalctl -u gunicorn_mf_fastapi_gunicorn_txn.service -n 100 --no-pager
```

```bash
sudo ss -lntp | grep 9001
```

```bash
find /home/ubuntu/actions-runner/_work/proj_mf/proj_mf -not -user ubuntu -ls | head -50
```

---

## 11. RCA – Short Version for Jira / Incident Report

> **RCA:** MF Watchlist API was returning 502 because the MF Transaction backend service on port 9001 was unavailable. Gunicorn failed to boot because the required `src.ec2.api` module was missing from the deployment workspace. The UAT GitHub Actions checkout was unable to clean the workspace due to incorrect ownership/permissions on a Python `__pycache__` file, resulting in an incomplete repository checkout. Workspace ownership and problematic cache files were corrected, the UAT deployment was re-run successfully, and the MF Gunicorn service was restored on port 9001.

---

## 12. Incident Timeline

| Time / Stage | Activity |
|---|---|
| Initial | MF Watchlist API returned 502 |
| Investigation | Nginx configuration validated |
| Investigation | Backend `10.0.3.213:9001` found unavailable |
| Investigation | Gunicorn service found failed |
| Root cause identified | `src.ec2.api` missing / `ModuleNotFoundError` |
| Deployment investigation | UAT GitHub Actions checkout found failing |
| Checkout error | `EACCES` while deleting `__pycache__` file |
| Fix | Workspace ownership corrected |
| Fix | Python `__pycache__` directories removed |
| Fix | UAT deployment re-run |
| Verification | `src/ec2/api/txn_server.py` restored |
| Verification | Port `9001` confirmed LISTENING |
| Status | Backend restored |

---

## 13. Final Technical Conclusion

The incident was **not primarily an Nginx configuration issue**.

The failure chain was:

```text
Workspace permission issue
        ↓
GitHub Actions checkout failure
        ↓
Incomplete MF source deployment
        ↓
src.ec2.api missing
        ↓
Gunicorn worker failed to boot
        ↓
Port 9001 unavailable
        ↓
Nginx connection refused
        ↓
502 Bad Gateway
```

The immediate service issue was resolved by restoring the correct application source and restarting/deploying the MF transaction service.

The permanent preventive action is to identify and eliminate the process/deployment step that creates root-owned files in the GitHub Actions workspace.
