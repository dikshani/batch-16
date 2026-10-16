# UAT Charting API 500 Error — Resolution Document

**Date:** 16 October 2026  
**Environment:** UAT  
**Server:** `ip-10-0-0-83`  
**Container:** `charting-app`  
**Port:** `8099`

## 1. Issue Description

The following API was returning `500 Internal Server Error`:

`/charting/fno/v4/candlechart?token=864571&period=1d`

## 2. Root Cause

Application logs showed:

```text
ReferenceError: typ1 is not defined
at /charting/fno/candlechart.js:129:72
```

In `/charting/fno/candlechart.js`, the variable was declared as `type1`, but line 129 incorrectly referenced `typ1`.

## 3. Fix Applied

Used `sed` to correct the variable directly inside the running Docker container without opening the file in an editor.

```bash
sudo docker exec charting-app sh -lc '
cp /charting/fno/candlechart.js /charting/fno/candlechart.js.bak &&
sed -i "s/typ1 == \"BFO\"/type1 == \"BFO\"/" /charting/fno/candlechart.js &&
grep -n "BFO" /charting/fno/candlechart.js
'
```

The incorrect condition:

```javascript
hours <= 10 && typ1 == "BFO"
```

was corrected to:

```javascript
hours <= 10 && type1 == "BFO"
```

Restarted the container:

```bash
sudo docker restart charting-app
```

## 4. Verification

Checked the corrected source:

```bash
sudo docker exec charting-app sh -lc \
'sed -n "124,132p" /charting/fno/candlechart.js'
```

Searched for remaining occurrences of the incorrect variable:

```bash
sudo docker exec charting-app sh -lc \
'grep -Rnw /charting --include="*.js" -e "typ1" 2>/dev/null'
```

No matches were returned.

Tested the API locally:

```bash
curl -sS -i --max-time 15 \
'http://127.0.0.1:8099/charting/fno/v4/candlechart?token=864571&period=1d'
```

**Result:** `HTTP 200 OK`

The API returned candle-chart JSON data successfully.

## 5. Final Status

**Immediate UAT issue: Resolved and verified.**

The variable typo was corrected, the container was restarted, and the API returned HTTP 200 with candle-chart data.

## 6. Permanent Fix / Follow-up

The change was made directly inside the running container. It may be lost if the container is replaced or a new image is deployed.

For a permanent fix:

1. Update `typ1` to `type1` in the source repository.
2. Review and commit the change.
3. Build and deploy a new `charting-app` image using the existing CI/CD pipeline.
4. Retest the API after deployment.

**Note:** The source repository has not yet been confirmed as updated.
