# A5 Integration Evidence
## Job Board API ↔ Notification Hub Team20

**Main System:** Job Board API  
**Partner System:** Notification Hub Team20  
**Test Date:** 22 September 2026

**Job Board API:** https://jobboard-api-lz9f.onrender.com  
**Notification Hub:** https://notification-hub-team20.onrender.com  
[6731503012 Taweesak Sangkoranee]

[6731503036 Wichayapon Seepin]

[6731503044 Aubolwan Maneechan]

[6731503123 Sawitta Thiabsaeng]

[6731503127 Onpreya Thinan]
> Evidence screenshots are hosted in the public GitHub repository `Taweesaksangkorane/Md` so this document can be submitted as a single `.md` file.

---

# 1. Consumer Proof

## Objective

Demonstrate that the Job Board API can consume the Notification Hub REST API.

## Partner URL

```http
POST https://notification-hub-team20.onrender.com/api/notifications
```

The integration was triggered through the Job Board endpoint:

```http
POST https://jobboard-api-lz9f.onrender.com/api/integration/notifications
```

Request body:

```json
{
  "application_id": 3,
  "message": "Your application has been shortlisted"
}
```

## Request Timestamp and Partner Response

The Job Board recorded the partner request timestamp:

```text
2026-09-22T12:28:34.102Z
```

Notification Hub returned:

```text
HTTP 201 Created
status = created
```

The Job Board integration endpoint returned `200 OK` and included the partner URL, request timestamp, partner HTTP status, and partner response body.

## Evidence

![Consumer Proof](https://raw.githubusercontent.com/Taweesaksangkorane/Md/main/01-consumer-proof.png)

**Figure 1.** Job Board calls the Notification Hub REST API and receives a successful `201 Created` partner response.

## Result

**PASS**

---

# 2. Provider Proof

## Objective

Demonstrate that Job Board provides an API endpoint that Notification Hub can consume.

## Job Board Provider Endpoint

```http
GET https://jobboard-api-lz9f.onrender.com/api/integration/applications/3
```

Partner identification header:

```http
X-Partner-Service: notification-service
```

The provider returned:

```json
{
  "success": true,
  "requested_at": "2026-09-22T12:30:31.568Z",
  "data": {
    "application_id": 3,
    "student_id": 4,
    "job_id": 6,
    "status": "shortlisted",
    "applied_at": "2026-08-29T14:13:32.870835+00:00",
    "updated_at": "2026-09-22T11:27:48.919+00:00"
  }
}
```

## Internal Request Log and Partner Identification

The Job Board Render log recorded:

```text
[INTEGRATION REQUEST]
Timestamp: 2026-09-22T12:30:31.568Z
Partner: notification-service
Endpoint: /api/integration/applications/3
Application ID: 3
Response: 200 OK
```

## Evidence

![Provider API](https://raw.githubusercontent.com/Taweesaksangkorane/Md/main/02-provider-api.png)

**Figure 2.** Provider endpoint returns application data with `200 OK` and the partner identification header.

![Provider Internal Log](https://raw.githubusercontent.com/Taweesaksangkorane/Md/main/03-provider-log.png)

**Figure 3.** Internal Job Board log records the request from `notification-service`, endpoint, application ID, timestamp, and `200 OK`.

## Result

**PASS**

---

# 3. Webhook Receiver

## Objective

Demonstrate that Job Board can receive a callback webhook, verify the shared secret, and store the event.

## Receiver Endpoint

```http
POST https://jobboard-api-lz9f.onrender.com/api/webhooks/notification
```

Test payload:

```json
{
  "event_id": "notification-callback-003",
  "event_type": "NOTIFICATION_DELIVERED",
  "notification_id": "3b01e730-e3bc-4b77-a7f8-62ccd1c50124",
  "application_id": 3,
  "status": "delivered"
}
```

The callback uses:

```http
X-Webhook-Secret: <redacted>
```

The actual secret is stored in the Job Board Render environment as `WEBHOOK_SECRET` and is intentionally omitted from this document.

## Secret Verification Result

The Job Board log showed both a failed verification attempt and a successful verification attempt. The valid request recorded:

```text
Secret verification: PASS
Stored event: notification-callback-003
```

The successful request returned:

```json
{
  "success": true,
  "message": "Webhook received successfully",
  "event_id": "notification-callback-003",
  "stored": true
}
```

## Stored Log / Database Proof

The callback was stored in `integration_events`:

```text
event_id   = notification-callback-003
event_type = NOTIFICATION_DELIVERED
source     = notification-service
```

## Evidence

![Webhook Receiver](https://raw.githubusercontent.com/Taweesaksangkorane/Md/main/04-webhook-receiver.png)

**Figure 4.** Job Board receives the callback and returns `stored: true`.

![Webhook Receiver Log](https://raw.githubusercontent.com/Taweesaksangkorane/Md/main/05-webhook-receiver-log.png)

**Figure 5.** Render log shows the incoming payload, secret verification result, and stored event.

![Stored Webhook Event](https://raw.githubusercontent.com/Taweesaksangkorane/Md/main/06-webhook-stored-event.png)

**Figure 6.** Supabase `integration_events` contains the stored callback event.

## Result

**PASS**

---

# 4. Webhook Sender

## Objective

Demonstrate that an internal Job Board action triggers an outgoing webhook and that Notification Hub successfully responds.

## Internal Trigger Action

Application `3` was updated from `shortlisted` to `accepted`:

```http
PATCH https://jobboard-api-lz9f.onrender.com/api/applications/3/status
```

Request:

```json
{
  "status": "accepted"
}
```

Job Board generated:

```text
event_id = app-status-3-accepted-1790080879215
```

The API response confirmed:

```text
notification_sent   = true
notification_status = sent
```

## Outgoing Webhook

```http
POST https://notification-hub-team20.onrender.com/api/webhooks/jobboard
```

Render recorded the outgoing payload at:

```text
2026-09-22T12:41:21.810Z
```

with:

```json
{
  "event_id": "app-status-3-accepted-1790080879215",
  "event_type": "APPLICATION_STATUS_CHANGED",
  "title": "Job Application Status Updated",
  "message": "Your application status has changed from shortlisted to accepted",
  "severity": "medium",
  "data": {
    "application_id": 3,
    "student_id": 4,
    "job_id": 6,
    "old_status": "shortlisted",
    "new_status": "accepted"
  }
}
```

Job Board signs the outgoing webhook using HMAC-SHA256 and sends the generated signature through `x-event-signature`. The shared `EVENT_WEBHOOK_SECRET` is stored in environment variables and is not included in this document.

## Partner Response Log

Notification Hub returned:

```text
Partner status: 201
status: accepted
```

and returned an `eventReceiptId` and `notificationId`.

## Evidence

![Webhook Internal Trigger](https://raw.githubusercontent.com/Taweesaksangkorane/Md/main/07-webhook-trigger.png)

**Figure 7.** Internal application status update creates an integration event and reports successful notification delivery.

![Webhook Outgoing Payload](https://raw.githubusercontent.com/Taweesaksangkorane/Md/main/08-webhook-outgoing-payload.png)

**Figure 8.** Render log shows the partner URL, timestamp, outgoing payload, and successful partner response.

![Retry Duplicate Handling](https://raw.githubusercontent.com/Taweesaksangkorane/Md/main/09-idempotency-retry-success.png)

**Figure 9.** A repeated sender delivery is safely recognized as a duplicate and the retry finishes successfully.

## Result

**PASS**

---

# 5. Idempotency Proof

## Objective

Demonstrate that sending the exact same webhook event twice results in only one stored event.

## Request 1

The first request used:

```json
{
  "event_id": "notification-idempotency-001",
  "event_type": "NOTIFICATION_DELIVERED",
  "notification_id": "3b01e730-e3bc-4b77-a7f8-62ccd1c50124",
  "application_id": 3,
  "status": "delivered"
}
```

The first response returned:

```json
{
  "success": true,
  "message": "Webhook received successfully",
  "event_id": "notification-idempotency-001",
  "stored": true
}
```

Request timestamp:

```text
2026-09-22T12:46:27.709Z
```

## Request 2

The exact same payload and `event_id` were sent again.

The second response returned:

```json
{
  "success": true,
  "duplicate": true,
  "event_id": "notification-idempotency-001",
  "message": "Event already processed"
}
```

## Database Proof of Single Creation

After both requests, Supabase contained only one row for:

```text
event_id = notification-idempotency-001
```

with:

```text
event_type = NOTIFICATION_DELIVERED
source     = notification-service
```

## Evidence

![Idempotency Request 1](https://raw.githubusercontent.com/Taweesaksangkorane/Md/main/10-idempotency-request1.png)

**Figure 10.** Request 1 is accepted and stored.

![Idempotency Request 2](https://raw.githubusercontent.com/Taweesaksangkorane/Md/main/11-idempotency-request2.png)

**Figure 11.** The identical second request is detected as a duplicate.

![Idempotency Database Proof](https://raw.githubusercontent.com/Taweesaksangkorane/Md/main/12-idempotency-database.png)

**Figure 12.** Database proof shows a single record for `notification-idempotency-001`.

## Result

**PASS**

---

# 6. Degradation Proof

## Objective

Demonstrate graceful degradation when Notification Hub is unavailable, preservation of the pending integration event, and automatic recovery after the partner endpoint is restored.

## Controlled Breakage

For this controlled test, the Job Board `NOTIFICATION_SERVICE_URL` was temporarily changed to an intentionally unavailable endpoint.

Application `3` was updated to `reviewing`, creating:

```text
event_id = app-status-3-reviewing-1790081561585
```

Even though the partner notification service could not be reached, the Job Board primary operation still returned:

```text
HTTP 200 OK
success = true
notification_sent = false
notification_status = pending
```

This is the fallback behavior: the application update succeeds while notification delivery remains pending.

## Breakage Timestamp

The retry log recorded the failed delivery at:

```text
2026-09-22T12:53:54.930Z
```

with:

```text
[WEBHOOK FAILED]
Error: fetch failed
[RETRY FAILED] app-status-3-reviewing-1790081561585: fetch failed
```

The pending event remained available for automatic retry instead of being lost.

## Automatic Recovery

The correct Notification Hub URL was restored without creating a new application event.

The retry worker automatically retried the same event:

```text
app-status-3-reviewing-1790081561585
```

The recovery log later showed:

```text
[RETRY SUCCESS] app-status-3-reviewing-1790081561585
Partner status: 200
status: duplicate
```

The `duplicate` response is successful because Notification Hub had already safely accepted the same event during overlapping delivery attempts. Idempotency prevented a second event from being created, while Job Board completed the retry successfully.

## Evidence

![Degradation Fallback](https://raw.githubusercontent.com/Taweesaksangkorane/Md/main/13-degradation-fallback.png)

**Figure 13.** Fallback JSON shows the Job Board business operation still succeeds while notification delivery is `pending`.

![Degradation Failure Log](https://raw.githubusercontent.com/Taweesaksangkorane/Md/main/14-degradation-failure-log.png)

**Figure 14.** Breakage timestamp and retry failure are recorded while the partner endpoint is unavailable.

![Degradation Recovery](https://raw.githubusercontent.com/Taweesaksangkorane/Md/main/15-degradation-recovery.png)

**Figure 15.** Automatic retry succeeds after the partner URL is restored; duplicate-safe processing prevents duplicate data.

## Result

**PASS**

---

# Final Evidence Summary

| Requirement | Required Evidence | Result |
|---|---|---|
| 1. Consumer Proof | Partner URL, request timestamp, response body | **PASS** |
| 2. Provider Proof | Endpoint URL, internal request log, partner identification/confirmation | **PASS** |
| 3. Webhook Receiver | Incoming payload, secret verification, stored log/database record | **PASS** |
| 4. Webhook Sender | Internal trigger, outgoing payload, partner response log | **PASS** |
| 5. Idempotency Proof | Request 1, Request 2, database proof of single creation | **PASS** |
| 6. Degradation Proof | Breakage timestamp, fallback JSON, automatic recovery log | **PASS** |

---

# Overall Result

```text
Job Board API ↔ Notification Hub Team20
INTEGRATION SUCCESSFUL
```

All six required A5 integration evidence categories were tested and documented.
