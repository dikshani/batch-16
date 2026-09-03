# VPP QA Aadhar Upload Status Lambda -- Missing Environment Variable Fix

## 1. Problem Statement

The AWS Lambda function:

``` text
ventura1-vpp-qa-AadharUploadStatusFunction
```

was failing during initialization. The failure was observed while
invoking the API associated with:

``` text
/vpp/v1/signup/user/aadhar/upload/status
```

The CloudWatch error indicated:

``` text
KeyError: 'VENTURA1_VPP_ONBOARDING_SLACK_ALERT_CHANNEL'
```

Because the exception occurred while importing/loading the application
configuration, the Lambda could not initialize successfully. This could
result in API Gateway integration failures such as HTTP 502.

The issue was not related to the API Gateway `OPTIONS` method
configuration itself.

------------------------------------------------------------------------

## 2. Impact

The Lambda could not complete initialization because a required
environment variable was missing.

The application configuration was reading environment variables at
module-load time. Therefore, the missing variable caused the function to
fail before the request handler could execute normally.

The affected flow was:

``` text
Client / Browser
      |
      v
API Gateway
      |
      v
Aadhar Upload Status API
      |
      v
Lambda: AadharUploadStatusFunction
      |
      v
auth.py imports app_config.py
      |
      v
Environment variable lookup
      |
      v
KeyError
      |
      v
Lambda initialization failure
      |
      v
API Gateway integration error / 502
```

------------------------------------------------------------------------

## 3. Root Cause

The Lambda code imports the application configuration:

``` python
from app_config import *
```

Inside `app_config.py`, the following environment variables are read
directly using `os.environ[...]`:

``` python
slack_alert_channel = os.environ["VENTURA1_VPP_ONBOARDING_SLACK_ALERT_CHANNEL"]

slack_alert_webhook_url = os.environ[
    "VENTURA1_VPP_ONBOARDING_SLACK_ALERT_CHANNEL_WEBHOOK_URL"
]
```

Using `os.environ["KEY"]` means that the variable must exist. If it does
not exist, Python raises:

``` text
KeyError
```

The affected Lambda did not have the required Slack alert variables
configured.

------------------------------------------------------------------------

## 4. Why These Variables Are Required

The variables are used by the Lambda's Slack error-alert functionality.

The code contains:

``` python
def send_error_alert(client_id, session_id, message):
```

The Slack payload is constructed using:

``` python
slack_data = {
    "channel": slack_alert_channel,
    "blocks": blocks
}
```

The webhook URL is then used to send the alert:

``` python
response = requests.post(
    slack_alert_webhook_url,
    data=json.dumps(slack_data),
    headers=headers
)
```

Therefore, both of these variables are application configuration values:

``` text
VENTURA1_VPP_ONBOARDING_SLACK_ALERT_CHANNEL
VENTURA1_VPP_ONBOARDING_SLACK_ALERT_CHANNEL_WEBHOOK_URL
```

They should not be replaced with arbitrary values.

------------------------------------------------------------------------

## 5. Investigation Performed

### 5.1 Checked the affected Lambda

The affected Lambda was:

``` text
ventura1-vpp-qa-AadharUploadStatusFunction
```

Runtime:

``` text
Python 3.11
```

Package type:

``` text
Zip
```

The Lambda's environment variables were checked.

The Slack-related variables were missing from this Lambda.

------------------------------------------------------------------------

### 5.2 Checked the application code

The relevant code in `app_config.py` showed:

``` python
slack_alert_channel = os.environ["VENTURA1_VPP_ONBOARDING_SLACK_ALERT_CHANNEL"]

slack_alert_webhook_url = os.environ[
    "VENTURA1_VPP_ONBOARDING_SLACK_ALERT_CHANNEL_WEBHOOK_URL"
]
```

This confirmed that the Lambda expected both variables during
initialization.

------------------------------------------------------------------------

### 5.3 Verified another working VPP QA Lambda

Instead of guessing the values, another related VPP QA Lambda was
checked:

``` text
ventura1-vpp-qa-AadharUploadFunction
```

Its environment variables contained both required keys:

``` text
VENTURA1_VPP_ONBOARDING_SLACK_ALERT_CHANNEL
VENTURA1_VPP_ONBOARDING_SLACK_ALERT_CHANNEL_WEBHOOK_URL
```

This provided a valid QA configuration source.

The values were copied from the existing working VPP QA Lambda to the
affected Lambda.

> Secret values, especially the Slack webhook URL, should not be
> committed to Git or documented in plaintext.

------------------------------------------------------------------------

## 6. Solution Implemented

The following two environment variables were added to:

``` text
ventura1-vpp-qa-AadharUploadStatusFunction
```

### Variable 1

``` text
Key:
VENTURA1_VPP_ONBOARDING_SLACK_ALERT_CHANNEL
```

The value was copied from the existing working VPP QA
`AadharUploadFunction`.

### Variable 2

``` text
Key:
VENTURA1_VPP_ONBOARDING_SLACK_ALERT_CHANNEL_WEBHOOK_URL
```

The value was also copied from the existing working VPP QA
`AadharUploadFunction`.

The actual values are intentionally not included in this document
because the webhook URL is sensitive configuration.

------------------------------------------------------------------------

## 7. Why We Copied the Values Instead of Creating New Ones

We did not create random values because:

1.  The application expects a real Slack channel configuration.
2.  The webhook URL must point to the correct QA Slack integration.
3.  Using an incorrect value could cause Slack alerts to fail.
4.  The existing VPP QA Lambda provided a verified configuration source.
5.  Reusing the existing environment-specific configuration reduces the
    risk of introducing a new configuration inconsistency.

------------------------------------------------------------------------

## 8. Lambda Configuration Procedure

The configuration was updated through the AWS Lambda Console.

### Step 1 -- Open Lambda

Open:

``` text
ventura1-vpp-qa-AadharUploadStatusFunction
```

### Step 2 -- Open Configuration

Navigate to:

``` text
Configuration
    -> Environment variables
    -> Edit
```

### Step 3 -- Add the first variable

``` text
Key:
VENTURA1_VPP_ONBOARDING_SLACK_ALERT_CHANNEL
```

Use the corresponding QA value from the working VPP Lambda.

### Step 4 -- Add the second variable

``` text
Key:
VENTURA1_VPP_ONBOARDING_SLACK_ALERT_CHANNEL_WEBHOOK_URL
```

Use the corresponding QA webhook value from the working VPP Lambda.

### Step 5 -- Save

Click:

``` text
Save
```

After saving, the Lambda showed an updated modification time, confirming
that the configuration change was applied.

------------------------------------------------------------------------

## 9. Validation Performed

After updating the environment variables, the Lambda was tested directly
from the AWS Lambda Console.

The test event initially contained the default Hello World JSON:

``` json
{
  "key1": "value1",
  "key2": "value2",
  "key3": "value3"
}
```

Because this Lambda is associated with an API Gateway HTTP request, a
minimal API Gateway-style `OPTIONS` event was used for validation:

``` json
{
  "httpMethod": "OPTIONS",
  "headers": {
    "origin": "https://example.com"
  },
  "body": null
}
```

The Lambda execution result was:

``` text
Executing function: succeeded
```

The execution details also showed:

``` text
Function version: $LATEST
```

and an initialization duration was recorded.

Most importantly, the previous:

``` text
KeyError: 'VENTURA1_VPP_ONBOARDING_SLACK_ALERT_CHANNEL'
```

was no longer present.

This confirmed that the Lambda was able to initialize successfully after
the environment variables were added.

------------------------------------------------------------------------

## 10. API Gateway Configuration Checked

The API Gateway resource was then inspected.

The relevant resource is:

``` text
/vpp/v1/signup/user/aadhar/upload/status
```

The following method was present:

``` text
OPTIONS
```

The API Gateway method execution flow showed:

``` text
Client
   |
   v
Method request
   |
   v
Integration request
   |
   v
Lambda integration
   |
   v
Integration response
   |
   v
Method response
```

The `OPTIONS` method was already configured to use Lambda integration.

Therefore, converting the `OPTIONS` method to a MOCK integration was not
necessary for this particular issue.

------------------------------------------------------------------------

## 11. Important Finding About OPTIONS

The Lambda code already contains explicit handling for `OPTIONS`
requests:

``` python
if event["httpMethod"] == "OPTIONS":
    return check_request_origin(event, header, ALLOWED_ORIGIN_LIST)
```

The helper function returns:

``` python
return {
    "statusCode": 204,
    "headers": header
}
```

Therefore:

``` text
OPTIONS request
      |
      v
Lambda
      |
      v
check_request_origin()
      |
      v
HTTP 204
```

is an intended application flow.

The root cause was the missing environment variable during Lambda
initialization, not the absence of an `OPTIONS` implementation.

------------------------------------------------------------------------

## 12. Before vs After

### Before

Environment variable:

``` text
VENTURA1_VPP_ONBOARDING_SLACK_ALERT_CHANNEL
```

Status:

``` text
Missing
```

Result:

``` text
Lambda initialization failed
```

CloudWatch error:

``` text
KeyError: 'VENTURA1_VPP_ONBOARDING_SLACK_ALERT_CHANNEL'
```

### After

Environment variables:

``` text
VENTURA1_VPP_ONBOARDING_SLACK_ALERT_CHANNEL
VENTURA1_VPP_ONBOARDING_SLACK_ALERT_CHANNEL_WEBHOOK_URL
```

Status:

``` text
Configured
```

Result:

``` text
Lambda initialization succeeded
```

Direct Lambda test:

``` text
Executing function: succeeded
```

------------------------------------------------------------------------

## 13. Security Considerations

The Slack webhook URL is sensitive configuration.

Do NOT:

-   Commit the webhook URL to Git.
-   Put the webhook URL in this Markdown document.
-   Share the webhook URL in Slack or tickets unnecessarily.
-   Hard-code the webhook URL into Python source code.
-   Paste the webhook URL into public documentation.

Recommended practice is to keep sensitive configuration in an
appropriate secret/configuration-management mechanism and reference it
securely from the deployment configuration.

The Markdown document should contain only the environment variable name:

``` text
VENTURA1_VPP_ONBOARDING_SLACK_ALERT_CHANNEL_WEBHOOK_URL
```

and not its actual value.

------------------------------------------------------------------------

## 14. Lessons Learned

### 14.1 Environment variables are part of the deployment

A Lambda can have correct Python code and still fail if required
environment variables are missing.

Application code such as:

``` python
os.environ["VARIABLE_NAME"]
```

creates a hard dependency on that variable being present.

------------------------------------------------------------------------

### 14.2 Do not guess configuration values

When a configuration value is missing, the correct approach is to
identify an authoritative source such as:

-   A working Lambda in the same environment.
-   SAM/CloudFormation configuration.
-   Deployment pipeline configuration.
-   AWS Systems Manager Parameter Store.
-   AWS Secrets Manager.

For this incident, a working VPP QA Lambda was used as the configuration
reference.

------------------------------------------------------------------------

### 14.3 Initialization errors can look like API Gateway errors

The API may appear to have a 502/integration problem, but the actual
root cause can be inside Lambda initialization.

The correct troubleshooting sequence was:

``` text
API Gateway error
      |
      v
Check Lambda
      |
      v
Check CloudWatch
      |
      v
Find initialization error
      |
      v
Inspect app_config.py
      |
      v
Identify missing environment variable
      |
      v
Compare with working Lambda
      |
      v
Fix environment configuration
      |
      v
Retest Lambda
```

------------------------------------------------------------------------

## 15. Final Status

### Resolved

The missing Slack alert environment variables were identified and added
to:

``` text
ventura1-vpp-qa-AadharUploadStatusFunction
```

The Lambda was successfully invoked from the AWS Lambda Test console
after the change.

The previous initialization error:

``` text
KeyError: 'VENTURA1_VPP_ONBOARDING_SLACK_ALERT_CHANNEL'
```

was resolved.

### Remaining Validation

The next validation step is to test the actual API Gateway `OPTIONS`
request for:

``` text
/vpp/v1/signup/user/aadhar/upload/status
```

Expected application response:

``` text
HTTP 204
```

If API Gateway still returns `502`, the next investigation should focus
on the API Gateway integration request/response, Lambda execution logs,
permissions, and the deployed API stage.

------------------------------------------------------------------------

## 16. Quick Troubleshooting Checklist

If the issue happens again:

-   [ ] Confirm the Lambda name.
-   [ ] Check CloudWatch logs.
-   [ ] Check whether the failure occurs during initialization.
-   [ ] Search the error for `KeyError`.
-   [ ] Identify the missing environment variable.
-   [ ] Check `app_config.py` or the relevant configuration module.
-   [ ] Compare environment variables with a working Lambda in the same
    environment.
-   [ ] Never guess Slack/webhook/secret values.
-   [ ] Add the missing non-secret configuration through the deployment
    process where possible.
-   [ ] Keep secrets out of Git.
-   [ ] Save the Lambda configuration.
-   [ ] Invoke the Lambda again.
-   [ ] Test the actual API Gateway route.
-   [ ] Verify the API Gateway response and CloudWatch logs.

------------------------------------------------------------------------

## 17. References

Affected Lambda:

``` text
ventura1-vpp-qa-AadharUploadStatusFunction
```

Reference Lambda:

``` text
ventura1-vpp-qa-AadharUploadFunction
```

API resource:

``` text
/vpp/v1/signup/user/aadhar/upload/status
```

Method checked:

``` text
OPTIONS
```

Runtime:

``` text
Python 3.11
```

Missing configuration identified:

``` text
VENTURA1_VPP_ONBOARDING_SLACK_ALERT_CHANNEL
VENTURA1_VPP_ONBOARDING_SLACK_ALERT_CHANNEL_WEBHOOK_URL
```

Root cause:

``` text
Required Lambda environment variables were missing,
causing app_config.py to raise a KeyError during initialization.
```

Resolution:

``` text
Copied the verified VPP QA Slack configuration from the
working AadharUploadFunction to AadharUploadStatusFunction.
```

Validation:

``` text
Lambda direct invocation: SUCCESS
```
