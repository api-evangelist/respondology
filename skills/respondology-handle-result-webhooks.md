---
name: respondology-handle-result-webhooks
description: Build a correct receiver for all six Respondology result webhooks, covering correlation, retry pressure, and the unverified-sender gap.
api: respondology-api
generated: 2026-08-26
method: generated
source: openapi/respondology-api-openapi.json webhooks object (6 definitions)
operations:
  - webhook comment_result
  - webhook comment_update_result
  - webhook comment_error_result
  - webhook post_result
  - webhook post_update_result
  - webhook post_error_result
---

# Receive Respondology result webhooks

Every outcome in this API arrives here. If the receiver is wrong, the integration is wrong.

## Register the endpoint

You cannot set the callback URL through the API. It is configured during onboarding; changes go
through your account manager (support@respondology.com). Plan for that lead time.

## The six events

| Event | Fires when |
|---|---|
| `comment_result` | Moderation completed, or comment recorded when moderation was not requested |
| `comment_update_result` | Moderation completed for an updated comment |
| `comment_error_result` | Error during comment processing or moderation |
| `post_result` | Post recorded |
| `post_update_result` | Post updated |
| `post_error_result` | Error during post processing |

## Correlate on request_id

Every event echoes the `request_id` returned at submission, plus `comment_id` or `post_id`, plus your
`custom` payload verbatim. Key your reconciliation on `request_id`; use `custom` to carry your own
primary key so you never have to maintain a mapping table.

## Always return 200 OK, fast

A non-200 puts you into a **72-hour exponential-backoff retry** loop. Two consequences:

1. Acknowledge before you do slow work. Persist the raw body, return 200, process asynchronously.
2. A handler that throws on an unexpected field will be retried for three days. Parse defensively.

Because retries are expected, **make your processing idempotent on `request_id`** — the same event
can legitimately be delivered more than once.

## Verify the sender yourself

No signature, HMAC, or shared-secret scheme is documented for these callbacks. Anyone who learns your
URL can POST to it. Mitigate with an unguessable path, mTLS, or an IP allowlist arranged with your
account manager, and never trust webhook content as authenticated input.

## Reading a moderation outcome

On `comment_result`, branch on `action` first:

- `action: recorded` — no moderation was requested. `moderation_result`, `moderation_reasons`, and
  `moderation_completed_at` are **absent**. Do not read them.
- `action: moderated` — `moderation_result` is `approved` or `rejected`. On `rejected`,
  `moderation_reasons[]` lists every reason that fired, from the 38-term vocabulary in
  `vocabulary/respondology-moderation-reasons.yml`.

`language` is ISO 639-2, or `UN` when detection failed.

## Known contract defects to code around

- `comment_error_result` requires `message` but declares the property as `comment_message`.
- `post_error_result` requires `message` but declares no `message` property at all.

Treat both as optional and read whichever key is present.
