# Single Verification

Verify one email address at a time. Three endpoints:

1. **Waterfall endpoint** - one synchronous request, recommended for Clay, n8n and similar services.
2. **Standard endpoint** - 15-second timeout, then async via polling or webhook.
3. **Status endpoint** - poll for an async result. Available on **both** base URLs.

---

## Waterfall single verification (recommended for Clay / n8n)

```
GET https://api-waterfall.bounceban.com/v1/verify/single
```

**Note: different base URL** (`api-waterfall.bounceban.com`). Holds the connection open until the result is ready (default 80 s, configurable). No separate polling call needed. If it times out with HTTP `408`, send the same request again - retries within 30 minutes of the initial request for the same email don't deduct credits. The `408` response includes the verification `id`; verification continues in the background, so you can also fetch the finished result later at no charge via [`GET /v1/verify/single/status?id=<id>`](#get-single-verification-result-poll) on this same base URL.

**Only `/v1/verify/single` and `/v1/verify/single/status` are available on the waterfall base URL.** `?email=` always (re)submits the email for verification - it never looks up a past result. Within **30 minutes** of the initial request, resending the same request is **free** and returns the same verification; after that window, it starts a **new, billed** verification. To look up an existing result without any risk of a new charge - at any time within 90 days - use `?id=` with the status endpoint instead.

### Parameters (query)

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `email` | string | Yes | The email to verify. |
| `mode` | string | No | `regular` (default) or `deepverify`. |
| `url` | string | No | Webhook URL to receive the result via HTTP POST. Event header: `event-type: single.email_verification_finished`. |
| `timeout` | integer | No | Timeout in seconds, 30-300, default 80. Must not exceed your own service's timeout limit. |
| `disable_catchall_verify` | string | No | `0` (default) or `1`. When `1`, only basic SMTP verification runs; catch-all/SEG-protected emails return `result: "unknown"`, `score: -1`, cost 0 credits. |

### Example request

```bash
curl "https://api-waterfall.bounceban.com/v1/verify/single?email=dev@bounceban.com" \
  -H "Authorization: YOUR_API_KEY"
```

### Example response (200)

```json
{
  "id": "wf502abcde",
  "status": "success",
  "email": "dev@bounceban.com",
  "result": "deliverable",
  "score": 99,
  "is_disposable": false,
  "is_accept_all": false,
  "is_role": true,
  "is_free": false,
  "mx_records": ["alt1.aspmx.l.google.com", "aspmx.l.google.com"],
  "smtp_provider": "Google",
  "mode": "regular",
  "verify_at": "2022-11-16T07:21:24.943Z",
  "credits_consumed": 1.0,
  "credits_remaining": 375214.0
}
```

### Timeout response (408)

```json
{
  "id": "8416f480",
  "error": "Verification timeout",
  "message": "The email verification request timed out after 80 seconds. Please submit the email again. You won't be charged for resubmitting the same email within 30 minutes of the initial request. Verification continues in the background - you can also fetch the result later at no charge via GET /v1/verify/single/status?id=<id>."
}
```

On `408`: retry the same request - retries within 30 minutes are free. Or use the returned `id` with the [status endpoint](#get-single-verification-result-poll) (free, results kept for 90 days) - verification keeps running in the background after the timeout.

Tutorials: [Clay](https://support.bounceban.com/article/how-to-use-bounceban-with-clay) | [n8n](https://support.bounceban.com/article/how-to-use-the-n8n-node)

---

## Standard single verification

```
GET https://api.bounceban.com/v1/verify/single
```

15-second timeout. If verification finishes in time, the full result is returned. Otherwise the response contains `status: "verifying"` and an `id` - get the final result by polling `/v1/verify/single/status` or via webhook (`url` parameter).

**Costs 1 credit per request, even for the same email.** Never resubmit while `status` is `verifying` or `queue`.

### Parameters (query)

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `email` | string | Yes | The email to verify. |
| `mode` | string | No | `regular` (default) or `deepverify`. |
| `url` | string | No | Webhook URL for the result. Event header: `event-type: single.email_verification_finished`. |
| `disable_catchall_verify` | string | No | `0` (default) or `1`. See waterfall endpoint above. |

### Example request

```bash
curl "https://api.bounceban.com/v1/verify/single?email=dev@bounceban.com" \
  -H "Authorization: YOUR_API_KEY"
```

### Example response - finished (200)

```json
{
  "id": "502abcde",
  "status": "success",
  "email": "dev@bounceban.com",
  "result": "deliverable",
  "score": 99,
  "is_disposable": false,
  "is_accept_all": false,
  "is_role": true,
  "is_free": false,
  "mx_records": ["alt1.aspmx.l.google.com", "aspmx.l.google.com"],
  "smtp_provider": "Google",
  "verify_at": "2022-11-16T07:21:24.943Z",
  "credits_consumed": 1.0,
  "credits_remaining": 375214.0
}
```

### Example response - still verifying (200)

```json
{
  "id": "502abcde",
  "msg": "The email verification process is not yet complete.",
  "status": "verifying",
  "try_again_at": 1731286696
}
```

`try_again_at` is the recommended Unix timestamp for the next poll.

---

## Get single verification result (poll)

```
GET https://api.bounceban.com/v1/verify/single/status
GET https://api-waterfall.bounceban.com/v1/verify/single/status
```

Free - does not cost credits. Results available for 90 days. Available on **both** the standard and the waterfall base URL (same behavior).

### Parameters (query)

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `id` | string | Yes | The `id` returned by `/v1/verify/single` (standard or waterfall, including the `id` in a waterfall `408` timeout response). The same value appears as **Task ID** in the [Credits Usage export](https://bounceban.com/app/account/credits/usage) for `Single (API)` rows. |

### Polling rules (important for agents)

- Cap polling at **10 requests per `id` per 5 minutes**.
- After 5 minutes with no result, stop polling and start a new verification request.
- Prefer webhooks over polling: pass a `url` parameter in the original `/v1/verify/single` request.

### Example request

```bash
curl "https://api.bounceban.com/v1/verify/single/status?id=502abcde" \
  -H "Authorization: YOUR_API_KEY"
```

Responses have the same shape as `/v1/verify/single` above (`success` or `verifying`).

---

## Error responses (all single endpoints)

| HTTP status | Meaning |
| --- | --- |
| `400` | Invalid parameter. |
| `401` | API key missing or invalid, IP address not whitelisted, or account blocked - see `msg`. A disabled or expired key also returns `401`, with `error` set to `api_key_disabled` or `api_key_expired`; an account whose email has not been verified gets `error: account_not_activated`. |
| `403` | Insufficient credits (body includes `credits_required`, `credits_remaining`, `msg`), or the API key reached its daily or monthly quota (`error: api_key_quota_exceeded`, plus `quota_period` and `quota_resets_at`). |
| `408` | Verification timeout (waterfall endpoint only) - retry the same request (free within 30 minutes), or fetch the result later via the status endpoint with the returned `id`. |
| `429` | Rate limited. |
| `500` | Unexpected error. |
