---
name: email-verification
description: Verify email addresses with the BounceBan API. Single and bulk email verification specializing in catch-all (accept-all) and SEG-protected emails, fast email/domain checks (free/disposable/role), credits and rate-limit management, and webhooks. Use when the user asks to verify emails, clean an email list, check deliverability, or detect disposable/free/role addresses.
version: 1.0.0
metadata:
  openclaw:
    emoji: "📧"
    homepage: https://bounceban.com/public/doc/api.html
    requires:
      env:
        - BOUNCEBAN_API_KEY
      bins:
        - curl
    primaryEnv: BOUNCEBAN_API_KEY
---

# BounceBan Email Verification

Verify emails via the BounceBan API. BounceBan specializes in accept-all (catch-all) emails and emails protected by Secure Email Gateways - it identifies which accept-all emails are actually deliverable instead of marking them all as risky.

## Setup

- API key: read from `$BOUNCEBAN_API_KEY` (get one at https://bounceban.com/app/api/settings).
- Auth: send the key in the `Authorization` header, **no `Bearer` prefix**.
- Base URL: `https://api.bounceban.com` (exception: waterfall endpoint uses `https://api-waterfall.bounceban.com`).
- Each verification costs 1 credit. Check balance with `GET /v1/account`.

```bash
curl -s "https://api.bounceban.com/v1/account" -H "Authorization: $BOUNCEBAN_API_KEY"
```

## Choosing an endpoint

| Task | Endpoint | Details |
| --- | --- | --- |
| Verify 1 email, wait for result (simplest) | `GET https://api-waterfall.bounceban.com/v1/verify/single` | references/single-verification.md |
| Verify 1 email, async (poll or webhook) | `GET /v1/verify/single` + `GET /v1/verify/single/status` | references/single-verification.md |
| Verify a list of emails (<= ~500k) | `POST /v1/verify/bulk` | references/bulk-verification.md |
| Verify emails in a CSV file (<= 25 MB) | `POST /v1/verify/bulk/file` | references/bulk-verification.md |
| Task progress | `GET /v1/verify/bulk/status` | references/bulk-verification.md |
| Fetch bulk results (JSON, paginated) | `GET /v1/verify/bulk/dump` | references/bulk-verification.md |
| Fetch results for specific emails (<= 100) | `POST /v1/verify/bulk/emails` | references/bulk-verification.md |
| Export bulk results to CSV link | `POST /v1/verify/bulk/export` | references/bulk-verification.md |
| Stop a queued/verifying bulk task (refunds unverified emails) | `POST /v1/verify/bulk/stop` | references/bulk-verification.md |
| Delete a bulk task (irreversible) | `POST /v1/verify/bulk/destroy` | references/bulk-verification.md |
| Quick check: free/disposable/role/syntax (no SMTP) | `GET /v1/check` | references/check.md |
| Credits balance and rate limits | `GET /v1/account` | references/account.md |
| Webhook payloads and event types | - | references/webhooks.md |

Read the matching reference file before calling an endpoint you haven't used in this session. references/introduction.md covers auth, credits, result enums, and rate limits in full.

## Quick recipes

Verify one email and wait (recommended default; holds connection up to 80 s):

```bash
curl -s "https://api-waterfall.bounceban.com/v1/verify/single?email=someone@example.com" \
  -H "Authorization: $BOUNCEBAN_API_KEY"
```

Verify a list:

```bash
# 1. Create task -> returns {"id": ...}
curl -s -X POST "https://api.bounceban.com/v1/verify/bulk" \
  -H "Authorization: $BOUNCEBAN_API_KEY" -H "Content-Type: application/json" \
  -d '{"emails": ["a@example.com", "b@example.com"]}'
# 2. Poll until status=finished
curl -s "https://api.bounceban.com/v1/verify/bulk/status?id=TASK_ID" \
  -H "Authorization: $BOUNCEBAN_API_KEY"
# 3. Fetch results (cursor pagination; loop until cursor is null)
curl -s "https://api.bounceban.com/v1/verify/bulk/dump?id=TASK_ID&page_size=1000" \
  -H "Authorization: $BOUNCEBAN_API_KEY"
```

Stop a queued or verifying bulk task (only when the user asks; results already returned are kept):

```bash
curl -s -X POST "https://api.bounceban.com/v1/verify/bulk/stop" \
  -H "Authorization: $BOUNCEBAN_API_KEY" -H "Content-Type: application/json" \
  -d '{"id": "TASK_ID"}'
# Then poll /v1/verify/bulk/status until status=finished (at most ~60 s after the stop).
```

## Reading results

- `result`: `deliverable` (safe) | `risky` (check `score`, higher is better) | `undeliverable` (will bounce) | `unknown`.
- `score`: 0-100 deliverability confidence.
- Flags: `is_disposable`, `is_accept_all`, `is_role`, `is_free`; plus `mx_records`, `smtp_provider`.
- Optional `mode=deepverify` parameter improves accept-all accuracy when the email domain matches the owner's company website (e.g. sourced from LinkedIn).

## Critical rules (violating these wastes the user's credits)

1. **Never resubmit an email while its status is `verifying` or `queue`** - every submit to `/v1/verify/single` costs 1 credit. Poll `/v1/verify/single/status` instead (free).
2. Waterfall endpoint HTTP `408` = timeout, not failure: retry the **same** request; retries within 30 minutes are free.
3. Polling cap: max 10 polls per verification `id` per 5 minutes; after 5 minutes without a result, start a new verification instead.
4. Bulk export links expire after 4 hours; results are retained for 90 days.
5. `POST /v1/verify/bulk/destroy` is irreversible - confirm with the user first.
6. To abort a running bulk task, use `POST /v1/verify/bulk/stop` (confirm with the user first). Credits for every email without a result are refunded when the task finishes. A stopped task reports `stopped: true` and `not_verified_count` in status and the `bulk.task_finished` webhook, and `POST /v1/verify/bulk/emails` returns its unverified emails with `status: canceled` (not charged) - report them as not verified, not as failures.
7. For `/v1/verify/bulk/file`, the `file` field must be the LAST field in the multipart form.
8. Rate limits: `/verify/single` 25/s, `/verify/bulk` 3/s, others 25/s (live values: `GET /v1/account`).

## Error handling

| HTTP | Meaning | Action |
| --- | --- | --- |
| 400 | Invalid parameter | Fix the request; body may include details. |
| 401 | Bad/missing API key, IP not whitelisted, or account blocked (see `msg`); `error` may be `api_key_disabled`, `api_key_expired` or `account_not_activated` | Check `$BOUNCEBAN_API_KEY` (no Bearer prefix) and relay `msg`; for a disabled/expired key or unverified account, tell the user to fix it at https://bounceban.com/app/api/settings. Do not retry automatically. |
| 403 | Insufficient credits, or `error: api_key_quota_exceeded` (key's daily/monthly quota) | Credits: report `credits_required` vs `credits_remaining`; buy at https://bounceban.com/pricing. Quota: report `quota_period` and `quota_resets_at`; do not retry before then. |
| 408 | Waterfall timeout | Retry same request (free within 30 min). |
| 429 | Rate limited | Back off and retry with delay. |
| 500 | Server error | Retry once, then report. |
