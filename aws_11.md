# Building a `serverless` Email Reminder System with AWS Lamda

## Overview

This guide builds a **fully serverless request-processing pipeline**. A user visits a static website (hosted on S3), fills out a form, and submits it. That submission triggers a chain of managed AWS services that validate the input, run it through an orchestrated workflow, and ultimately send a confirmation/reminder email — with **no servers to manage** anywhere in the stack.

### Traffic Flow

<img width="1052" height="522" alt="image" src="https://github.com/user-attachments/assets/e48dd8fc-53ca-47cf-8492-83163d4a4ef9" />

**Summary of the Flow:**

1. User interacts with the web app hosted on S3.
2. S3 frontend sends a request to API Gateway.
3. API Gateway forwards the request to Lambda (api_handler Lambda).
4. Lambda triggers AWS Step Functions.
5. Step Functions initiates another Lambda function (Email Lambda).
6. Lambda (Email Lambda) calls Amazon SES to send an email.
7. SES delivers the email notification.

### Why this architecture?
- **S3** serves the frontend cheaply and reliably with no web server.
- **API Gateway** is the single public entry point — handles auth, throttling, CORS.
- **api_handler Lambda** keeps the API layer thin: validate → hand off to orchestration.
- **Step Functions** exists so the workflow (validation, retries, branching, delays, multi-step processing) is **visual, auditable, and easy to extend** later (e.g., "wait 24 hours then send a reminder") without cramming all logic into one Lambda.
- **email Lambda + SES** isolates the "side effect" (sending email) from the orchestration logic — single responsibility.

### What you'll build (in order)
1. IAM roles (least privilege for each Lambda)
2. SES — verify a sender/recipient email identity
3. `email` Lambda — sends email via SES
4. Step Functions state machine — calls the `email` Lambda
5. `api_handler` Lambda — validates input, starts the Step Functions execution
6. API Gateway — public HTTP endpoint that invokes `api_handler`
7. S3 static website — hosts the HTML form that calls API Gateway
8. End-to-end test + validation checklist

We build **backend-to-frontend** (bottom of the diagram up) so that by the time you wire the browser form to API Gateway, every downstream piece already works and you can test each layer in isolation before connecting the next.

---

## Prerequisites

- An AWS account with console access (a non-root IAM user with `AdministratorAccess` for this lab is fine; production would scope this down).
- AWS CLI installed and configured (`aws configure`) — optional but recommended so you can test with `curl`/CLI instead of only the console.
- Basic familiarity with JSON (state machine definitions and IAM policies are JSON).
- A real email address you can access, to verify in SES (SES starts in **Sandbox mode**, which only allows sending to/from **verified** addresses).
- Region: pick one and stay consistent for every resource (this guide assumes `ap-southeast-1`, matching your other labs — swap if you use a different region).

---

## Phase 1 — IAM Roles & Policies

Each Lambda function needs an **execution role**: an IAM role that the Lambda service assumes so the function's code can call other AWS APIs (SES, Step Functions, CloudWatch Logs).

### 1.1 Create the execution role for the `email` Lambda

1. Console → **IAM** → **Roles** → **Create role**.
2. Trusted entity type: **AWS service**. Use case: **Lambda**. Click **Next**.
3. Attach policy: search and select **AWSLambdaBasicExecutionRole** (grants CloudWatch Logs write access — every Lambda needs this so you can see logs).
4. Click **Next**. Role name: `email-lambda-role`. Click **Create role**.
5. Open the role you just created → **Add permissions** → **Create inline policy** → JSON tab, paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ses:SendEmail",
      "Resource": "*"
    }
  ]
}
```

6. Name it `ses-send-email-policy` → **Create policy**.

> **Why scoped this tight?** The `email` Lambda's *only* job is sending mail. It should never have permission to touch S3, DynamoDB, or anything else — this is the principle of least privilege, same as your SG-chaining pattern in the VPC labs, just applied to IAM instead of network ACLs.

### 1.2 Create the execution role for the `api_handler` Lambda

1. Repeat the same **Create role** flow → AWS service → Lambda.
2. Attach **AWSLambdaBasicExecutionRole**.
3. Role name: `api-handler-lambda-role`.
4. Add an inline policy allowing it to **start** (not manage) Step Functions executions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "states:StartExecution",
      "Resource": "arn:aws:states:REGION:ACCOUNT_ID:stateMachine:FormProcessingStateMachine"
    }
  ]
}
```

Replace `REGION` and `ACCOUNT_ID` — you'll come back and fill this in once the state machine exists in Phase 4 (or just use `"Resource": "*"` for now during learning, and tighten it afterward — note this trade-off explicitly, don't leave `*` in anything you call "production").

### 1.3 Create the execution role for Step Functions itself

Step Functions needs its own role to be allowed to **invoke the email Lambda**.

1. **IAM** → **Roles** → **Create role** → Trusted entity: AWS service → Use case: **Step Functions**.
2. Click **Next** → **Next** (no managed policy needed yet) → Name: `stepfunctions-execution-role` → **Create role**.
3. Add inline policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:email-lambda"
    }
  ]
}
```

You'll fill in the exact ARN once the `email` Lambda exists (Phase 3) — or use the console's built-in "Create new role" option when you build the state machine, which auto-generates this policy for you (recommended for beginners — see Phase 4).

---

## Phase 2 — Amazon SES Setup

SES accounts start in the **sandbox**: you can only send **to** and **from** addresses/domains you've verified. This is fine for a lab.

### 2.1 Verify a sender identity

1. Console → **Simple Email Service (SES)** → left menu **Verified identities** → **Create identity**.
2. Identity type: **Email address**.
3. Enter the email you'll send *from*, e.g. `you+sender@example.com`.
4. Click **Create identity**.
5. Check that inbox for a verification email from AWS → click the confirmation link.
6. Back in the console, refresh — status should flip to **Verified**.

### 2.2 Verify a recipient identity

Since you're in sandbox mode, repeat step 2.1 for the address you intend to send *to* (this can be the same address as sender, or a different one you also control — e.g., the "user" submitting the form, for testing).

> **Why two identities?** In sandbox mode, SES rejects `SendEmail` calls if either the `Source` or any `Destination` address is unverified. Once you request **production access** later (Account dashboard → "Request production access"), this restriction is lifted and you can email anyone.

### 2.3 Note your region

SES identity verification is **per-region**. Make sure you verified the identity in the *same region* you'll deploy your Lambdas and Step Functions in.

---

## Phase 3 — Lambda Functions

### 3.1 `email` Lambda (build and test first — it's the leaf of the workflow)

1. Console → **Lambda** → **Create function**.
2. **Author from scratch**.
3. Function name: `email-lambda`.
4. Runtime: **Node.js 20.x** (or Python 3.12 — code below is Node; Python equivalent noted after).
5. Execution role: **Use an existing role** → select `email-lambda-role` (from Phase 1.1).
6. **Create function**.

Replace the default code in `index.mjs` (or `index.js`) with:

```javascript
import { SESClient, SendEmailCommand } from "@aws-sdk/client-ses";

const ses = new SESClient({ region: "ap-southeast-1" }); // match your SES region

export const handler = async (event) => {
  const { toAddress, subject, message } = event;

  const command = new SendEmailCommand({
    Source: "you+sender@example.com", // your verified SES sender
    Destination: {
      ToAddresses: [toAddress || "you+recipient@example.com"], // verified recipient
    },
    Message: {
      Subject: { Data: subject || "Reminder" },
      Body: {
        Text: { Data: message || "This is your reminder email." },
      },
    },
  });

  try {
    const response = await ses.send(command);
    return {
      statusCode: 200,
      messageId: response.MessageId,
    };
  } catch (err) {
    console.error("SES send failed:", err);
    throw err; // let Step Functions see the failure
  }
};
```

> The AWS SDK v3 (`@aws-sdk/client-ses`) ships pre-bundled in the Node.js 20.x Lambda runtime, so no extra layer/zip is needed for this simple case.

7. Click **Deploy**.

**Test it in isolation** (before touching Step Functions):
1. **Test** tab → **Create new test event**.
2. Event JSON:
```json
{
  "toAddress": "you+recipient@example.com",
  "subject": "Test from Lambda",
  "message": "If you got this, SES + Lambda works."
}
```
3. Click **Test** → check the recipient inbox. If it fails, check:
   - CloudWatch Logs (Lambda console → **Monitor** tab → **View CloudWatch logs**) for the exact SES error.
   - Both addresses are **Verified** in SES (Phase 2).
   - The Lambda's region matches the SES-verified identity's region.

### 3.2 `api_handler` Lambda

1. **Lambda** → **Create function** → `api-handler-lambda`.
2. Runtime: Node.js 20.x.
3. Execution role: `api-handler-lambda-role`.
4. Code:

```javascript
import { SFNClient, StartExecutionCommand } from "@aws-sdk/client-sfn";

const sfn = new SFNClient({ region: "ap-southeast-1" });

export const handler = async (event) => {
  // API Gateway (proxy integration) sends the HTTP body as a JSON string in event.body
  let body;
  try {
    body = typeof event.body === "string" ? JSON.parse(event.body) : event.body;
  } catch (err) {
    return {
      statusCode: 400,
      headers: { "Access-Control-Allow-Origin": "*" },
      body: JSON.stringify({ error: "Invalid JSON body" }),
    };
  }

  const { name, email, message } = body || {};

  // --- Basic validation ---
  if (!name || !email) {
    return {
      statusCode: 400,
      headers: { "Access-Control-Allow-Origin": "*" },
      body: JSON.stringify({ error: "name and email are required" }),
    };
  }

  const input = {
    toAddress: email,
    subject: `Hi ${name}, we received your submission`,
    message: message || "Thanks for reaching out — this confirms we got your request.",
  };

  const command = new StartExecutionCommand({
    stateMachineArn: process.env.STATE_MACHINE_ARN,
    input: JSON.stringify(input),
  });

  try {
    const result = await sfn.send(command);
    return {
      statusCode: 200,
      headers: { "Access-Control-Allow-Origin": "*" },
      body: JSON.stringify({
        message: "Workflow started",
        executionArn: result.executionArn,
      }),
    };
  } catch (err) {
    console.error("Failed to start execution:", err);
    return {
      statusCode: 500,
      headers: { "Access-Control-Allow-Origin": "*" },
      body: JSON.stringify({ error: "Failed to start workflow" }),
    };
  }
};
```

5. **Configuration** tab → **Environment variables** → add `STATE_MACHINE_ARN` (you'll fill in the real value after Phase 4 — for now you can put a placeholder and edit it later, or build Phase 4 first and come back).
6. Click **Deploy**.

> **Note on `Access-Control-Allow-Origin: *` headers**: these are here because the browser (S3-hosted page) and the API (API Gateway) live on different origins → this is a **CORS** request. Returning this header from the Lambda is one way to handle CORS on proxy integrations; Phase 5 also shows enabling CORS at the API Gateway level, which is the more standard approach — you generally want both, or at least API Gateway's built-in CORS handling for `OPTIONS` preflight plus this header on actual responses.

---

## Phase 4 — AWS Step Functions

This is the orchestration layer sitting between `api_handler` and `email`. For this lab, the state machine is intentionally simple (one task), but structured so you can extend it later (e.g., add a `Wait` state for a "reminder after 24 hours" flow, or a `Choice` state for branching logic).

### 4.1 Create the state machine

1. Console → **Step Functions** → **Create state machine**.
2. Choose **Write your workflow in code** (Amazon States Language / ASL — this is the JSON-based DSL Step Functions uses).
3. Type: **Standard** (Standard = full execution history + up to 1 year runtime, good for this use case; **Express** is for high-volume, short-duration, no full history — not needed here).
4. Paste this definition:

```json
{
  "Comment": "Form processing workflow: send confirmation email",
  "StartAt": "SendEmail",
  "States": {
    "SendEmail": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:email-lambda",
      "Retry": [
        {
          "ErrorEquals": ["States.TaskFailed"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ],
      "End": true
    }
  }
}
```

Replace `REGION` and `ACCOUNT_ID` with your actual values (find `ACCOUNT_ID` in the console top-right account menu, or run `aws sts get-caller-identity`).

> **Why the `Retry` block?** SES calls can transiently fail (throttling, brief network blips). Step Functions' built-in retry/backoff means you don't have to hand-roll retry logic inside the Lambda — this is one of the main reasons to use Step Functions instead of just calling `email-lambda` directly from `api_handler`.

5. Click **Next**.
6. Name: `FormProcessingStateMachine`.
7. Permissions: choose **Create new role** (simplest for beginners — Step Functions auto-generates a role scoped to invoke exactly the Lambda ARN referenced in your definition) **or** select the existing `stepfunctions-execution-role` from Phase 1.3 if you already attached the correct Lambda ARN to it.
8. Click **Create state machine**.

### 4.2 Copy the State Machine ARN

1. On the state machine's detail page, copy the **ARN** (top of page, looks like `arn:aws:states:ap-southeast-1:123456789012:stateMachine:FormProcessingStateMachine`).
2. Go back to the `api_handler` Lambda → **Configuration** → **Environment variables** → set `STATE_MACHINE_ARN` to this value → **Save**.
3. Also go back to `api-handler-lambda-role` (Phase 1.2) and update the inline policy's `Resource` field with this exact ARN if you left it as a placeholder.

### 4.3 Test the state machine in isolation

1. On the state machine page → **Start execution**.
2. Input:
```json
{
  "toAddress": "you+recipient@example.com",
  "subject": "Step Functions test",
  "message": "This came through the state machine."
}
```
3. **Start execution** → watch the visual graph turn green on success. Check the recipient inbox.
4. If it fails, click the failed state in the graph → **Output** tab shows the Lambda's error — common causes: wrong Lambda ARN in the definition, or the Step Functions role lacking `lambda:InvokeFunction` on that ARN.

---

## Phase 5 — API Gateway

Now expose `api_handler` publicly.

### 5.1 Create the API

1. Console → **API Gateway** → **Create API** → **HTTP API** → **Build** (HTTP API is simpler/cheaper than REST API for this use case; REST API adds more features like request validation and usage plans if you need them later).
2. **Add integration** → Lambda → select `api-handler-lambda` → integration name auto-fills.
3. Click **Next**.
4. Configure routes: Method `POST`, Resource path `/submit`. Click **Next**.
5. Stage name: `$default` (auto-deploys on every change — fine for a lab; use named stages like `prod`/`dev` for anything more serious).
6. **Next** → **Create**.

### 5.2 Enable CORS

1. Left menu → **CORS** (under your API).
2. **Configure**:
   - Access-Control-Allow-Origin: `*` (for the lab; in production, restrict this to your actual S3 website URL/CloudFront domain).
   - Access-Control-Allow-Methods: `POST, OPTIONS`.
   - Access-Control-Allow-Headers: `content-type`.
3. **Save**.

### 5.3 Grant API Gateway permission to invoke the Lambda

Usually the console does this automatically when you add the Lambda integration (it creates a **resource-based policy** on the Lambda allowing `apigateway.amazonaws.com` to invoke it). Verify:
1. Go to `api-handler-lambda` → **Configuration** → **Permissions** → scroll to **Resource-based policy statements** → confirm there's a statement with principal `apigateway.amazonaws.com` and your API's ARN as source.
2. If missing, back in API Gateway the integration setup screen has a button to auto-add this — or add it manually via CLI:
```bash
aws lambda add-permission \
  --function-name api-handler-lambda \
  --statement-id apigateway-invoke \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:REGION:ACCOUNT_ID:API_ID/*/*/submit"
```

### 5.4 Note the Invoke URL

On the API's main page, copy the **Invoke URL** (e.g., `https://abc123xyz.execute-api.ap-southeast-1.amazonaws.com`). Your full endpoint is:
```
https://abc123xyz.execute-api.ap-southeast-1.amazonaws.com/submit
```

### 5.5 Test with curl before touching the frontend

```bash
curl -X POST https://abc123xyz.execute-api.ap-southeast-1.amazonaws.com/submit \
  -H "Content-Type: application/json" \
  -d '{"name":"Rifat","email":"you+recipient@example.com","message":"Testing via curl"}'
```

Expect a `200` JSON response with an `executionArn`, and check the recipient inbox. **Don't move to Phase 6 until this works** — it isolates frontend bugs from backend bugs.

---

## Phase 6 — S3 Static Frontend Hosting

### 6.1 Create the bucket

1. Console → **S3** → **Create bucket**.
2. Bucket name: globally unique, e.g. `rifat-form-frontend-lab`.
3. **Uncheck** "Block all public access" (a static website bucket must be publicly readable) → acknowledge the warning checkbox.
4. Leave other defaults → **Create bucket**.

### 6.2 Enable static website hosting

1. Open the bucket → **Properties** tab → scroll to **Static website hosting** → **Edit**.
2. Enable it → Hosting type: **Host a static website**.
3. Index document: `index.html`.
4. **Save changes**.
5. Note the **Bucket website endpoint** shown here (e.g., `http://rifat-form-frontend-lab.s3-website-ap-southeast-1.amazonaws.com`).

### 6.3 Add a bucket policy for public read

**Permissions** tab → **Bucket policy** → **Edit** → paste (replace bucket name):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::rifat-form-frontend-lab/*"
    }
  ]
}
```

**Save changes**.

### 6.4 Create `index.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Contact Form</title>
</head>
<body>
  <h1>Send us a message</h1>
  <form id="contactForm">
    <label>Name: <input type="text" id="name" required /></label><br /><br />
    <label>Email: <input type="email" id="email" required /></label><br /><br />
    <label>Message: <textarea id="message"></textarea></label><br /><br />
    <button type="submit">Submit</button>
  </form>
  <p id="status"></p>

  <script>
    const API_URL = "https://abc123xyz.execute-api.ap-southeast-1.amazonaws.com/submit"; // <-- your Invoke URL + /submit

    document.getElementById("contactForm").addEventListener("submit", async (e) => {
      e.preventDefault();
      const name = document.getElementById("name").value;
      const email = document.getElementById("email").value;
      const message = document.getElementById("message").value;
      const status = document.getElementById("status");

      status.textContent = "Sending...";

      try {
        const res = await fetch(API_URL, {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({ name, email, message }),
        });
        const data = await res.json();
        status.textContent = res.ok
          ? "Submitted! Check your inbox shortly."
          : `Error: ${data.error}`;
      } catch (err) {
        status.textContent = "Network error — check console.";
        console.error(err);
      }
    });
  </script>
</body>
</html>
```

### 6.5 Upload

1. Bucket → **Objects** → **Upload** → add `index.html` → **Upload**.
2. Visit the **Bucket website endpoint** from step 6.2 in your browser.

---

## Phase 7 — Wire Frontend to API Gateway (already done above, verify here)

1. Confirm `API_URL` in `index.html` exactly matches your Invoke URL + `/submit` (Phase 5.4).
2. Open the S3 website endpoint in a browser → open **DevTools → Network tab** before submitting, so you can inspect the actual request/response.
3. Fill the form with a **verified** email address (sandbox SES restriction) → **Submit**.
4. Confirm:
   - Network tab shows a `POST` to your API Gateway URL returning `200`.
   - No CORS errors in the Console tab (if you see one, re-check Phase 5.2).
   - The recipient inbox receives the email within a few seconds.

---

## Phase 8 — End-to-End Test & Common Failure Points

| Symptom | Likely cause | Fix |
|---|---|---|
| CORS error in browser console | API Gateway CORS not configured, or Lambda response missing `Access-Control-Allow-Origin` header | Redo Phase 5.2; ensure Lambda returns the header on **every** response path, including error paths |
| `403 Forbidden` calling API Gateway | Lambda resource policy missing `apigateway.amazonaws.com` permission | Phase 5.3 |
| API Gateway returns `200` but no email arrives | Step Functions execution failed, or SES rejected due to unverified address | Check Step Functions **Execution history** for the failed state's error output |
| Step Functions execution never starts | `api_handler`'s IAM role lacks `states:StartExecution` on the right ARN, or `STATE_MACHINE_ARN` env var unset/wrong | Recheck Phase 1.2 and 4.2 |
| SES `MessageRejected: Email address not verified` | Sandbox mode + unverified source or destination | Phase 2 — verify both addresses |
| Lambda times out | Default timeout is 3 seconds; SES/Step Functions calls plus cold start can exceed this | Lambda → **Configuration → General configuration** → increase timeout to 10–15s |

---

## Validation Checklist

- [ ] SES sender identity verified
- [ ] SES recipient identity verified (sandbox mode)
- [ ] `email-lambda` sends mail successfully when tested in isolation
- [ ] `email-lambda-role` has only `ses:SendEmail` + basic execution policy (no extra permissions)
- [ ] State machine `FormProcessingStateMachine` created and its `SendEmail` state's `Resource` ARN matches `email-lambda`'s actual ARN
- [ ] Step Functions execution role can invoke `email-lambda` (test a manual execution — turns green)
- [ ] `api-handler-lambda` has `STATE_MACHINE_ARN` env var set to the real ARN
- [ ] `api-handler-lambda-role` can call `states:StartExecution` on that exact ARN
- [ ] API Gateway route `POST /submit` integrated with `api-handler-lambda`
- [ ] API Gateway CORS configured; Lambda resource policy allows API Gateway to invoke it
- [ ] `curl` test against the Invoke URL returns `200` and triggers an email
- [ ] S3 bucket has static website hosting enabled + public read bucket policy
- [ ] `index.html` uploaded, `API_URL` points to the correct Invoke URL + `/submit`
- [ ] Full browser flow: form submit → network 200 → email received

---

## Quick Reference

### Resources created

| Resource | Name/ID | Purpose |
|---|---|---|
| S3 bucket | `rifat-form-frontend-lab` | Static frontend hosting |
| API Gateway (HTTP API) | *(Invoke URL)* | Public entry point, route `POST /submit` |
| Lambda | `api-handler-lambda` | Validates input, starts Step Functions execution |
| Lambda | `email-lambda` | Sends confirmation email via SES |
| Step Functions | `FormProcessingStateMachine` | Orchestrates the (currently single-step) workflow |
| SES identity (sender) | `you+sender@example.com` | Verified "From" address |
| SES identity (recipient) | `you+recipient@example.com` | Verified "To" address (sandbox mode requirement) |

### IAM roles

| Role | Attached to | Key permission |
|---|---|---|
| `email-lambda-role` | `email-lambda` | `ses:SendEmail` |
| `api-handler-lambda-role` | `api-handler-lambda` | `states:StartExecution` on `FormProcessingStateMachine` |
| `stepfunctions-execution-role` | State machine | `lambda:InvokeFunction` on `email-lambda` |

### Environment variables

| Lambda | Variable | Value |
|---|---|---|
| `api-handler-lambda` | `STATE_MACHINE_ARN` | ARN of `FormProcessingStateMachine` |

---

## Key Takeaways

- **Build backend-first, bottom-up**: verify SES → test `email-lambda` alone → test the state machine alone → test `api_handler` alone via `curl` → only then wire up the browser. Each layer tested in isolation makes failures trivial to localize.
- **IAM roles should be scoped per-function, per-purpose** — `email-lambda` can send email but knows nothing about Step Functions; `api-handler-lambda` can start executions but can't send email directly. This mirrors the SG-chaining-by-ID discipline from your VPC labs, just at the IAM layer.
- **Step Functions exists for orchestration concerns** (retries, backoff, branching, waits, visual audit trail) that would otherwise be hand-rolled inside a single Lambda — even a "one task" state machine like this one is worth it the moment you expect the workflow to grow (e.g., adding a delayed reminder step later).
- **SES sandbox mode requires both sender and recipient to be verified** — this is the single most common reason emails silently fail to send in a first-time lab.
- **CORS has two independent halves**: API Gateway must handle the `OPTIONS` preflight, and the Lambda's actual response must also carry `Access-Control-Allow-Origin` — missing either one causes browser-side failures even when `curl` works fine (since `curl` doesn't enforce CORS).
- **`$default` auto-deploy stages are fine for learning** but production APIs should use named stages (`dev`/`prod`) with explicit deployments, matching the "managed services are the production default" principle from your RDS/DocumentDB labs.
