# BounceBan API Introduction

> BounceBan is an email verification API. It verifies all kinds of emails and specializes in verifying emails that are accept-all (catch-all) or protected by SEGs (Secure Email Gateways). Instead of marking all accept-all emails as risky, BounceBan identifies which of them are actually deliverable.

## Base URLs

| API | Base URL |
| --- | --- |
| All standard endpoints | `https://api.bounceban.com` |
| Waterfall single verification (Clay, n8n, etc.) | `https://api-waterfall.bounceban.com` |

`/v1/verify/single/status` is available on **both** base URLs, so waterfall integrations can look up past results without switching hosts. All other endpoints exist only on the standard base URL.

## Authentication

All endpoints require an API key sent in the `Authorization` header (no `Bearer` prefix):

```
Authorization: YOUR_API_KEY
```

Manage your API keys, set IP whitelist, and track usage at: https://bounceban.com/app/api/settings

```bash
curl "https://api.bounceban.com/v1/account" \
  -H "Authorization: YOUR_API_KEY"
```

## API keys

Each account can have up to 100 API keys. All keys of an account share its credits, concurrency, rate limits and IP whitelist. For each key you can set:

- **Name** - to tell your integrations apart.
- **Expiry date** (UTC) - the key works until 23:59:59 UTC on that day. The account gets an email 7 days before a key expires.
- **Daily and monthly quotas** - the most verification credits the key can use per UTC day (resets at 00:00 UTC) and per calendar month (resets at 00:00 UTC on the 1st).

Quota rules:

- Single verification, bulk verification and bulk file uploads count against the quota; `/v1/check` does not.
- Credits refunded for `unknown` results go back to the quota.
- A request or bulk task that needs more than what is left is rejected as a whole with HTTP `403` and `"error": "api_key_quota_exceeded"`. Do not retry before `quota_resets_at`.
- For `/v1/verify/bulk/file`, the quota is checked when the file is uploaded and again when it is imported. If other requests used up the quota in between, the task is canceled without charging credits and the account gets an email.

A disabled or expired key is rejected with HTTP `401` and `error` set to `api_key_disabled` or `api_key_expired`. Tasks already submitted with that key keep running, and their results can be fetched with any other active key of the same account.

## Errors

Error responses are JSON with a human-readable `msg`. Some also include `error`, a stable code you can branch on. Do not retry `401` or `403` automatically.

| HTTP status | `error` | Meaning |
| --- | --- | --- |
| `401` | - | API key missing or invalid, IP address not whitelisted, or account blocked. See `msg`. |
| `401` | `api_key_disabled` | The API key has been disabled. |
| `401` | `api_key_expired` | The API key has expired. |
| `401` | `account_not_activated` | The account email has not been verified, including after changing the account email. Verify it, or resend the verification email, at https://bounceban.com/account/profile. |
| `403` | - | Insufficient credits. See `credits_required` and `credits_remaining`. |
| `403` | `api_key_quota_exceeded` | The API key reached its daily or monthly quota. See `quota_period` (`daily` or `monthly`) and `quota_resets_at` (ISO 8601, UTC); `credits_remaining` is what is left of the key's quota. |
| `429` | - | Rate limited. Back off and retry. |

Example `403` for a key over its quota:

```json
{
  "credits_remaining": 12.5,
  "credits_required": 100,
  "msg": "This API key has reached its daily quota of 1000 credits. The quota resets at 2026-10-30 00:00:00 UTC. You can change the quota here: https://bounceban.com/app/api/settings",
  "error": "api_key_quota_exceeded",
  "quota_period": "daily",
  "quota_resets_at": "2026-10-30T00:00:00Z"
}
```

## Credits

- Each successful verification costs **1 credit** (single or bulk).
- Credits are deducted for every request to `/v1/verify/single`, even for repeated verification of the same email.
- For the waterfall endpoint, retrying the same email within 30 minutes of the initial request does **not** deduct extra credits.
- When `disable_catchall_verify=1` and the result is `unknown` with `score: -1`, the credit cost is 0.
- For bulk tasks, credits for `unknown` results are refunded when the task finishes.
- The `/v1/check` endpoint uses a **separate** Check Plan credit balance.
- If the API key has a daily or monthly quota, verification credits also count against it (see API keys above).
- Buy credits: https://bounceban.com/pricing. Test credits: contact dev@bounceban.com.

## Verification results

Every verified email gets a `result` and a `score`:

| Field | Values | Meaning |
| --- | --- | --- |
| `result` | `deliverable` | Safe to send. |
| | `risky` | May bounce; check `score` (higher is better). |
| | `undeliverable` | Will bounce; do not send. |
| | `unknown` | Verification could not determine a result. |
| `score` | `0`-`100` | Deliverability confidence; higher is better. |

Additional boolean flags per email: `is_disposable`, `is_accept_all`, `is_role`, `is_free`. `mx_records` (array) and `smtp_provider` (string, e.g. `Google`) are also returned when available.

## Task status values

| Status | Meaning |
| --- | --- |
| `success` | Verification finished; final result included. |
| `verifying` | Still verifying (e.g. greylisting). Poll status endpoint or use a webhook. Do NOT resubmit - each request costs 1 credit. |
| `queue` | Account rate limit reached; email queued. Poll or use webhook. |

## Verification modes

| Mode | Description |
| --- | --- |
| `regular` | Default. Does not assume the email's domain matches the owner's current company website. |
| `deepverify` | Assumes the email's domain matches the owner's current company website (e.g. from LinkedIn). Improves success rate for accept-all emails. Learn more: https://support.bounceban.com/article/what-is-deepverify |

## Choosing the right endpoint

| Scenario | Use |
| --- | --- |
| One HTTP request, wait for result (Clay, n8n, Zapier-like waterfall) | `GET https://api-waterfall.bounceban.com/v1/verify/single` |
| Async single verification with polling or webhook | `GET /v1/verify/single` + `GET /v1/verify/single/status` |
| Verify a list of emails (up to ~500,000 per task) | `POST /v1/verify/bulk` |
| Verify emails from a CSV file (max 25 MB) | `POST /v1/verify/bulk/file` |
| Quick syntax/domain/role check (no SMTP verification) | `GET /v1/check` |
| Credits balance and rate limits | `GET /v1/account` |

## Rate limits

Default per-account limits (check live values via `GET /v1/account`):

| Endpoint | Limit |
| --- | --- |
| `/v1/verify/single` | 25 per second |
| `/v1/verify/single/status` | 25 per second |
| `/v1/verify/bulk` | 3 per second |
| `/v1/verify/bulk/status` | 25 per second |
| `/v1/verify/bulk/export` | 25 per second |
| `/v1/verify/bulk/dump` | 25 per second |
| `/v1/account` | 5 per second |

Request a higher limit: https://forms.gle/3De4UMZKpxPiPv5M8

## Data retention

Verification results are available for **90 days** after verification.

## Resources

- Interactive API reference (Redoc): https://bounceban.com/public/doc/api.html
- OpenAPI 3.0 spec: https://bounceban.com/public/doc/api.yaml
- Postman workspace: https://www.postman.com/galactic-comet-480286/workspace/bounceban-developers
- Support: dev@bounceban.com
