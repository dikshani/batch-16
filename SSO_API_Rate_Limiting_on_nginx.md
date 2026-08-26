# SSO API Rate Limiting — Testing Summary
 
**Ticket:** Rate limiting on Nginx for SSO APIs — Block @ 10 RPS / Client ID
 
---
 
## 1. Objective
 
The ticket required rate limiting to be implemented on the SSO API such that **each Client ID is allowed a maximum of 10 requests per second**. Any requests beyond that from the same client_id should be **blocked (HTTP 429)**.
 
My task was to verify whether this rule was **already implemented**, and if so, whether it was **working correctly**.
 
---
 
## 2. What I Did — Step by Step
 
### Step 1: Identified the endpoint and request format
- Used browser DevTools (Network tab) to trace the actual SSO login/OTP flow
- Found the real endpoint: `https://sso2-qa.venturasecurities.com/auth/user/v5/otp/send`
- Method: `POST` (not GET, as initially assumed)
- Identified the required headers: `session_id`, `x-api-key`, `X-Client-Id`
- Body format: `{"client_id": "...", "journey": "login"}`
### Step 2: Checked the actual Nginx config on the server
Logged into the server (`INVEST-UAT-NGINX`) via SSH and reviewed the Nginx config file:
```
/home/ubuntu/nginx-1.29.2/conf.d/sso-qa.venturasecurities.uat.conf
```
 
Findings:
- Rate limiting was **already configured**, but at **50 requests/second** (not 10, as the ticket required)
- Client ID was being tracked via the **`X-Client-Id` HTTP header** (not a query parameter)
- A `burst=20` setting was also in place (allowing a small buffer for sudden spikes)
### Step 3: Used load-testing tools to try to "break" the rate limit
I deliberately sent a large number of requests from a single client_id **all at once (in parallel)** to see exactly when Nginx would start blocking. Two tools were used:
- `curl` + `xargs` (to fire parallel requests)
- `hey` (a load-testing tool that gives a detailed summary)
### Step 4: Changed the rate from 50 to 10 RPS
After confirming with the manager (once the mismatch was found), I edited the config file:
```
Before: rate=50r/s
After:  rate=10r/s
```
Before making the change, I took a backup, verified the syntax with `nginx -t`, then safely reloaded using `docker exec nginx nginx -s reload` (with zero downtime).
 
### Step 5: Re-verified thoroughly after the change
I ran three different types of tests to be fully certain:
 
---
 
## 3. How I Tested — 3 Methods
 
### Test A: Burst Test (sudden spike)
Sent 30 requests from a single client_id **all at the same instant, in parallel**.
 
**Why:** This checks whether the system can handle a sudden flood of requests, like an attacker or a buggy app sending everything at once.
 
**Result (repeated 5 rounds for consistency):**
| Round | Passed | Blocked (429) |
|---|---|---|
| 1-5 | 22 | 8 |
 
All 5 rounds gave the **exact same result** — proving this wasn't a fluke, but reliable, predictable behavior.
 
### Test B: Sustained Rate Test (spread-out load)
Sent requests **spread out over time** instead of all at once — one request every **50 milliseconds** (roughly ~20 requests/second, double the limit).
 
**Why:** This mimics a real-world scenario — a client genuinely sending requests at a sustained high rate (not just a one-time burst).
 
**Result:** The first ~40 requests passed (since the burst buffer was empty and absorbed them), after which the pattern alternated: **pass, block, pass, block...** — meaning exactly ~50% of requests were blocked, which is mathematically correct since I was sending at 2x the 10 RPS limit.
 
### Test C: Independent Check with a Second Client ID
Ran the same tests using a **different client_id** (`AG8075`).
 
**Why:** This confirms that blocking is scoped to that **specific client_id only**, not the whole system — i.e., if one client is blocked, a different client should still get its own fresh limit.
 
**Result:** Got the exact same pattern (22 passed, 8 blocked) — independently, with no impact from `AG3618`.
 
---
 
## 4. Final Results — Before vs After
 
| Parameter | Before | After Fix |
|---|---|---|
| Rate | 50 requests/sec | **10 requests/sec** ✅ (matches ticket) |
| Burst | 20 | 20 (unchanged) |
| Tracking basis | `X-Client-Id` header | Same |
| Per-client independent tracking | — | ✅ Confirmed (tested with 2 client IDs) |
| Reset behavior | — | ✅ Per-second rolling window, no permanent block |
| Sustained high-speed traffic | — | ✅ Correctly blocks ~50% of requests at 2x the limit |
 
---
 
## 5. Two Issues I Flagged (Both Resolved)
 
1. **Rate mismatch:** Config had 50 RPS while the ticket required 10 RPS → **Fixed, now set to 10 RPS**
2. **Client ID source:** Not read from a query parameter, but from the `X-Client-Id` header → Confirmed this is the intended design (the frontend also sends it this way)
---
 
## 6. Conclusion (for the manager, in short)
 
> "I fully tested the SSO API's rate limiting. I first found that the config had 50 RPS set, while the ticket required 10 RPS. After manager confirmation, I safely changed it to 10 RPS — taking a backup, verifying syntax, and reloading with zero downtime. I then tested it three different ways (sudden burst, sustained high-speed traffic, and two separate client IDs) — all tests consistently confirmed that rate limiting is working correctly: it tracks each client_id independently, resets every second, and enforces the exact 10 RPS threshold."
 
---
 
## Appendix: Sample Test Evidence
 
```
=== Burst Test - AG3618 ===
Round 1: 22 passed, 8 blocked (429)
Round 2: 22 passed, 8 blocked (429)
Round 3: 22 passed, 8 blocked (429)
Round 4: 22 passed, 8 blocked (429)
Round 5: 22 passed, 8 blocked (429)
 
=== Burst Test - AG8075 (independent verification) ===
Round 1: 22 passed, 8 blocked (429)
Round 2: 22 passed, 8 blocked (429)
Round 3: 23 passed, 7 blocked (429)
 
=== Sustained Rate Test (20 req/sec, spread over time) ===
First ~40 requests: passed (burst buffer absorption)
After buffer exhausted: alternating pass/block pattern (~50% blocked, matching 2x rate)
```
 
---
 
## Appendix: Nginx Config
 
**Before (rate = 50 RPS):**
```nginx
limit_req_zone $http_x_client_id zone=ratelimit:5m rate=50r/s;
limit_req_status 429;
...
location / {
    limit_req zone=ratelimit burst=20;
    ...
}
```
 
**Command used to change the rate:**
```bash
# 1. Take a backup first
sudo cp /home/ubuntu/nginx-1.29.2/conf.d/sso-qa.venturasecurities.uat.conf \
        /home/ubuntu/nginx-1.29.2/conf.d/sso-qa.venturasecurities.uat.conf.bak_$(date +%Y-%m-%d_%H-%M-%S)
 
# 2. Replace 50r/s with 10r/s directly in the file
sudo sed -i 's/rate=50r\/s/rate=10r\/s/' /home/ubuntu/nginx-1.29.2/conf.d/sso-qa.venturasecurities.uat.conf
 
# 3. Verify the change
sudo grep "limit_req_zone" /home/ubuntu/nginx-1.29.2/conf.d/sso-qa.venturasecurities.uat.conf
 
# 4. Check config syntax before reloading
sudo docker exec nginx nginx -t
 
# 5. Reload Nginx (zero downtime)
sudo docker exec nginx nginx -s reload
```
 
**After (rate = 10 RPS):**
```nginx
limit_req_zone $http_x_client_id zone=ratelimit:5m rate=10r/s;
limit_req_status 429;
...
location / {
    limit_req zone=ratelimit burst=20;
    ...
}
```
 
---
 
## Appendix: Test Commands Used
 
### Test A: Burst Test (5 rounds, single client_id)
```bash
CLIENT_ID="AG3618"
for round in 1 2 3 4 5; do
  echo "=== Round $round ==="
  seq 1 30 | xargs -P 30 -I{} curl -s -o /dev/null -w "%{http_code}\n" -X POST \
    -H "Content-Type: application/json" \
    -H "session_id: <session_id_value>" \
    -H "x-api-key: <api_key_value>" \
    -H "X-Client-Id: $CLIENT_ID" \
    -H "Origin: https://invest.venturasecurities.uat" \
    -d "{\"client_id\": \"$CLIENT_ID\", \"journey\": \"login\"}" \
    "https://sso2-qa.venturasecurities.com/auth/user/v5/otp/send" | sort | uniq -c
  sleep 2
done
```
- `seq 1 30` → generates 30 numbers to loop over
- `xargs -P 30` → fires all 30 requests **in parallel** (at the same instant)
- `-w "%{http_code}\n"` → prints only the HTTP status code of each response
- `| sort | uniq -c` → groups and counts how many requests got each status code
- `sleep 2` → waits 2 seconds between rounds before repeating
### Test B: Sustained Rate Test (spread-out requests)
```bash
CLIENT_ID="AG3618"
for i in $(seq 1 60); do
  curl -s -o /dev/null -w "%{http_code}\n" -X POST \
    -H "Content-Type: application/json" \
    -H "session_id: <session_id_value>" \
    -H "x-api-key: <api_key_value>" \
    -H "X-Client-Id: $CLIENT_ID" \
    -H "Origin: https://invest.venturasecurities.uat" \
    -d "{\"client_id\": \"$CLIENT_ID\", \"journey\": \"login\"}" \
    "https://sso2-qa.venturasecurities.com/auth/user/v5/otp/send" &
  sleep 0.05
done
wait
```
- The loop sends **60 requests total**, one every **50 milliseconds** (`sleep 0.05`), simulating a sustained ~20 requests/second load
- The trailing `&` runs each curl call in the background so the loop doesn't wait for one request to finish before sending the next
- `wait` at the end makes the script pause until all background requests have completed
### Test C: Independent Client ID Check
```bash
CLIENT_ID="AG8075"
for round in 1 2 3; do
  echo "=== Round $round ==="
  seq 1 30 | xargs -P 30 -I{} curl -s -o /dev/null -w "%{http_code}\n" -X POST \
    -H "Content-Type: application/json" \
    -H "session_id: <session_id_value>" \
    -H "x-api-key: <api_key_value>" \
    -H "X-Client-Id: $CLIENT_ID" \
    -H "Origin: https://invest.venturasecurities.uat" \
    -d "{\"client_id\": \"$CLIENT_ID\", \"journey\": \"login\"}" \
    "https://sso2-qa.venturasecurities.com/auth/user/v5/otp/send" | sort | uniq -c
  sleep 2
done
```
Identical to Test A, just run with a second, independent `client_id` to confirm that rate-limit tracking is scoped per client and not global.
 
> **Note:** `session_id` and `x-api-key` values have been replaced with placeholders in this document for security. The actual values used during testing should not be shared or logged outside the team.
