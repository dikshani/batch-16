# Troubleshooting 502 Bad Gateway for Stock SIP API (UAT)

## Overview

This document explains the complete troubleshooting process followed to resolve the **502 Bad Gateway** issue for the Stock SIP API in the UAT environment.

**API URL**

```text
https://invest.venturasecurities.uat/stocks/sip/v1/list?page=1
```

---

## Problem Statement

While accessing the above API, the application returned:

```text
502 Bad Gateway
```

A **502 Bad Gateway** indicates that:

- Nginx is running successfully.
- The client request reached Nginx.
- Nginx was unable to receive a valid response from the backend application.

---

# Troubleshooting Steps

## Step 1: Check Nginx Error Log

### Command

```bash
sudo tail -100 /var/log/nginx/invest.venturasecurities.uat.error.log
```

### Purpose

Checks whether the virtual host has a dedicated error log containing backend errors.

### Result

```text
No such file or directory
```

Since the log file was unavailable, the default Nginx error log was checked.

---

## Step 2: Check Default Nginx Error Log

### Command

```bash
sudo tail -100 /var/log/nginx/error.log
```

### Purpose

Checks the default Nginx error log for proxy or backend-related issues.

### Result

No useful backend errors were found.

---

## Step 3: Validate Nginx Configuration

### Command

```bash
sudo nginx -T | grep error_log
```

### Result

```text
server directive is not allowed here
```

### Purpose

Verifies whether the Nginx configuration contains syntax errors.

### Observation

The error belonged to another configuration (`charts.venturasecurities.uat.conf`) and was unrelated to the SIP API.

---

## Step 4: Locate the SIP API Configuration

### Command

```bash
grep -R "stocks/sip" /etc/nginx/
```

### Result

```text
/etc/nginx/conf.d/cash_node_api_v5.conf
```

### Purpose

Finds which Nginx configuration handles the `/stocks/sip` endpoint.

---

## Step 5: Verify Proxy Configuration

### Command

```bash
sudo grep -A15 -B5 "location ~ ^/stocks/sip" /etc/nginx/conf.d/cash_node_api_v5.conf
```

### Result

```nginx
proxy_pass http://10.0.1.223:2012/sip/$1$is_args$args;
```

### Purpose

Verifies the backend server and port to which Nginx forwards requests.

---

## Step 6: Test Backend Connectivity

### Command

```bash
curl -v http://10.0.1.223:2012/sip/v1/list?page=1
```

### Result

```text
Connection refused
```

### Purpose

Checks whether the backend API is running.

### Observation

No application was listening on port **2012**.

---

## Step 7: Verify SIP Services

### Command

```bash
systemctl --type=service | grep -i sip
```

### Result

```text
stock_sip_prepare_json.service
stock_sip_txn.service
```

### Purpose

Checks whether SIP-related services are active.

---

## Step 8: Verify Listening Ports

### Command

```bash
sudo ss -lntp | grep LISTEN
```

### Purpose

Lists all listening TCP ports.

### Observation

Port **2012** was missing.

This confirmed that no backend process was serving requests.

---

## Step 9: Inspect Service Configuration

### Command

```bash
sudo systemctl cat stock_sip_txn.service
```

### Purpose

Checks which binary the service starts.

### Observation

The service starts:

```text
sip_txn
```

---

## Step 10: Verify Service Status

### Command

```bash
sudo systemctl status stock_sip_txn.service
```

### Result

```text
Active: running
```

### Purpose

Confirms that the transaction service is running.

---

## Step 11: Review Configuration File

### Command

```bash
cat /home/ubuntu/uat/project-cash/src/transaction/sip/txn/conf/txn-nonprod.ini
```

### Observation

```ini
PORT=5454
```

### Purpose

Verified that this port belongs to the database configuration and is **not** the API port.

---

## Step 12: Check Process Network Connections

### Command

```bash
sudo lsof -Pan -p <PID> -i
```

### Observation

The process had only a database connection.

No HTTP listener was found.

---

## Step 13: Locate SIP API Binary

### Command

```bash
ls -la /home/ubuntu/uat/project-cash/src/transaction/sip/apis
```

### Observation

Found the API executable:

```text
build/sip
```

---

## Step 14: Verify API Process

### Command

```bash
ps -ef | grep "/home/ubuntu/uat/project-cash/src/transaction/sip/apis/build/sip"
```

### Result

No running process found.

### Purpose

Confirms that the API server is not running.

---

## Step 15: Start SIP API

### Command

```bash
cd /home/ubuntu/uat/project-cash/src/transaction/sip/apis

./build/sip config.json
```

### Result

```text
Add listener: 0.0.0.0:2012
```

### Purpose

Starts the SIP API server.

---

## Step 16: Verify Port 2012

### Command

```bash
ss -lntp | grep 2012
```

### Result

```text
LISTEN 0.0.0.0:2012
```

### Purpose

Confirms that the API is listening on the expected port.

---

## Step 17: Test API Locally

### Command

```bash
curl -v http://127.0.0.1:2012/sip/v1/list?page=1
```

### Response

```json
{
  "message":"invalid/missing x-client-id in header",
  "status":"error"
}
```

### Purpose

Confirms that:

- The backend API is running.
- Port 2012 is reachable.
- Nginx can communicate with the backend.

The remaining issue is an application-level validation requiring the `x-client-id` request header.

---

## Step 18: Verify Source Code

### Command

```bash
grep -R "invalid/missing x-client-id" /home/ubuntu/uat/project-cash/src/transaction/sip/apis
```

### Purpose

Locates where the application validates the request header.

### Observation

Multiple controllers require the `x-client-id` header before processing requests.

---

# Root Cause

The **502 Bad Gateway** occurred because:

- Nginx was configured to proxy requests to **10.0.1.223:2012**.
- No backend application was listening on port **2012**.
- The SIP API executable (`build/sip`) was not running.

---

# Resolution

Started the SIP API manually:

```bash
cd /home/ubuntu/uat/project-cash/src/transaction/sip/apis

./build/sip config.json
```

After starting the application:

- Port **2012** became active.
- Nginx successfully connected to the backend.
- The **502 Bad Gateway** error was resolved.

The API then returned:

```json
{
  "message":"invalid/missing x-client-id in header"
}
```

which confirms that the backend is operational and only requires the expected request header.

---

# Final Status

| Component | Status |
|-----------|--------|
| Nginx | ✅ Running |
| Proxy Configuration | ✅ Correct |
| SIP Transaction Service | ✅ Running |
| SIP API | ✅ Running |
| Port 2012 | ✅ Listening |
| 502 Bad Gateway | ✅ Resolved |
| Current Issue | Missing `x-client-id` header |








---

## Permanent Fix - Create Systemd Service for SIP API

### Problem

The SIP API (`build/sip`) was not configured as a systemd service.

As a result:

- The EC2 server was stopped every night and started every morning using a scheduler.
- After server startup, the SIP API process did not start automatically.
- Nginx attempted to forward requests to `10.0.1.223:2012`, but no process was listening on port **2012**.
- This resulted in a **502 Bad Gateway** error.

Example:

```
https://invest.venturasecurities.uat/stocks/sip/v1/list?page=1
```

---

## Verify Existing Process

Before creating the service, check whether the SIP API is already running.

```bash
ps -ef | grep "/home/ubuntu/uat/project-cash/src/transaction/sip/apis/build/sip"
```

Check whether port **2012** is listening.

```bash
ss -lntp | grep 2012
```

If the API is already running manually, stop it before starting the systemd service.

```bash
pkill -f "/home/ubuntu/uat/project-cash/src/transaction/sip/apis/build/sip"
```

---

## Create Systemd Service

Create a new service file.

```bash
sudo nano /etc/systemd/system/stock_sip_api.service
```

Paste the following configuration.

```ini
[Unit]
Description=Stock SIP API Service
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/uat/project-cash/src/transaction/sip/apis
ExecStart=/home/ubuntu/uat/project-cash/src/transaction/sip/apis/build/sip /home/ubuntu/uat/project-cash/src/transaction/sip/apis/config.json
Restart=always
RestartSec=5

StandardOutput=append:/var/log/stock_sip_api.log
StandardError=append:/var/log/stock_sip_api.log

[Install]
WantedBy=multi-user.target
```

---

## Reload Systemd

Reload the systemd daemon to recognize the new service.

```bash
sudo systemctl daemon-reload
```

---

## Enable Service

Enable the service so it starts automatically whenever the server boots.

```bash
sudo systemctl enable stock_sip_api.service
```

---

## Start Service

```bash
sudo systemctl start stock_sip_api.service
```

---

## Verify Service Status

```bash
sudo systemctl status stock_sip_api.service
```

Expected output:

```
Active: active (running)
```

---

## Verify Listening Port

Confirm that the SIP API is listening on port **2012**.

```bash
ss -lntp | grep 2012
```

Expected:

```
LISTEN 0 4096 0.0.0.0:2012
```

---

## Test the API

Verify that the API is responding.

```bash
curl -H "x-client-id: test" "http://127.0.0.1:2012/sip/v1/list?page=1"
```

If the API responds, Nginx will also be able to forward requests successfully.

---

## Monitor Logs

Check service logs.

```bash
sudo journalctl -u stock_sip_api.service -f
```

Or check the application log.

```bash
tail -f /var/log/stock_sip_api.log
```

---

## Benefits

- Automatically starts after every server reboot.
- No manual execution of `./build/sip config.json` is required.
- Automatically restarts if the application crashes.
- Prevents recurring **502 Bad Gateway** errors caused by the SIP API not running.
- Suitable for servers that are stopped and started daily using AWS Scheduler.

