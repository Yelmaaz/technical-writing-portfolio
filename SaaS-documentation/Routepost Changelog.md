Routepost is an event delivery infrastructure for distributed SaaS systems. This changelog documents all notable changes across releases, including breaking changes, migration requirements, new features, deprecations, and bug fixes.
Routepost follows [semantic versioning](https://semver.org). Breaking changes increment the major version. New backward-compatible features increment the minor version. Bug fixes and patches increment the patch version.

## v2.0.0 — May 2026
### Overview
v2.0.0 is a major release introducing scoped token authentication, a restructured events API, and expanded observability. This release contains breaking changes. All integrations using v1.x authentication or the legacy `/deliver` endpoint must migrate before the deprecation deadline.

See the [Migration Guide](#migration-guide-v1x-to-v200) below for step-by-step instructions.

### Breaking Changes
1. API key authentication replaced with scoped token authentication
**What changed:**  
The `X-Routepost-Key` header used for authentication in v1.x has been replaced with Bearer token authentication using scoped access tokens. Legacy API keys are no longer accepted.
**Why:**  
Previously, a single API key granted full access. Routepost now supports multi-tenant and service-level deployments, making a flat key model a security risk. Scoped tokens restrict permissions, limiting the impact of compromised credentials.
**What breaks:**  
Any integration that passes credentials via `X-Routepost-Key` will receive a `401 Unauthorized` error after upgrading.
**Before:**
```bash
curl -X POST https://api.routepost.io/v1/events \
  -H "X-Routepost-Key: rp_abc123xyz"
```
**After:**
```bash
curl -X POST https://api.routepost.io/v1/events \
  -H "Authorization: Bearer rt_live_4a8f92c1d3e74b6a9f0c2d5e8b1a3f7e"
```
2. `/v1/deliver` endpoint removed
**What changed:**  
The `/v1/deliver` endpoint introduced in v1.0.0 has been removed. Event delivery now uses `/v1/events` exclusively.
**Why:**  
The `/v1/deliver` endpoint was introduced as a convenience alias during the v1.0.0 beta period. Maintaining two delivery endpoints created inconsistencies in log attribution and retry handling. `/v1/events` is the canonical endpoint and has been since v1.1.0.
**What breaks:**  
Any integration that calls`/v1/deliver` will receive a `404 Not Found`.
**Before:**
```bash
curl -X POST https://api.routepost.io/v1/deliver \
  -H "X-Routepost-Key: rp_abc123xyz" \
  -d '{"event": "invoice.paid", ...}'
```
**After:**
```bash
curl -X POST https://api.routepost.io/v1/events \
  -H "Authorization: Bearer rt_live_xxxxx" \
  -d '{"event": "invoice.paid", ...}'
```
3. Webhook signature header renamed
**What changed:**  
The webhook signature header has been renamed from `X-Routepost-Signature` to `Routepost-Signature` to align with emerging industry conventions for vendor-prefixed headers.
**What breaks:**  
Endpoint verification logic that reads the `X-Routepost-Signature` header will fail to find the signature and may reject valid deliveries.
**Before:**
```python
signature = request.headers.get('X-Routepost-Signature')
```
**After:**
```python
signature = request.headers.get('Routepost-Signature')
```
### Migration Guide: v1.x to v2.0.0
Complete all steps before the deprecation deadline of **August 31, 2026**. Legacy v1.x credentials stop functioning on that date.
Step 1: Generate a scoped access token
Log in to your Routepost dashboard and navigate to **Settings > Access Tokens**. Click **Generate Token** and select the required permission scopes for your integration.
Available scopes:

| Scope             | Permission                    |
| ----------------- | ----------------------------- |
| `events:write`    | Send events                   |
| `events:read`     | Read event logs and status    |
| `endpoints:write` | Register and manage endpoints |
| `endpoints:read`  | Read endpoint configuration   |
For most integrations, `events:write` and `endpoints:read` are sufficient. Issue tokens with the minimum required scopes.
Copy the token immediately. It will not be shown again.
```
Token: rt_live_4a8f92c1d3e74b6a9f0c2d5e8b1a3f7e
```
Store the token in an environment variable:
```bash
export ROUTEPOST_TOKEN="rt_live_4a8f92c1d3e74b6a9f0c2d5e8b1a3f7e"
```
Step 2: Replace authentication headers
Replace every instance of `X-Routepost-Key` in your integration with `Authorization: Bearer`.
**Before:**
```bash
-H "X-Routepost-Key: rp_abc123xyz"
```
**After:**
```bash
-H "Authorization: Bearer $ROUTEPOST_TOKEN"
```
Step 3: Update endpoint paths
Replace any calls to `/v1/deliver` with `/v1/events`.
```bash
# Before
POST https://api.routepost.io/v1/deliver

# After
POST https://api.routepost.io/v1/events
```
No request body changes are required. The `/v1/events` endpoint accepts the same payload structure as `/v1/deliver`.
Step 4: Update webhook signature verification
Update your endpoint verification logic to read `Routepost-Signature` instead of `X-Routepost-Signature`.
**Before:**
```python
signature = request.headers.get('X-Routepost-Signature')
```
**After:**
```python
signature = request.headers.get('Routepost-Signature')
```
Signing secrets remain unchanged. No secret rotation is required unless you choose to rotate as a security precaution.
Step 5: Validate in your test environment
Use a test-scoped token (`rt_test_`) to validate your updated integration against the Routepost test environment before promoting to production.
```bash
export ROUTEPOST_TOKEN="rt_test_9b3c2e1d4a7f8b0c5d2e6f3a1b4c7d9e"
```
Test tokens route deliveries to registered test endpoints only and do not trigger real event processing.
Step 6: Revoke legacy API keys
After validating your migration in production, revoke all legacy API keys under **Settings > API Keys**. Leaving unused credentials active unnecessarily increases your attack surface.
### New Features
**Scoped access tokens**  
Access tokens now support explicit permission scopes. Issue service-specific credentials with minimum required permissions. See [Authentication](https://github.com/Yelmaaz/technical-writing-portfolio/blob/main/SaaS-documentation/Routepost%20Developer%20Documentation.md#authentication) for full details.
**Event filtering per endpoint**  
Endpoints can now be configured to receive only specific event types. Previously, all endpoints received all events and filtering required application-level logic.
```bash
curl -X PATCH https://api.routepost.io/v1/endpoints/ep_7c3a1f9d2b4e8a0c \
  -H "Authorization: Bearer $ROUTEPOST_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"events": ["invoice.paid", "invoice.failed"]}'
```
**Delivery attempt metadata**  
Event log entries now include response body snapshots (up to 1KB) from delivery attempts. Previously, only response codes and latency were recorded. Response body logging aids debugging of application-level rejections.
**Token expiry configuration**  
Access tokens can now be issued with configurable expiry windows (1 day, 7 days, 30 days, or no expiry). Short-lived tokens are recommended for CI/CD integrations.
### Improvements
- Retry backoff intervals are now configurable per endpoint. Previously, all endpoints used the platform default schedule.
- Delivery log retention has been extended from 30 days to 90 days.
- `/v1/events` response latency reduced by approximately 18% following queue infrastructure optimization.
- Idempotency key deduplication window extended from 24 hours to 72 hours.
### Deprecations

| Column 1                                | Column 2   | Column 3          |
| --------------------------------------- | ---------- | ----------------- |
| Feature                                 | Deprecated | Removal           |
| `X-Routepost-Key` authentication header | v1.1.0     | Removed in v2.0.0 |
| `/v1/deliver` endpoint                  | v1.1.0     | Removed in v2.0.0 |
| `X-Routepost-Signature` webhook header  | v2.0.0     | August 31, 2026   |
### Known Issues
- Batch delivery logs may show delayed aggregation metrics for queues processing more than 10,000 events per minute. Individual event records are accurate. Aggregate dashboard metrics may lag by up to 5 minutes under sustained high volume.
- Token scope validation errors return `403 Forbidden` without a detailed scope breakdown in the response body. A descriptive error response is planned for v2.1.0.

## v1.1.0 — February 2026
### New Features
**Batch event sending**  
Send up to 100 events in a single API request using the new `/v1/events/batch` endpoint. Batch delivery reduces API call overhead for high-volume integrations.
```bash
curl -X POST https://api.routepost.io/v1/events/batch \
  -H "X-Routepost-Key: rp_abc123xyz" \
  -H "Content-Type: application/json" \
  -d '{
    "events": [
      {
        "event": "invoice.paid",
        "endpoint_id": "ep_7c3a1f9d2b4e8a0c",
        "data": {"customer_id": "cus_001", "amount": 800}
      },
      {
        "event": "invoice.paid",
        "endpoint_id": "ep_7c3a1f9d2b4e8a0c",
        "data": {"customer_id": "cus_002", "amount": 1200}
      }
    ]
  }'
```
Batch requests are processed atomically per event. A failure in one event does not affect the delivery of others in the batch. Each event receives an independent retry schedule.
**Retry policy configuration**  
Retry schedules can now be configured per endpoint. Available policies:

| Policy         | Description                                                  |
| -------------- | ------------------------------------------------------------ |
| `default`      | Platform default exponential backoff (7 attempts)            |
| `aggressive`   | 10 attempts with shorter intervals for time-sensitive events |
| `conservative` | 5 attempts with longer intervals to reduce endpoint load     |
| `none`         | No retries. Deliver once and mark failed on non-200          |
Configure retry policy when registering or updating an endpoint:
```bash
curl -X PATCH https://api.routepost.io/v1/endpoints/ep_7c3a1f9d2b4e8a0c \
  -H "X-Routepost-Key: rp_abc123xyz" \
  -H "Content-Type: application/json" \
  -d '{"retry_policy": "aggressive"}'
```
**Event replay**  
Failed events can now be replayed via the API without manual intervention from the Routepost dashboard.
```bash
curl -X POST https://api.routepost.io/v1/events/evt_5d2a8f1c3b7e9a0d/replay \
  -H "X-Routepost-Key: rp_abc123xyz"
```
### Bug Fixes
- Fixed an issue where idempotency keys containing special characters were incorrectly returning `422 Unprocessable Entity` responses. Idempotency keys now accept any URL-safe string up to 255 characters.
- Fixed delivery log timestamps displaying in server local time rather than UTC for accounts in non-UTC timezones.
- Fixed a race condition where simultaneous delivery attempts to the same endpoint could produce duplicate log entries under high concurrency.
### Deprecations

| Feature                                 | Status                                                               | Planned removal |
| --------------------------------------- | -------------------------------------------------------------------- | --------------- |
| `X-Routepost-Key` authentication header | Deprecated. Use `Authorization: Bearer` with scoped tokens in v2.0.0 | v2.0.0          |
| `/v1/deliver` endpoint                  | Deprecated. Use `/v1/events`                                         | v2.0.0          |
Deprecation warnings are now returned in the `Routepost-Deprecation` response header for requests using deprecated features:
```
Routepost-Deprecation: X-Routepost-Key is deprecated and will be removed in v2.0.0. Migrate to Bearer token authentication before August 31, 2026.
```

## v1.0.0 — November 2025
### Initial Release
Routepost v1.0.0 is the initial public release of the Routepost event delivery API.
**Core capabilities:**
- Send webhook events to registered HTTPS endpoints via `POST /v1/events`
- Automatic retry on delivery failure with exponential backoff (7 attempts)
- HMAC-SHA256 signed payloads via `X-Routepost-Signature` header
- Idempotency key support with a 24-hour deduplication window
- Delivery logs with per-attempt response codes and latency
- Live and test environment support via `rp_live_` and `rp_test_` key prefixes
- Rate limits: 1,000 events/minute, 100 log queries/minute, 10 endpoint registrations/minute
**Supported resources:**

| Resource      | Endpoint             |
| ------------- | -------------------- |
| Events        | `POST /v1/events`    |
| Endpoints     | `POST /v1/endpoints` |
| Event status  | `GET /v1/events/:id` |
| Delivery logs | `GET /v1/events`     |
See the [Routepost Developer Documentation](https://github.com/Yelmaaz/technical-writing-portfolio/blob/main/SaaS-documentation/Routepost%20Developer%20Documentation.md) for full API reference.
### Known Issues at Launch
- Batch event sending is not supported in v1.0.0. Each event requires an individual API request. Batch support is planned for v1.1.0.
- Delivery log retention is 30 days. Longer retention periods are not configurable in this release.
- Retry schedules are not configurable per endpoint. All endpoints use the platform default backoff schedule.



*Routepost API Changelog*  
*Last updated: May 2026*