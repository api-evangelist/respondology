---
name: respondology-moderate-a-comment
description: Submit a user comment to Respondology for moderation and correctly handle the asynchronous result webhook, including the duplicate-submission risk.
api: respondology-api
generated: 2026-08-26
method: generated
source: openapi/respondology-api-openapi.json (https://api.respondology.io/swagger.json)
operations:
  - POST /external_api/v1/comments
  - GET /external_api/v1/comments/{id}
  - DELETE /external_api/v1/comments/{id}
---

# Moderate a comment with Respondology

Respondology never returns a moderation decision on the HTTP call. You submit, you get an
acknowledgement, and the decision arrives later on a webhook. Build for that or you will ship a bug.

## Prerequisites

- An `X-Api-Key` issued by a Respondology account manager. There is no self-service key.
- A result endpoint already registered with Respondology (also configured during onboarding —
  you cannot set it via the API).
- Base URL: `https://webhooks.respondology.io/`

## Step 1 — Submit the comment

`POST /external_api/v1/comments` with header `X-Api-Key: <key>`.

Required: `account_id`, `message`. Set `moderate: true` to request a moderation action;
`moderate: false` records and analyzes the comment without acting on it.

Useful optional fields: `post_id`, `parent_comment_id` (for replies), `comment_permalink`,
`posted_at` (UTC ISO 8601), `media[]`, `message_tags[]`, and the `user` object
(`id`, `screen_name`, `display_name`, `avatar_url`, `owner`, `bio`, `created_at`, `profile_link`).

**Always send `custom`.** It is a free-form object echoed back verbatim on the result webhook, and it
is the only place to put your own record id.

## Step 2 — Store the identifiers from the 200

The response returns `comment_id`, `account_id`, `screen_name`, `message`, and `request_id`.
Persist `comment_id` and `request_id` before doing anything else — `request_id` is the join key
between this call and the webhook that follows.

A 200 here means *accepted*, not *approved*. Do not surface it to a user as a moderation outcome.

## Step 3 — Handle retries carefully

There is **no idempotency key on this endpoint**. If the request times out and you retry, you will
create a second comment record with a different `comment_id` and receive two result webhooks, and
neither you nor Respondology can tell they were the same intent.

Before retrying a timed-out submission, prefer to reconcile: if you already captured a `comment_id`,
call `GET /external_api/v1/comments/{id}` instead of resubmitting. Only resubmit when you never
received a response body at all, and record that a duplicate is possible.

## Step 4 — Consume the result webhook

Respondology POSTs `comment_result` to your endpoint:

- `action` — `moderated` or `recorded`
- `moderation_result` — `approved` or `rejected` (absent when `action` is `recorded`)
- `moderation_reasons[]` — present on rejection; values come from the 38-term vocabulary in
  `vocabulary/respondology-moderation-reasons.yml`
- `language` — ISO 639-2, or `UN` when undetected
- `request_id`, `comment_id`, `custom` — match these against what you stored in Step 2
- `moderation_completed_at`, `webhook_sending_initiated_at` — UTC ISO 8601

**Return 200 OK.** If you do not, Respondology retries for 72 hours with exponential backoff, so a
handler that 500s on a parse error will be hammered for three days.

No webhook signature scheme is documented, so authenticate the callback with your own control —
a secret path, mTLS, or an IP allowlist agreed with your account manager.

## Step 5 — Errors

Synchronous failures use a flat envelope: `{"error": "<english string>"}`.

- `400` — missing required parameters, e.g. `param is missing or the value is empty: account_id`
- `401` — `API key not found`

There is **no machine-readable error code**, so do not branch on the string. There is no declared
`5xx` and no declared `429`; treat any non-2xx you did not expect as retryable-with-backoff, subject
to the duplicate risk in Step 3.

Asynchronous failures arrive separately on the `comment_error_result` webhook, carrying an `error`
field plus `comment_id` and `request_id`.

## Undoing it

`DELETE /external_api/v1/comments/{id}` returns `202 Accepted`. No result webhook is declared for it
and no time window is published, so you get no confirmation that the delete completed. The contract
does not state whether deleting reverses a hide already applied on the source platform — confirm that
with your account manager before relying on delete as an undo.
