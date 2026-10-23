# Lambda Troubleshooting: SignatureUploadFunction — OpenCV (`cv2`) Missing

## 1. Lambda Details

- **Production function:** `ventura1-co-ssov5-SignatureUploadFunction`
- **Test function:** `ventura1-co-ssov5-SignatureUploadFunction-cv2-test`
- **AWS Region:** `ap-south-1`
- **Runtime:** Python 3.14
- **Architecture:** `x86_64`
- **Original package type:** ZIP
- **Issue:** `No module named 'cv2'`

## 2. Issue Observed

Original Lambda invocation failed during initialization:

```text
Runtime.ImportModuleError:
Unable to import module 'auth': No module named 'cv2'
```

`cv2` OpenCV ka Python module hai. Lambda ke runtime import path mein required OpenCV dependency available nahi thi.

## 3. Original Lambda Layers

Troubleshooting ke waqt original function mein five layers configured the:

1. `vensec-python314-database-layer:7`
2. `PyCryptoDome_py_3_14_Layer:2`
3. `aws-lambda-powertools:4`
4. `mvp-reopening-policy-python314:1`
5. `Non_Cash-common-data-layer-python314:5`

**Important:** Is fix ke dauran production function par koi naya Lambda layer attach nahi kiya gaya.

Lambda mein maximum five layers attach ki ja sakti hain. ZIP package aur layers ka combined uncompressed content bhi Lambda ke 250 MB limit ke andar hona chahiye.

## 4. CloudShell Working Directory

Troubleshooting ke liye directory create/use ki gayi:

```bash
mkdir -p /tmp/signature-cv2-check
cd /tmp/signature-cv2-check
```

Is workspace mein existing function ZIP, extracted code, merged dependencies aur Dockerfile rakhe gaye.

## 5. Existing Function Package and Layers

Existing function code ZIP aur layer ZIPs ko inspect/download karke working directory mein extract kiya gaya.

Dependencies ko merge karne ke liye directory use hui:

```text
layer-merged/python/
```

Function code ko alag directory mein rakha gaya:

```text
handler-test/
```

Ek backup directory bhi maintain ki gayi:

```text
layer-python-backup/
```

Layer ZIP mein Windows-style backslash paths ka issue mila tha. Paths normalize karke dependency files merge ki gayi.

## 6. OpenCV and NumPy Installation

Python 3.14 Linux x86_64 Lambda environment ke liye packages install kiye gaye:

```text
opencv-python-headless==4.14.0.94
numpy==2.3.5
```

CloudShell ka direct Python interpreter Python 3.13 tha, isliye Python 3.14 binary wheels ko host Python mein import karke validate nahi kiya gaya. Validation Lambda Python 3.14 container mein hui.

## 7. Dockerfile

Container image banane ke liye Dockerfile use ki gayi:

```dockerfile
FROM public.ecr.aws/lambda/python:3.14

COPY layer-merged/python/ /opt/python/
COPY handler-test/ ${LAMBDA_TASK_ROOT}/

ENV PYTHONPATH="/opt/python:${LAMBDA_TASK_ROOT}"

CMD ["auth.lambda_handler"]
```

Dependencies `/opt/python/` mein bundle ki gayi aur function code Lambda task root mein copy kiya gaya.

## 8. Local Container Validation

Image build command:

```bash
docker build -t signature-cv2-lambda:test .
```

Image ke andar imports verify kiye gaye. Results:

- OpenCV import successful — version `4.14.0`
- NumPy import successful — version `2.3.5`
- SQLAlchemy import successful — version `2.0.52`
- Signature image processing test successful
- Processed image shape: `(400, 800, 3)`
- Foreground mask shape: `(400, 800)`

Isse confirm hua ki test container mein OpenCV dependency available thi aur image processing operation execute hua.

## 9. ECR Repository and Login

AWS region aur account variables set kiye:

```bash
export AWS_REGION=ap-south-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export REPO=ventura1-signature-cv2-lambda
```

Repository create karne ki command run hui:

```bash
aws ecr create-repository \
  --repository-name "$REPO" \
  --region "$AWS_REGION" \
  --image-scanning-configuration scanOnPush=true
```

Repository already existed, isliye `RepositoryAlreadyExistsException` mila. Existing repository use ki gayi.

ECR login:

```bash
aws ecr get-login-password --region "$AWS_REGION" \
  | docker login \
      --username AWS \
      --password-stdin "$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
```

Output:

```text
Login Succeeded
```

## 10. Image Tag and Push

Image tag:

```bash
docker tag signature-cv2-lambda:test \
  "$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$REPO:cv2-fix"
```

Image push:

```bash
docker push \
  "$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$REPO:cv2-fix"
```

Push successful hui.

- **ECR repository:** `ventura1-signature-cv2-lambda`
- **Image tag:** `cv2-fix`
- **Image URI:**

```text
500535936343.dkr.ecr.ap-south-1.amazonaws.com/ventura1-signature-cv2-lambda:cv2-fix
```

- **Image digest:**

```text
sha256:5038360aa39c2bb583b939ad8acceb4b42aa223080db965b46768f7bad9166f4
```

## 11. Test Lambda Configuration

Separate test function create ki gayi:

```text
ventura1-co-ssov5-SignatureUploadFunction-cv2-test
```

Original configuration se role, VPC, environment variables, timeout, memory aur architecture copy kiye gaye.

Configuration:

- **Memory:** 512 MB
- **Timeout:** 10 seconds
- **Architecture:** `x86_64`
- **VPC:** `vpc-0ece430a071bd913c`
- **Subnets:**
  - `subnet-0769d9a099aef85d0`
  - `subnet-0bcb55ad6f9868323`
- **Security group:** `sg-0efef9bdd7d618983`
- **Execution role:** `ventura1-co-ssov5-LambdaRole`

Environment variables ko local `test-env.json` file mein save kiya gaya aur file permissions restrict ki gayi:

```bash
chmod 600 test-env.json
```

Environment variables mein credentials/tokens ho sakte hain. File ko Git mein commit nahi karna chahiye.

## 12. Test Lambda Invocation

Function active hone ka wait:

```bash
aws lambda wait function-active-v2 \
  --function-name ventura1-co-ssov5-SignatureUploadFunction-cv2-test \
  --region ap-south-1
```

Initial empty event invocation:

```bash
aws lambda invoke \
  --function-name ventura1-co-ssov5-SignatureUploadFunction-cv2-test \
  --region ap-south-1 \
  --cli-binary-format raw-in-base64-out \
  --payload '{}' \
  /tmp/cv2-test-response.json
```

Result:

```json
{
  "errorMessage": "'resource'",
  "errorType": "KeyError"
}
```

Ye `cv2` error nahi tha. Handler ko API Gateway event mein `resource` field chahiye thi, jo empty event mein missing thi.

## 13. API Gateway-Shaped Test Event

Synthetic event file banayi gayi:

```bash
cat > /tmp/apigw-test-event.json <<'EOF'
{
  "resource": "/signature/upload",
  "path": "/signature/upload",
  "httpMethod": "POST",
  "headers": {
    "Content-Type": "application/json"
  },
  "queryStringParameters": null,
  "pathParameters": null,
  "stageVariables": null,
  "requestContext": {
    "requestId": "cv2-test",
    "stage": "qa"
  },
  "body": "{}",
  "isBase64Encoded": false
}
EOF
```

Invocation:

```bash
aws lambda invoke \
  --function-name ventura1-co-ssov5-SignatureUploadFunction-cv2-test \
  --region ap-south-1 \
  --cli-binary-format raw-in-base64-out \
  --payload file:///tmp/apigw-test-event.json \
  /tmp/cv2-test-response.json
```

Response:

```json
{
  "statusCode": 400,
  "body": "{\"phone\": \"\", \"message\": \"There was an error while completing this step.\"}"
}
```

Handler execute hua aur original `No module named 'cv2'` error response mein nahi aaya. `400` response expected ho sakta hai, kyunki synthetic event mein valid phone/signature upload data nahi tha.

## 14. CloudWatch Logs

Test function ke logs check kiye:

```bash
aws logs tail \
  /aws/lambda/ventura1-co-ssov5-SignatureUploadFunction-cv2-test \
  --region ap-south-1 \
  --since 10m
```

Logs mein:

- Initial cold start par ek `INIT_REPORT` initialization timeout record aaya.
- Empty event par `KeyError: 'resource'` aaya.
- Synthetic API Gateway-shaped event handler tak pahucha.
- Latest response `400` tha aur original `cv2` import error nahi tha.

Cold-start timeout agar fresh deployment mein repeat hota hai, to initialization time aur timeout configuration review karni hogi.

## 15. Final Status

### Confirmed

- OpenCV and NumPy imports local Python 3.14 container mein successful hue.
- Signature image processing test successful hua.
- Container image ECR par push hui.
- Separate image-based test Lambda create hui.
- Test Lambda handler tak pahucha bina original `cv2` import error ke.

### Still Pending

- Valid signature-upload request se end-to-end functional test.
- Production deployment/cutover.
- Production function par final verification.

**Production function abhi original ZIP package par hai. Usmein fix deploy nahi hua hai.**

ZIP-based Lambda ko in-place container image mein convert nahi kiya ja sakta. Production remediation ke liye compatible ZIP/layer packaging solution ya controlled migration to a separate image-based function chahiye.

## 16. Security Notes

Troubleshooting ke dauran environment variables mein sensitive-looking API keys, tokens aur database credentials output mein expose hue. Affected credentials ko rotate karna chahiye.

- `test-env.json` ko Git mein commit na karein.
- API keys, passwords, tokens aur database URLs logs/documentation mein include na karein.
- Production cutover se pehle test function ke shared role aur live resource access ko review karein.

## 17. Summary

**Root cause:** Original Lambda ke runtime mein OpenCV module `cv2` available nahi tha.

**Test remediation:** OpenCV and NumPy ko Python 3.14-compatible container image mein bundle karke ECR par push kiya, phir separate test Lambda create ki.

**Result:** Test Lambda mein original `cv2` import error resolve hota hua verify hua. End-to-end signature upload aur production deployment abhi pending hain.
