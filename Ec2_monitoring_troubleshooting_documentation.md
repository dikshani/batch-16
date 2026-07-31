# EC2 Server Monitoring — Setup & Troubleshooting Documentation
 
This document records the full process followed to (1) check scheduled jobs on EC2 servers via AWS Systems Manager, (2) locate EC2 auto start/stop schedules, and (3) fix missing servers in the Grafana/Prometheus monitoring dashboard by installing Node Exporter.
 
---
 
## Part 1: Checking Cron Jobs on Multiple EC2 Servers via SSM Run Command
 
**Goal:** View scheduled cron jobs across many EC2 instances without SSH-ing into each one individually.
 
### Steps
 
1. Go to **AWS Console → Systems Manager → Run Command → Run command**.
2. Under **Command document**, search for and select:
   - `AWS-RunShellScript` (for Linux instances)
   - `AWS-RunPowerShellScript` (for Windows instances)
3. In the **Command parameters** box, enter:
```bash
   echo "=== User crontab ==="
   crontab -l 2>/dev/null || echo "No user crontab"
   echo "=== System crontab ==="
   cat /etc/crontab
   echo "=== Cron.d ==="
   ls -la /etc/cron.d/ 2>/dev/null
```
4. Under **Target selection**, choose one of:
   - **Specify instance tags** — target instances that share a tag
   - **Choose instances manually** — manually tick the instances to run the command on
5. Under **Rate control**:
   - **Concurrency (targets):** set to the number of instances you selected (e.g. 10)
   - **Error threshold (error):** set to the same number, so that one failing instance does **not** cancel the rest of the batch
6. Click **Run**.
7. On the results page, open the **Targets and outputs** tab. Click any instance row to see its **Output**, which contains the cron job listing for that server.
### Key Learning — Why some instances showed "Terminated"
When multiple instances are targeted together, AWS Run Command has a default **Error threshold**. If this is set too low (e.g. 1) and even one instance fails, AWS automatically cancels (`Terminated`) execution on the remaining pending instances — even ones that would have succeeded. **Fix:** raise the Concurrency and Error threshold to match (or exceed) the number of targeted instances.
 
### Prerequisite for SSM to work
Only instances that are **SSM-managed** appear in the target list. This requires:
- **SSM Agent** installed and running on the instance
- IAM role attached to the instance with the **AmazonSSMManagedInstanceCore** policy
- Network path to SSM endpoints (internet/NAT or VPC endpoints)
Check managed instance status at: **Systems Manager → Fleet Manager (Managed Instances)**.
 
---
 
## Part 2: Finding EC2 Auto Start/Stop Schedules
 
**Goal:** Find out what time an EC2 instance is automatically started and stopped.
 
### Where this lives
Instance start/stop scheduling in this AWS account is implemented using **Amazon EventBridge Rules**, created by the **AWS QuickSetup Scheduler**.
 
### Steps
 
1. Go to **AWS Console → EventBridge → Rules**.
2. Look for rule names like:
   - `AWSQuickSetup-Scheduler-StartEC2Rule-xxxxx`
   - `AWSQuickSetup-Scheduler-StopEC2Rule-xxxxx`
   (The matching suffix code, e.g. `2p64k`, pairs a Start rule with its corresponding Stop rule for the same instance/group.)
3. Click a rule to open **Rule details**.
4. In this account, the rules did **not** contain a direct cron expression. Instead, the **Event pattern** referenced an **SSM Change Calendar**:
```json
   {
     "detail-type": ["Calendar State Change"],
     "resources": ["arn:aws:ssm:ap-south-1:<account-id>:calendar/..."],
     "source": ["aws.ssm"],
     "detail": { "state": ["OPEN"] }
   }
```
5. To find the actual start/stop **times**, go to **AWS Console → Systems Manager → Change Calendar**, open the matching calendar, and check its defined **events** (these contain the actual daily/weekly open and close times).
---
 
## Part 3: Fixing Missing Servers in Grafana (Prometheus Monitoring)
 
**Problem:** Some EC2 servers were not showing up in the Grafana dashboard (data source: Prometheus + Node Exporter).
 
### Step 1 — Access the Prometheus server
Since the Prometheus server was on a **private IP** (not internet-reachable), it required company **VPN** access to open in a browser:
```
http://<prometheus-private-ip>:9090
```
 
### Step 2 — Check scrape targets
1. In the Prometheus UI, go to **Status → Targets**.
2. This showed a scrape pool (e.g. `NonProduction-Metrics`) with a count like `23 / 28 up` — meaning 5 targets were failing.
3. Used the **"Filter by target health" → Down** filter to isolate the failing targets and read their error messages.
### Step 3 — Diagnose each failure type
 
| Error message | Meaning | Fix |
|---|---|---|
| `context deadline exceeded` + `state="stopped"` | The EC2 instance itself is stopped | Start the instance from the EC2 console |
| `dial tcp ... connect: connection refused` | Instance is running, but nothing is listening on port 9100 (Node Exporter is not running/installed) | Install/start Node Exporter (see Step 4 below) |
| `context deadline exceeded` (instance running) | Request is timing out — usually a Security Group / firewall issue | Allow inbound TCP port 9100 from the Prometheus server's IP in the instance's Security Group |
 
### Step 4 — Install Node Exporter (for "connection refused" cases)
 
Commands run on the affected server (via SSH):
 
1. **Check if Node Exporter is already running:**
```bash
   ps aux | grep node_exporter
```
   (Empty result = not running.)
 
2. **Download Node Exporter:**
```bash
   cd /tmp
   wget https://github.com/prometheus/node_exporter/releases/download/v1.8.0/node_exporter-1.8.0.linux-amd64.tar.gz
```
 
3. **Extract the archive:**
```bash
   tar xvfz node_exporter-1.8.0.linux-amd64.tar.gz
```
 
4. **Move the binary into the system path:**
```bash
   sudo mv node_exporter-1.8.0.linux-amd64/node_exporter /usr/local/bin/
```
 
5. **Create a dedicated system user (security best practice):**
```bash
   sudo useradd --no-create-home --shell /usr/sbin/nologin node_exporter
   sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```
 
6. **Create a systemd service file:**
```bash
   sudo nano /etc/systemd/system/node_exporter.service
```
   Contents:
```ini
   [Unit]
   Description=Node Exporter
   After=network.target
 
   [Service]
   User=node_exporter
   Group=node_exporter
   Type=simple
   ExecStart=/usr/local/bin/node_exporter
 
   [Install]
   WantedBy=multi-user.target
```
 
7. **Reload systemd and start the service:**
```bash
   sudo systemctl daemon-reload
   sudo systemctl enable node_exporter
   sudo systemctl start node_exporter
```
 
8. **Verify status:**
```bash
   sudo systemctl status node_exporter
```
 
### Issue encountered — "Exec format error"
First attempt failed with:
```
Failed to execute /usr/local/bin/node_exporter: Exec format error
```
**Cause:** The downloaded binary architecture did not match the server's CPU architecture (e.g. amd64 binary on an ARM/Graviton server, or a corrupted download).
 
**Diagnosis command:**
```bash
uname -m
```
- `x86_64` → use the `linux-amd64` release
- `aarch64` → use the `linux-arm64` release
After re-downloading the correct binary and restarting the service, status changed to:
```
Active: active (running)
...
msg="Listening on" address=[::]:9100
```
✅ Node Exporter was now running correctly on port 9100.
 
### Step 5 — Confirm the target came back UP
Went back to **Prometheus → Status → Targets** — the previously-down server now showed status **UP**.
 
### Step 6 — Grafana dashboard still showed "No data" / "N/A"
Even though:
- Prometheus showed the target as **UP**
- A direct query in Prometheus (`node_cpu_seconds_total{job="BankFD-Development"}`) returned data successfully
...the Grafana dashboard still showed **No data**, because the dashboard uses **two chained variables** — `InstanceName` and `Node` — and both must match the correct Prometheus labels (`job`/`instance`) for the panel queries to return data. Selecting only `InstanceName` without the matching `Node` value results in an empty query.
 
**Next troubleshooting step (in progress):** Open the panel's **Inspect → Query** view to see the exact label the dashboard filters on (e.g. `job="$job"` or `instance="$instance"`), and match the `Node` dropdown selection to that exact value — or manually test the same query directly in Prometheus with the specific `instance="<ip>:9100"` label to confirm the correct format.
 
---
 
## Quick Reference — Useful Commands
 
```bash
# Check if Node Exporter is running
ps aux | grep node_exporter
 
# Check server CPU architecture
uname -m
 
# Check what's listening on the Node Exporter port
sudo netstat -tulnp | grep 9100
# or
sudo ss -tulnp | grep 9100
 
# Test Node Exporter locally on the server
curl http://localhost:9100/metrics
 
# Node Exporter service management
sudo systemctl status node_exporter
sudo systemctl start node_exporter
sudo systemctl restart node_exporter
sudo systemctl enable node_exporter
```
 
---
 
## Summary of Servers Found DOWN in Prometheus and Their Status
 
| Server | Root Cause | Status |
|---|---|---|
| SSO-LoadTesting | EC2 instance stopped | Pending — needs to be started in EC2 console |
| BankFD-Development | Node Exporter not installed | ✅ Fixed — Node Exporter installed and running |
| (3rd target) | Node Exporter not installed | Pending — repeat Node Exporter install steps |
| UAT-OPENVPN-ACCESS-SERVER-50Users | Timeout — likely Security Group blocking port 9100 | Pending — check/add inbound rule for port 9100 |
| (5th target) | Not yet diagnosed | Pending |
