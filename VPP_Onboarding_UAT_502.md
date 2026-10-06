# VPP Onboarding UAT – Incident RCA & Resolution

## 1. Incident Summary

The VPP Onboarding application in the UAT/STAGE environment experienced startup/build issues on port `3004`.

Two issues were identified and resolved:

1. Next.js production build failure due to a missing `Modal` component import.
2. Port `3004` conflict caused by a temporary Docker container occupying the same port.

After resolution, the application successfully started on port `3004`.

## 2. Environment

| Item | Details |
|---|---|
| Application | VPP Onboarding |
| Environment | UAT / STAGE |
| Application Port | 3004 |
| Process Manager | PM2 |
| Application | Next.js |
| Build | `APP_ENV=STAGE npm run build` |
| Start | `APP_ENV=STAGE next start -p 3004` |

---

## 3. RCA – Next.js Build Failure

### Root Cause

The `Modal` component was used in:

`pages/vpp/itd-details/index.js`

but the required import was missing.

This resulted in:

```text
'Modal' is not defined
react/jsx-no-undef
```

The failed production build prevented the normal PM2 application startup flow.

### Resolution

The missing import was added:

```javascript
import Modal from "../../../components/ui/Modal/Modal.component";
```

A backup of the original file was created before the change.

### Validation

The build was executed using the STAGE environment:

```bash
APP_ENV=STAGE npm run build
```

The build completed successfully:

```text
Compiled successfully
```

The `/vpp/itd-details` route was also successfully generated.

---

## 4. RCA – Port 3004 Conflict

After the build issue was resolved, the application encountered:

```text
Error: listen EADDRINUSE: address already in use 0.0.0.0:3004
```

### Investigation

Port 3004 was checked using:

```bash
sudo lsof -i :3004
```

The port was initially occupied by `docker-proxy`.

The Docker container using the port was identified as:

```text
vpp-web-image-only
```

with:

```text
0.0.0.0:3004->3000/tcp
```

This was a temporary testing container that had remained running.

### Resolution

The temporary container was stopped:

```bash
docker stop vpp-web-image-only
```

After stopping it, port 3004 was occupied by:

```text
node 34710 ubuntu ... TCP *:3004 (LISTEN)
```

The process was verified using:

```bash
ps -fp 34710
```

It was confirmed to be the actual VPP Next.js application:

```text
node .../node_modules/.bin/next start -p 3004
```

Therefore, the Node process was expected and was not terminated.

The temporary Docker container was then permanently removed:

```bash
docker rm vpp-web-image-only
```

This ensures that the temporary container cannot start again and reclaim port `3004`.

---

## 5. Final System State

### VPP Onboarding

The application successfully started with:

```text
ready - started server on 0.0.0.0:3004
```

### Docker

Final relevant state:

```text
vpp-web-image-only → Removed
vpp-web            → Running on port 3001
sso_node           → Running on port 3000
```

The original `vpp-web` container remained unchanged.

### Port 3004

Port `3004` is used by the actual VPP Onboarding Next.js application.

---

## 6. Final RCA

### Primary Application RCA

The VPP Onboarding production build was failing because the `Modal` component was referenced in `pages/vpp/itd-details/index.js` without the required import.

This caused the Next.js production build to fail and prevented normal application startup.

### Secondary Runtime RCA

After the build issue was fixed, port `3004` was occupied by the temporary Docker container `vpp-web-image-only`.

Since the PM2-managed VPP application also uses port `3004`, the application encountered:

```text
EADDRINUSE: address already in use 0.0.0.0:3004
```

The temporary container was stopped and removed. The actual VPP application was then confirmed running successfully on port `3004`.

---

## 7. Resolution Summary

| Issue | Root Cause | Resolution | Status |
|---|---|---|---|
| Build failure | Missing `Modal` import | Added required import | Resolved |
| Application startup failure | Failed production build | Successful STAGE build | Resolved |
| Port 3004 conflict | Temporary `vpp-web-image-only` container | Container stopped and removed | Resolved |
| Application on 3004 | PM2/Next.js process | Confirmed healthy | Resolved |

---

## 8. Verification Commands

### PM2 status

```bash
pm2 status
```

### Application logs

```bash
pm2 logs vpp-onboarding-web --lines 30
```

Expected:

```text
ready - started server on 0.0.0.0:3004
```

### Port verification

```bash
sudo lsof -i :3004
```

Expected listener:

```text
node ... next start -p 3004
```

### Local application check

```bash
curl -I http://localhost:3004
```

---

## 9. Preventive Actions

1. Do not run temporary Docker containers using port `3004` while the PM2-managed VPP application is active.
2. Before starting a temporary container, verify whether the host port is already in use.
3. Remove temporary test containers after validation.
4. Before restarting PM2, verify that no unrelated process/container is occupying port `3004`.
5. Validate builds using the same environment used by PM2:

```bash
APP_ENV=STAGE npm run build
```

6. Check application logs after every deployment/restart.
7. Avoid unnecessary PM2 restarts when the application is already healthy.

---

## 10. Existing Warnings

The following warnings were observed:

- Invalid `experimental.images.domains` configuration.
- `<img>` usage instead of Next.js `<Image />`.
- Missing `alt` attributes for some image elements.
- Automatic Static Optimization warning related to `getInitialProps`.
- Node Fetch API experimental warning.

These warnings did not prevent the STAGE production build from completing successfully and were not the primary RCA for this incident.

---

## 11. Final Status

**Status: RESOLVED**

The VPP Onboarding application is successfully running on port `3004`.

The temporary Docker container that caused the port conflict has been removed and cannot automatically start again.

The original `vpp-web` Docker container remains on port `3001` and was not removed or repurposed.
