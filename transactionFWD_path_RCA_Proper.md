# Root Cause Analysis (RCA)
## Transaction Forward Path (`txn`) — Port 9378 Not Listening

**Incident Type:** Application availability / service health issue  
**Application:** `txn` (Transaction Forward Path)  
**Service:** `transactionFWD_path.service`  
**Application Port:** `9378`  
**Affected PID:** `320476`  
**Application Binary:** `/home/ubuntu/uat/project-cash/src/transaction/orders/build/txn`  
**Configuration:** `/home/ubuntu/uat/project-cash/src/transaction/orders/config.json`  
**Application Log:** `/home/ubuntu/uat/project-cash/src/transaction/orders/drogon.log`  
**Lock File:** `/tmp/drogon.lock`

---

## 1. Executive Summary

The `txn` application process was running, but TCP port **9378 was not listening**. As a result, the application appeared healthy from a process-level check while remaining unavailable to downstream consumers.

Investigation showed that PID `320476` was in the kernel wait state:

```text
locks_lock_inode_wait
```

This indicated that the process was blocked while waiting on a file lock.

The affected process was confirmed to belong to:

```text
transactionFWD_path.service
```

using the process cgroup:

```text
cat /proc/320476/cgroup
```

The process was subsequently signalled. Shortly afterward, the same PID recovered, returned to the normal `ep_poll` state, and port `9378` started listening again.

> **Important:** The available evidence confirms the file-lock-related blocking state, but it does **not conclusively identify the underlying trigger** that caused the lock wait.

---

# 2. Issue Summary

### Observed State

| Check | Status |
|---|---|
| `txn` process | Running |
| Process PID | `320476` |
| Process WCHAN | `locks_lock_inode_wait` |
| Port `9378` | Not Listening |
| Service | `transactionFWD_path.service` |
| Lock file | `/tmp/drogon.lock` |

### Impact

The transaction-forwarding application was unavailable because port `9378` was not listening.

The incident demonstrated that **process-level monitoring alone is insufficient** for this application.

A process can remain alive while the application itself is unavailable.

---

# 3. Investigation

## 3.1 Process State Check

The process state and kernel wait channel were checked using:

```bash
sudo ps -o pid,stat,wchan:30,etime,cmd -p 320476
```

Initial state:

```text
PID      STAT   WCHAN
320476   Ssl    locks_lock_inode_wait
```

### Finding

`locks_lock_inode_wait` indicated that the process was blocked in a kernel-level file-lock wait state.

Therefore, although the process was technically alive, it was not operating normally.

---

## 3.2 Port Check

The application port was checked using:

```bash
sudo ss -lntp | grep ':9378'
```

No `LISTEN` entry was returned.

### Finding

Port `9378` was genuinely unavailable. This confirmed that the issue was not merely a misleading process status.

---

## 3.3 Lock Investigation

Locks were checked using:

```bash
sudo lslocks | grep txn
```

Initially, two lock entries were visible:

```text
txn   320476   FLOCK   WRITE*
txn   319203   FLOCK   WRITE*
```

PID `319203` was then checked:

```bash
sudo ps -fp 319203
```

The process was no longer present.

After recovery, the lock check returned no `txn` entries.

### Finding

A file-lock-related condition was present during the incident. However, because PID `319203` had already disappeared when investigated, the available evidence does not prove that it was the conflicting process responsible for the incident.

---

## 3.4 `/tmp/drogon.lock` Investigation

Open files for the affected process were inspected:

```bash
sudo lsof -p 320476
```

The process was found to hold:

```text
/tmp/drogon.lock
```

and was actively writing to:

```text
/home/ubuntu/uat/project-cash/src/transaction/orders/drogon.log
```

The lock file itself was inspected:

```bash
ls -l /tmp/drogon.lock
```

Result:

```text
-rw-r--r-- 1 ubuntu ubuntu 0 Sep 16 10:42 /tmp/drogon.lock
```

### Important Clarification

The lock file was **not deleted** during troubleshooting.

The existence of `/tmp/drogon.lock` alone does not prove that the file itself was the root cause. Deleting it without evidence could have removed useful diagnostic information and potentially changed application behaviour.

---

## 3.5 Configuration File Check

The configuration file was checked using:

```bash
sudo fuser /home/ubuntu/uat/project-cash/src/transaction/orders/config.json
```

No output was returned.

### Finding

`config.json` was not actively held open by another process at the time of investigation.

---

## 3.6 Application Log Review

The application log was reviewed:

```bash
sudo tail -100 /home/ubuntu/uat/project-cash/src/transaction/orders/drogon.log
```

The available log entries showed normal application activity, including token loading and database connectivity.

For example, successful SIP database connectivity was observed.

However, there was no explicit error in the available log window that directly explained the lock wait.

### Finding

The application log was treated as **supporting evidence**, not definitive proof of the root cause.

---

# 4. Correct Service Identification

Two similarly named services were present on the server:

```text
stock_sip_txn.service
transactionFWD_path.service
```

Because service names alone can be misleading, the process-to-service relationship was verified directly.

The cgroup of PID `320476` was checked:

```bash
sudo cat /proc/320476/cgroup
```

Result:

```text
0::/system.slice/transactionFWD_path.service
```

### Confirmed Ownership

PID `320476` belonged to:

```text
transactionFWD_path.service
```

and **not**:

```text
stock_sip_txn.service
```

This was a critical finding because both services had similar names.

For comparison:

```bash
sudo systemctl show stock_sip_txn.service -p MainPID -p ExecStart --no-pager
```

showed:

| Service | Executable / PID |
|---|---|
| `stock_sip_txn.service` | `sip_txn` — PID `292073` |
| `transactionFWD_path.service` | `txn` — PID `320476` — Port `9378` |

### Operational Lesson

Before restarting or killing a process, always verify the **PID-to-service mapping**, preferably through systemd/cgroup information.

---

# 5. Recovery Actions

## 5.1 Signal Sent to Affected Process

The affected PID was signalled:

```bash
sudo kill 320476
```

Immediately afterward, the process was still visible.

However, the application recovered shortly afterward.

---

## 5.2 Port Recovery Verification

The port was checked again:

```bash
sudo ss -lntp | grep ':9378'
```

Result:

```text
LISTEN ... 0.0.0.0:9378 ... users:(("txn",pid=320476,...))
```

### Finding

Port `9378` started listening again under the same PID:

```text
320476
```

This confirmed that the application had recovered.

---

## 5.3 Process State After Recovery

The process state was checked again:

```bash
sudo ps -o pid,stat,wchan:30,etime,cmd -p 320476
```

### State Comparison

| Stage | WCHAN |
|---|---|
| Before recovery | `locks_lock_inode_wait` |
| After recovery | `ep_poll` |

`ep_poll` is the normal event/network wait state for this application.

### Finding

The process returned to a normal operating state.

---

## 5.4 Final Verification

Locks were checked:

```bash
sudo lslocks | grep txn
```

No output was returned.

Service status was verified:

```bash
sudo systemctl status transactionFWD_path.service --no-pager
```

Final state:

| Validation | Result |
|---|---|
| Service | Active (running) |
| Correct service | `transactionFWD_path.service` |
| Process | Running |
| WCHAN | `ep_poll` |
| `txn` lock | No active unexpected lock |
| Port `9378` | Listening |

---

# 6. Root Cause Analysis

## 6.1 Confirmed Root Cause

The `txn` application entered a **file-lock-related blocking state**:

```text
locks_lock_inode_wait
```

This prevented the application from functioning normally and resulted in TCP port `9378` becoming unavailable.

After recovery:

```text
locks_lock_inode_wait
        ↓
     recovery
        ↓
     ep_poll
        ↓
Port 9378 LISTEN
```

---

## 6.2 Underlying Trigger

The underlying trigger for the lock wait was **not conclusively identified** from the available evidence.

The investigation does **not** provide sufficient evidence to state that:

- `/tmp/drogon.lock` was corrupted.
- A second `txn` process definitely caused the lock conflict.
- PID `319203` was definitively responsible for the blocking condition.

PID `319203` was already gone by the time it was investigated.

Therefore, the technically accurate RCA is:

> **Confirmed:** The application entered a file-lock-related blocking state, causing port `9378` to become unavailable.  
> **Not confirmed:** The original trigger that caused the lock wait.

---

# 7. RCA Statement for Jira / Change Record

### Root Cause

`txn` entered a file-lock-related blocking state (`locks_lock_inode_wait`), which prevented the application from functioning normally.

### Impact

TCP port `9378` was not listening, resulting in the transaction-forwarding application being unavailable to downstream consumers.

### Resolution

The affected process was signalled and subsequently recovered. The process returned to the normal `ep_poll` state and port `9378` resumed listening.

### Underlying Trigger

The exact trigger for the lock condition could not be conclusively identified from the available evidence. Further investigation should be performed if the issue recurs.

---

# 8. Important Clarification

During troubleshooting, `stock_sip_txn.service` was also restarted.

However, this was **not established as the fix** for the port `9378` issue.

The actual affected application was:

```text
transactionFWD_path.service
```

The PID-to-service mapping confirmed:

```text
PID 320476
      ↓
transactionFWD_path.service
      ↓
txn
      ↓
Port 9378
```

### Future Rule

Do not restart or kill a service based only on its name.

Always verify:

```bash
systemctl show <service> -p MainPID
```

and, where required:

```bash
cat /proc/<PID>/cgroup
```

---

# 9. Permanent Fix / Preventive Actions

At the time of this RCA, immediate recovery was completed, but a permanent root-cause fix has **not yet been proven**.

The following actions should be reviewed with the application/service owner.

## 9.1 Review Service Configuration

Run:

```bash
sudo systemctl cat transactionFWD_path.service
```

Review:

- `ExecStart`
- `ExecStartPre`
- `ExecStartPost`
- `Restart`
- `Environment`
- `WorkingDirectory`
- `User`
- Startup scripts

---

## 9.2 Review Restart Policy

Run:

```bash
sudo systemctl show transactionFWD_path.service \
  -p MainPID \
  -p ExecStart \
  -p Restart \
  -p RestartUSec \
  --no-pager
```

An appropriate systemd restart policy may help recover from transient application failures.

However, `Restart=always` should **not** be added blindly. The application's expected behaviour and failure modes should first be understood.

---

## 9.3 Review Duplicate Application Startup

Check whether `txn` is being started both manually and through systemd:

```bash
ps -ef | grep '[t]xn'
```

Also inspect:

```bash
systemctl cat transactionFWD_path.service
```

The application should have a clearly defined startup mechanism.

---

## 9.4 Review Lock Handling With Developers

The application owner/development team should verify how `/tmp/drogon.lock` is created, acquired and released.

Specifically confirm:

- Is there a lock-acquisition timeout?
- Is lock failure handled gracefully?
- Is a stale-lock scenario handled?
- Is the lock released gracefully during shutdown?
- Are multiple application instances allowed?
- If multiple instances are not allowed, how is that condition handled?
- Can the process remain alive while waiting indefinitely for the lock?

---

# 10. Monitoring Improvement

The incident exposed a monitoring gap.

### Current Risk

A process-level check such as:

```text
txn process exists = TRUE
```

can incorrectly indicate that the application is healthy.

### Recommended Monitoring

Both process and port health should be monitored.

| Health Check | Expected State |
|---|---|
| `txn` process exists | YES |
| Port `9378` LISTEN | YES |

### Alert Condition

```text
Process exists + Port 9378 NOT LISTENING
                    ↓
                  ALERT
```

This should be treated as an application availability issue.

### Key Monitoring Principle

> **Process health ≠ Application health**

For this service, port-level health monitoring is required in addition to process-level monitoring.

---

# 11. Evidence-First Procedure if the Issue Recurs

**Do not immediately kill the process.**

First collect evidence.

## Step 1 — Service Status

```bash
sudo systemctl status transactionFWD_path.service --no-pager
```

## Step 2 — Identify Main PID

```bash
sudo systemctl show transactionFWD_path.service -p MainPID --no-pager
```

## Step 3 — Check Process State

```bash
sudo ps -o pid,ppid,stat,wchan:30,etime,cmd -p <PID>
```

If the WCHAN shows:

```text
locks_lock_inode_wait
```

perform the full lock investigation.

## Step 4 — Check Port

```bash
sudo ss -lntp | grep ':9378'
```

No output means port `9378` is not listening.

## Step 5 — Check Locks

```bash
sudo lslocks | grep txn
```

Also capture:

```bash
sudo cat /proc/locks | grep '\<PID>'
```

This should ideally be captured **while the issue is active**.

## Step 6 — Inspect Lock File

```bash
stat /tmp/drogon.lock
```

## Step 7 — Check Open Files

```bash
sudo lsof -p <PID>
```

and:

```bash
sudo lsof /tmp/drogon.lock
```

## Step 8 — Verify Service Ownership

```bash
sudo cat /proc/<PID>/cgroup
```

This confirms which systemd service actually owns the process.

## Step 9 — Review Application Logs

```bash
sudo tail -100 /home/ubuntu/uat/project-cash/src/transaction/orders/drogon.log
```

For live troubleshooting:

```bash
sudo tail -f /home/ubuntu/uat/project-cash/src/transaction/orders/drogon.log
```

Search for relevant errors:

```bash
sudo grep -iE 'error|fail|lock|exception|timeout' \
  /home/ubuntu/uat/project-cash/src/transaction/orders/drogon.log
```

## Step 10 — Review Systemd Journal

```bash
sudo journalctl -u transactionFWD_path.service --since "30 minutes ago"
```

For the current boot:

```bash
sudo journalctl -u transactionFWD_path.service -b --no-pager
```

> In this incident, the journal had already rotated by the time it was checked and returned truncated/no-entry information. Future incidents should be investigated immediately so the relevant journal entries are still available.

## Step 11 — Recover

Only after evidence collection, recover the application using the **correct service/process**.

## Step 12 — Validate Recovery

Confirm all of the following:

```text
Service = Active (running)
Process = Running
WCHAN   = Normal / ep_poll
Port    = 9378 LISTEN
Locks   = No unexpected txn lock
```

---

# 12. Future Troubleshooting Flow

```text
Incident Reported
       |
       v
Check transactionFWD_path.service
       |
       v
Identify MainPID
       |
       v
Check Process STAT/WCHAN
       |
       v
Check Port 9378
       |
       +---- Port LISTEN ----> Application likely available
       |
       +---- Port DOWN
                |
                v
          Check lslocks
                |
                v
          Check lsof
                |
                v
       Verify PID → Service
                |
                v
       Review Application Logs
                |
                v
       Review Systemd Journal
                |
                v
       Collect Evidence
                |
                v
       Recover Correct Service
                |
                v
       Validate Service + Process + Port + Locks
```

---

# 13. Quick Reference — Commands

### 1. Service Status

```bash
sudo systemctl status transactionFWD_path.service --no-pager
```

### 2. Identify PID

```bash
sudo systemctl show transactionFWD_path.service -p MainPID --no-pager
```

### 3. Process State

```bash
sudo ps -o pid,ppid,stat,wchan:30,etime,cmd -p 320476
```

### 4. Port Check

```bash
sudo ss -lntp | grep ':9378'
```

### 5. Lock Check

```bash
sudo lslocks | grep txn
```

### 6. Open Files

```bash
sudo lsof -p 320476
```

### 7. Lock File Details

```bash
ls -l /tmp/drogon.lock
```

```bash
stat /tmp/drogon.lock
```

### 8. Service Ownership

```bash
sudo cat /proc/320476/cgroup
```

### 9. Application Logs

```bash
sudo tail -100 /home/ubuntu/uat/project-cash/src/transaction/orders/drogon.log
```

### 10. Systemd Journal

```bash
sudo journalctl -u transactionFWD_path.service --since "30 minutes ago"
```

### 11. Service Configuration

```bash
sudo systemctl cat transactionFWD_path.service
```

### 12. Restart Policy

```bash
sudo systemctl show transactionFWD_path.service \
  -p MainPID \
  -p ExecStart \
  -p Restart \
  -p RestartUSec \
  --no-pager
```

---

# 14. Final Key Takeaways

1. **A running process does not necessarily mean the application is healthy.**
2. Port `9378` must be monitored in addition to the `txn` process.
3. The confirmed problematic state was `locks_lock_inode_wait`.
4. After recovery, the process returned to `ep_poll` and port `9378` started listening.
5. PID `320476` was confirmed to belong to `transactionFWD_path.service`.
6. `stock_sip_txn.service` was a separate service and should not be confused with `transactionFWD_path.service`.
7. The exact underlying trigger for the lock wait was **not conclusively identified**.
8. `/tmp/drogon.lock` should not be deleted without evidence that it is actually causing the problem.
9. During a recurrence, collect `/proc/locks`, `lslocks`, `lsof`, cgroup, application logs and systemd journal **before taking recovery action**.
10. Any restart/kill action must first confirm the correct PID-to-service mapping.

---

## Final RCA

> **The `txn` Transaction Forward Path application became unavailable because its process entered a file-lock-related kernel wait state (`locks_lock_inode_wait`), resulting in port `9378` not listening. The process was confirmed to belong to `transactionFWD_path.service`. After the affected process was signalled, it recovered, returned to the normal `ep_poll` state, and port `9378` resumed listening. The precise underlying trigger for the lock condition could not be conclusively established from the available evidence.**
