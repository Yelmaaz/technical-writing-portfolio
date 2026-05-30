> Webhook delivery and event routing for development teams.


## Table of Contents

1. [Overview](#overview)
2. [Quickstart](#quickstart)
3. [Authentication](#authentication)
4. [Sending Events](https://coda.grammarly.com/d/_dyv1tghrpFm/_suevents)
5. [Receiving Webhooks](#receiving-webhooks)
6. [Retry Logic](#retry-logic)
7. [Failure Scenarios](#failure-scenarios)
8. [Security](https://coda.grammarly.com/d/_dyv1tghrpFm/_sucurity)
9. [Event Logs](#event-logs)
10. [Error Reference](#error-reference)
11. [Rate Limits](#rate-limits)
12. [Next Steps](#next-steps)

## Overview

Routepost is a webhook delivery and event routing API that helps development teams send, receive, and manage webhook events between their applications and third-party services.

When your application triggers an event, such as a completed payment, a new user signup, or a failed job, Routepost captures that event, delivers it to your configured endpoints, and handles the operational complexity of reliable delivery: retries, failure logging, payload signing, and delivery status tracking.

**What Routepost handles:**

- Webhook delivery with automatic retry on failure
- Signed payloads for endpoint verification
- Structured delivery logs for debugging
- Configurable retry schedules and timeout thresholds
- Real-time delivery status and failure alerts

**What Routepost does not handle:**

- Event processing logic (that stays in your application)
- Data transformation or payload modification
- Third-party API authentication on your behalf

If your team is manually managing webhook delivery, writing custom retry logic, or debugging failed deliveries without structured logs, Routepost removes that operational overhead.

## Quickstart

Get your first webhook event delivered in under five minutes.

**Prerequisites:**

- A Routepost account (sign up at [routepost.io](http://routepost.io))
- A publicly accessible HTTPS endpoint to receive events
- Basic familiarity with HTTP requests and JSON
### Step 1: Create an API key

Log in to your Routepost dashboard and navigate to **Settings > API Keys**. Click **Generate New Key**, give it a name, and copy the key immediately. It will not be shown again.

```
API Key: rp_live_4a8f92c1d3e74b6a9f0c2d5e8b1a3f7e
```

Store your API key in an environment variable. Never hardcode it in your application.

```bash
export ROUTEPOST_API_KEY="rp_live_4a8f92c1d3e74b6a9f0c2d5e8b1a3f7e"
```
### Step 2: Register your endpoint

Register the URL where Routepost will deliver events.

```bash
curl -X POST https://api.routepost.io/v1/endpoints \
  -H "Authorization: Bearer $ROUTEPOST_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://yourapp.com/webhooks/routepost",
    "description": "Production webhook receiver",
    "events": ["invoice.paid", "user.created", "job.failed"]
  }'
```

A successful response returns your endpoint ID:

```json
{
  "id": "ep_7c3a1f9d2b4e8a0c",
  "url": "https://yourapp.com/webhooks/routepost",
  "status": "active",
  "created_at": "2026-05-30T10:00:00Z"
}
```
### Step 3: Send a test event

```bash
curl -X POST https://api.routepost.io/v1/events \
  -H "Authorization: Bearer $ROUTEPOST_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "event": "invoice.paid",
    "endpoint_id": "ep_7c3a1f9d2b4e8a0c",
    "data": {
      "customer_id": "cus_29184",
      "amount": 1200,
      "currency": "usd"
    }
  }'
```
### Step 4: Verify delivery

Check delivery status in your dashboard under **Logs > Delivery History**, or query the API directly:

```bash
curl https://api.routepost.io/v1/events/evt_5d2a8f1c3b7e9a0d \
  -H "Authorization: Bearer $ROUTEPOST_API_KEY"
```

A successful delivery returns:

```json
{
  "id": "evt_5d2a8f1c3b7e9a0d",
  "status": "delivered",
  "endpoint_id": "ep_7c3a1f9d2b4e8a0c",
  "attempts": 1,
  "delivered_at": "2026-05-30T10:00:03Z",
  "response_code": 200
}
```

If your endpoint returns a non-200 response, Routepost automatically retries delivery. See [Retry Logic](#retry-logic) for details.

## Authentication

All Routepost API requests require an API key passed as a Bearer token in the `Authorization` header.

```bash
Authorization: Bearer rp_live_4a8f92c1d3e74b6a9f0c2d5e8b1a3f7e
```
### Key types

| Key type | Prefix     | Use                     |
| -------- | ---------- | ----------------------- |
| Live     | `rp_live_` | Production environments |
| Test     | `rp_test_` | Development and staging |
Test keys deliver events to registered test endpoints only and do not trigger real deliveries. Use test keys during development to avoid unintended side effects.
### Key management

- Generate and revoke keys under **Settings > API Keys**
- Assign descriptive names to keys (for example, `production-server`, `staging-ci`)
- Rotate keys immediately if exposure is suspected
- Never commit API keys to version control
### Authentication errors

A missing or invalid API key returns:

```json
{
  "error": "unauthorized",
  "message": "Invalid or missing API key.",
  "status": 401
}
```

## Sending Events

Use the `/v1/events` endpoint to send a webhook event to one or more registered endpoints.
### Request

```
POST https://api.routepost.io/v1/events
```
### Request body

```json
{
  "event": "invoice.paid",
  "endpoint_id": "ep_7c3a1f9d2b4e8a0c",
  "data": {
    "customer_id": "cus_29184",
    "amount": 1200,
    "currency": "usd",
    "invoice_id": "inv_00482"
  },
  "idempotency_key": "inv_00482_paid_20260530"
}
```
### Parameters

| Parameter         | Type     | Required    | Description                                         |
| ----------------- | -------- | ----------- | --------------------------------------------------- |
| `event`           | string   | Yes         | Event type identifier (for example, `invoice.paid`) |
| `endpoint_id`     | string   | Yes         | Target endpoint ID                                  |
| `data`            | object   | Yes         | Event payload. Must be valid JSON                   |
| `idempotency_key` | string   | Recommended | Unique key to prevent duplicate delivery            |
### Idempotency

Include an `idempotency_key` with every event to prevent duplicate deliveries due to network errors or retries on your side. Routepost stores idempotency keys for 24 hours.

If a request with a duplicate key is received within that window, Routepost returns the original response without reprocessing the event.
### Response

```json
{
  "id": "evt_5d2a8f1c3b7e9a0d",
  "event": "invoice.paid",
  "endpoint_id": "ep_7c3a1f9d2b4e8a0c",
  "status": "queued",
  "created_at": "2026-05-30T10:00:00Z"
}
```

A `queued` status means Routepost has accepted the event, and delivery is in progress. Use the event ID to track delivery status.

## Receiving Webhooks

Your endpoint receives a POST request from Routepost each time an event is delivered.
### Request format

Routepost sends a JSON payload with the following structure:

```json
{
  "id": "evt_5d2a8f1c3b7e9a0d",
  "event": "invoice.paid",
  "timestamp": "2026-05-30T10:00:03Z",
  "data": {
    "customer_id": "cus_29184",
    "amount": 1200,
    "currency": "usd",
    "invoice_id": "inv_00482"
  },
  "signature": "sha256=3b4f92a1c8d7e2f09b5a3c6d1e8f4a2b7c9d0e3f"
}
```
### Responding correctly

Your endpoint must return a `200 OK` response within **10 seconds** of receiving the request. Any other response code, or a timeout, is treated as a delivery failure and triggers the retry schedule.

Return `200` immediately after receiving the payload. Process the event asynchronously if your handling logic takes longer than a few seconds.

```python
# Example: acknowledge immediately, process async
@app.route('/webhooks/routepost', methods=['POST'])
def receive_webhook():
    payload = request.get_json()
    queue.enqueue(process_event, payload)  # handle async
    return '', 200
```
### Verifying the signature

Always verify the `signature` header before processing an event. See [Security](https://coda.grammarly.com/d/_dyv1tghrpFm/_sucurity) for implementation details.


## Retry Logic

If your endpoint does not return `200 OK` within the timeout window, Routepost retries delivery on an exponential backoff schedule.
### Retry schedule

| Attempt  | Delay after previous attempt |
| -------- | ---------------------------- |
| 1        | Immediate                    |
| 2        | 1 minute                     |
| 3        | 5 minutes                    |
| 4        | 30 minutes                   |
| 5        | 2 hours                      |
| 6        | 6 hours                      |
| 7        | 24 hours                     |


After 7 failed attempts, the event is marked `failed` and no further retries are made. A failure alert is sent to your configured notification email.
### Retry behavior

- Each retry sends an identical payload with the same event ID
- The `X-Routepost-Attempt` header indicates the current attempt number
- Idempotency keys prevent your application from processing the same event twice if delivery is confirmed late
### Disabling retries

Retries can be disabled per endpoint under **Settings > Endpoints** if your application handles deduplication independently. This is not recommended for production endpoints.

## Failure Scenarios

Understanding how Routepost handles failures helps you design resilient integrations.

### Endpoint timeout

Your endpoint did not respond within 10 seconds.

**Routepost behavior:** Marks attempt as failed, begins retry schedule.

**Resolution:** Ensure your endpoint acknowledges the request immediately and processes the payload asynchronously. Avoid synchronous database writes or external API calls in the request handler.

### Non-200 response

Your endpoint returned a 4xx or 5xx response code.

**Routepost behavior:** Treats any non-200 response as failure regardless of response body. Begins retry schedule.

**Common causes:**

- Application error during processing
- Missing route or handler not registered
- Upstream dependency failure (database, cache)

**Resolution:** Check your application logs for the timestamp of the failed attempt. Fix the underlying error before the next retry attempt.



### 500 Internal Server Error

Your endpoint returned a 500 response.

**Routepost behavior:** Same as non-200. Retries according to schedule.

**Note:** Routepost does not inspect response bodies. A 500 with a JSON error message is still treated as a failure.

### Malformed payload

Your application rejected the payload due to an unexpected structure.

**Routepost behavior:** Routepost delivers the payload as sent. If your endpoint returns non-200, retries proceed.

**Resolution:** Validate payloads defensively. Log and discard unrecognized event types rather than returning an error response.

```python
SUPPORTED_EVENTS = {"invoice.paid", "user.created", "job.failed"}

def process_event(payload):
    if payload["event"] not in SUPPORTED_EVENTS:
        logger.info(f"Ignoring unsupported event: {payload['event']}")
        return  # discard gracefully
    # continue processing
```

### Retry exhaustion

All 7 delivery attempts failed.

**Routepost behavior:** Event marked `failed`. Failure alert sent. No further retries.

**Resolution:** Retrieve the failed event from **Logs > Delivery History** and replay it manually once your endpoint is restored.

```bash
curl -X POST https://api.routepost.io/v1/events/evt_5d2a8f1c3b7e9a0d/replay \
  -H "Authorization: Bearer $ROUTEPOST_API_KEY"
```

## Security

### Signed payloads

Every request from Routepost includes a signature in the `X-Routepost-Signature` header. Verify this signature before processing any event to confirm the request originated from Routepost.
### How signing works

Routepost generates a signature using HMAC-SHA256 with your endpoint’s signing secret and the raw request body.

```
signature = HMAC-SHA256(signing_secret, raw_request_body)
```

Your signing secret is available under **Settings > Endpoints > [Endpoint Name] > Signing Secret**.
### Verification example

```python
import hmac
import hashlib

def verify_signature(payload_body, signature_header, signing_secret):
    expected = hmac.new(
        signing_secret.encode('utf-8'),
        payload_body,
        hashlib.sha256
    ).hexdigest()
    received = signature_header.replace("sha256=", "")
    return hmac.compare_digest(expected, received)

@app.route('/webhooks/routepost', methods=['POST'])
def receive_webhook():
    signature = request.headers.get('X-Routepost-Signature')
    if not verify_signature(request.get_data(), signature, SIGNING_SECRET):
        return 'Forbidden', 403
    # process event
    return '', 200
```

Use `hmac.compare_digest` rather than `==` to prevent timing attacks.
### API key handling

- Store API keys in environment variables or a secrets manager
- Never log API keys or include them in error messages
- Rotate keys immediately if a breach is suspected
- Use separate keys for production and staging environments
### HTTPS requirement

Routepost only delivers events to HTTPS endpoints. HTTP endpoints are rejected at registration. Self-signed certificates are not accepted in production environments.

## Event Logs

Routepost stores a full delivery log for every event, including all retry attempts.
### Viewing logs

Access delivery history under **Logs > Delivery History** in your dashboard, or query the API:

```bash
curl https://api.routepost.io/v1/events?endpoint_id=ep_7c3a1f9d2b4e8a0c&status=failed \
  -H "Authorization: Bearer $ROUTEPOST_API_KEY"
```
### Log entry structure

```json
{
  "id": "evt_5d2a8f1c3b7e9a0d",
  "event": "invoice.paid",
  "endpoint_id": "ep_7c3a1f9d2b4e8a0c",
  "status": "failed",
  "attempts": [
    {
      "attempt": 1,
      "timestamp": "2026-05-30T10:00:03Z",
      "response_code": 500,
      "response_time_ms": 320,
      "error": "Internal Server Error"
    },
    {
      "attempt": 2,
      "timestamp": "2026-05-30T10:01:03Z",
      "response_code": 500,
      "response_time_ms": 298,
      "error": "Internal Server Error"
    }
  ],
  "created_at": "2026-05-30T10:00:00Z",
  "last_attempt_at": "2026-05-30T10:01:03Z"
}
```
### Filtering logs

| Parameter     | Description                                                   |
| ------------- | ------------------------------------------------------------- |
| `endpoint_id` | Filter by endpoint                                            |
| `status`      | Filter by status: `delivered`, `failed`, `queued`, `retrying` |
| `event`       | Filter by event type                                          |
| `from` / `to` | Filter by timestamp range (ISO 8601)                          |
### Log retention

Delivery logs are retained for 30 days. Export logs before this window if longer retention is required for compliance purposes.


## Error Reference

All Routepost API errors follow a consistent structure:

```json
{
  "error": "error_code",
  "message": "Human-readable description.",
  "status": 400
}
```
### Error codes

| Code                  | Status   | Description                                  |
| --------------------- | -------- | -------------------------------------------- |
| `unauthorized`        | 401      | Missing or invalid API key                   |
| `forbidden`           | 403      | Key does not have permission for this action |
| `not_found`           | 404      | Resource does not exist                      |
| `validation_error`    | 422      | Request body failed validation               |
| `duplicate_event`     | 409      | Idempotency key already used                 |
| `rate_limit_exceeded` | 429      | Request rate limit reached                   |
| `internal_error`      | 500      | Routepost internal error                     |
### Handling errors

Implement exponential backoff when retrying requests that return `429` or `500`. Do not retry `401`, `403`, or `422` responses without resolving the underlying issue first.


## Rate Limits

Routepost enforces rate limits per API key to ensure consistent performance across all users.
### Limits

| Column 1 | Column 2 |
| --- | --- |
| Operation | Limit |
| Send event | 1,000 requests/minute |
| List events | 100 requests/minute |
| Register endpoint | 10 requests/minute |
### Rate limit headers

Every API response includes the following headers:

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 847
X-RateLimit-Reset: 1748606460
```

`X-RateLimit-Reset` is a Unix timestamp indicating when the limit resets.
### Handling rate limits

A `429` response means your request was not processed. Wait until `X-RateLimit-Reset` before retrying.

```python
import time

def send_event_with_backoff(payload, retries=3):
    for attempt in range(retries):
        response = requests.post(ROUTEPOST_URL, json=payload, headers=HEADERS)
        if response.status_code == 429:
            reset_time = int(response.headers.get('X-RateLimit-Reset', time.time() + 60))
            wait = max(reset_time - time.time(), 1)
            time.sleep(wait)
            continue
        return response
    raise Exception("Rate limit retry exhausted")
```


## Next Steps

You have completed the Routepost Getting Started guide. From here:

**Explore advanced event routing.** Route events to multiple endpoints simultaneously, filter by event type per endpoint, and configure endpoint-level retry policies under **Settings > Endpoints**.

**Set up failure alerts.** Configure email or Slack notifications for delivery failures under **Settings > Notifications**. Alerts fire after the first failed attempt and again after retry exhaustion.

**Integrate with your CI/CD pipeline.** Use Routepost test keys and the `/v1/events/test` endpoint to validate webhook delivery in your staging environment before deploying to production.

**Review the full API reference.** The complete API reference covering all endpoints, parameters, and response schemas is available at [docs.routepost.io/api](https://docs.routepost.io/api).



*Routepost Developer Documentation* *Last updated: May 2026*

