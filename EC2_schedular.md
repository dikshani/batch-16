# Incident Report — invest.venturasecurities.uat Delayed Start

**Date:** 07 September 2026
**Server:** invest.venturasecurities.uat (Instance ID: i-07c0f377cb883b127)
**Scheduler Group:** 2p64k (tag: `Schedulefrom = statemanager_cash`)
**Scheduled Start Time:** 06:30 AM IST

---

## Summary

The server did not start at its scheduled time of 6:30 AM. AWS returned an
**Insufficient Capacity** error when the scheduler tried to start the
instance. The server remained off until it was manually started at 8:37 AM.

---

## Timeline (IST)

| Time | Event | Triggered By |
|---|---|---|
| 06:30:35 AM – 06:30:47 AM | 5 automatic `StartInstances` attempts, all failed | AWS Systems Manager Scheduler (Automation role) |
| 06:30 AM (each attempt) | Error: `Server.InsufficientInstanceCapacity` — "Insufficient capacity." | AWS (system-side) |
| 08:37:01 AM | `StartInstances` succeeded | Manually started by aamir.ansari@venturasecurities.com |

---

## Root Cause

AWS did not have available hardware capacity for this instance type in the
Availability Zone at the scheduled time. This is a **transient, AWS-side
issue** — not a misconfiguration in our scheduler, tags, or permissions.
The scheduler itself worked correctly and retried 5 times within seconds
before giving up.

---

## How This Was Diagnosed (Commands Used)

**1. Checked current instance state:**
```bash
aws ec2 describe-instances --instance-ids "i-07c0f377cb883b127" \
  --query 'Reservations[].Instances[].{State:State.Name,StateReason:StateReason.Message}' \
  --output table --no-cli-pager
```

**2. Pulled all CloudTrail events for the instance during the relevant window:**
```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=i-07c0f377cb883b127 \
  --start-time 2026-09-07T00:00:00Z --end-time 2026-09-07T06:00:00Z \
  --output json --no-cli-pager > invest_events.json
```

**3. Extracted StartInstances events with the user/role that triggered them:**
```bash
cat invest_events.json | python3 -c "
import json
data = json.load(open('invest_events.json'))
for e in data['Events']:
    if e['EventName'] == 'StartInstances':
        print(e['EventTime'], '-', e.get('Username', 'N/A'))
"
```

**4. Extracted the exact error code/message from the failed attempt:**
```bash
cat invest_events.json | python3 -c "
import json
data = json.load(open('invest_events.json'))
for e in data['Events']:
    if e['EventName'] == 'StartInstances' and '01:00:47' in e['EventTime']:
        ct_event = json.loads(e['CloudTrailEvent'])
        print(ct_event.get('errorCode'))
        print(ct_event.get('errorMessage'))
"
```
Result: `Server.InsufficientInstanceCapacity` — `Insufficient capacity.`

---

## Resolution

Manually started by Aamir at 8:37 AM once AWS capacity became available again.
No further action was needed for that day.

---

## Recommendations Going Forward

1. **Treat as one-off for now** — this was a transient AWS capacity issue and
   not expected to recur frequently.
2. **If it repeats for this instance**, consider requesting an
   **On-Demand Capacity Reservation** from AWS for this instance type/AZ to
   guarantee availability at the scheduled time.
3. **Set up a CloudWatch Alarm / SNS notification** on scheduler failures so
   the team is alerted immediately instead of discovering it late.

---

## Manager Summary (Quick Version)

> `invest.venturasecurities.uat` didn't start at its scheduled 6:30 AM time
> because AWS returned an "Insufficient Capacity" error — a temporary issue
> on AWS's side, not related to our setup. It stayed off until 8:37 AM when
> Aamir manually started it. If this repeats, we can request an AWS capacity
> reservation or set up an alert for faster detection.
