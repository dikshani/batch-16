http://10.0.0.83:8011/globalsearch/v1/algosearch?searchkey=rel&page=0&size=15&type=0

# Service Down — Troubleshooting Runbook
 
Reference guide for diagnosing and fixing a service that is unreachable (e.g. `ERR_CONNECTION_REFUSED`, port not listening, crashed process).
 
## Step 1 — Connectivity Check
 
Confirm the target machine is reachable on the network.
 
```bash
ping <IP>
```
 
## Step 2 — Port Check (from remote)
 
Confirm whether the specific port is open or refusing connections.
 
```bash
telnet <IP> <PORT>
# or
nc -zv <IP> <PORT>
```
 
## Step 3 — Verify Port Locally (on the machine)
 
Log in to the target machine and confirm nothing is listening on that port.
 
```bash
sudo ss -tulnp | grep <PORT>
sudo ss -tulnp | grep LISTEN     # list all listening services
```
 
## Step 4 — Identify How the Service Is Managed
 
Check each process manager in turn to find where the service is registered.
 
```bash
pm2 list
pm2 show <service-name>
 
sudo systemctl list-units --type=service | grep -i <keyword>
sudo systemctl status <service-name>
 
docker ps -a
docker ps -a | grep -i <keyword>
```
 
## Step 5 — Locate the Service Source Code
 
Search the codebase and deployment configs to find the exact service.
 
```bash
grep -rl "<route-keyword>" ~/project-folder/src/
grep -rl "<PORT>" ~/project-folder/deploy/nginx/*.conf
grep -rl "<PORT>" ~/project-folder/deploy/
cat ~/project-folder/deploy/nginx/<file>.conf   # check upstream / proxy_pass
```
 
## Step 6 — Check Config / Env Files
 
```bash
cat <service-folder>/.env
cat <service-folder>/config.js
cat <service-folder>/package.json
```
 
## Step 7 — Install Dependencies (most common fix)
 
A missing `node_modules` folder is the single most common cause of a crash-on-start.
 
```bash
cd <service-folder>
npm install
```
 
## Step 8 — Start the Service
 
```bash
# PM2
pm2 start <entry-file.js> --name <service-name>
 
# systemd
sudo systemctl start <service-name>
sudo systemctl enable <service-name>   # auto-start on boot
 
# Docker
docker-compose up -d <service-name>
```
 
## Step 9 — Check Logs Immediately
 
```bash
pm2 logs <service-name> --lines 50 --nostream
sudo journalctl -u <service-name> -n 50
docker logs <container-name> --tail 50
```
 
## Step 10 — Test Locally
 
```bash
curl "http://localhost:<PORT>/<route>?param=value"
```
 
## Step 11 — Persist the Configuration
 
Critical — without this the fix will not survive a server reboot.
 
```bash
pm2 save
sudo systemctl enable <service-name>
```
 
## Step 12 — Restart Commands (already-registered service)
 
```bash
pm2 restart <service-name>
sudo systemctl restart <service-name>
docker restart <container-name>
```
 
## Step 13 — End-to-End Test
 
Retest the original public URL from a browser or Postman to confirm full resolution.
 
## Most Common Root Causes (check in this order)
 
1. `npm install` was never run → `node_modules` missing
2. Service was never registered in PM2 / systemd / Docker
3. `.env` or config file missing or incorrect
4. Redis / database connection failure
5. Port conflict with another service
6. `pm2 save` was skipped → service disappeared after a reboot
## Worked Example — `global_search_api` (10.0.0.83:8011)
 
Real incident trace, kept here as a reference of the full diagnostic path:
 
```
1. ping 10.0.0.83                          → host reachable
2. telnet 10.0.0.83 8011                   → connection refused
3. sudo ss -tulnp | grep 8011              → no output, confirmed down
4. grep -rl "algosearch" src/              → found src/api/search
5. grep -B5 -A10 "8011" deploy/nginx/*.conf → upstream global_search_api
                                               server 10.0.0.83:8011
6. pm2 list                                 → not registered
7. pm2 start src/api/search/app.js \
     --name global_search_api              → started, but crash-looped
8. pm2 logs global_search_api               → ERR_MODULE_NOT_FOUND: express
9. cd src/api/search && npm install         → installed dependencies
10. pm2 restart global_search_api           → stable, online
11. curl localhost:8011/globalsearch/v1/get → valid JSON response
12. pm2 save                                → persisted across reboots
```
