---
name: respondology-submit-a-post
description: Register a social post with Respondology so its comments can be analyzed in context, and handle the post result webhook.
api: respondology-api
generated: 2026-08-26
method: generated
source: openapi/respondology-api-openapi.json (https://api.respondology.io/swagger.json)
operations:
  - POST /external_api/v1/posts
  - PATCH /external_api/v1/posts/{id}
  - GET /external_api/v1/posts/{id}
  - DELETE /external_api/v1/posts/{id}
---

# Submit a post to Respondology

Posts are the container comments hang from. Register the post first so that comments carrying a
`post_id` can be analyzed with their parent context.

## Step 1 — Submit

`POST /external_api/v1/posts` with header `X-Api-Key: <key>`.

Required: `account_id`, `caption`. Optional: `title`, `posted_at` (UTC ISO 8601), `post_permalink`,
`media[]`, `custom`, and a `user` object describing the author.

If the post has only one text field, put it in `caption` — `title` is for the second field.

## Step 2 — Store the response

Returns the post identifiers plus `request_id`. Persist both. As with comments, the 200 is an
acceptance, not a completed analysis, and there is no idempotency key — a blind retry creates a
duplicate post record.

## Step 3 — Result webhook

`post_result` fires when the post has been recorded, carrying `action`, `post_id`, `account_id`,
`screen_name`, `title`, `caption`, `custom`, `webhook_sending_initiated_at`, and `request_id`.

Return 200 OK or Respondology retries for 72 hours with exponential backoff.

`post_error_result` carries an `error` field when processing fails.

> Contract note: `post_error_result` lists `message` in its `required` array but declares no `message`
> property (only `post_title` and `post_caption`). Do not assume `message` will be present.

## Step 4 — Update and link comments

`PATCH /external_api/v1/posts/{id}` updates a post and fires `post_update_result`.

When submitting comments on this post, set `post_id` to the id you stored, and set
`post_created_at` so Respondology can evaluate timing signals — the **Instant Commenter Hide**
moderation reason fires when a commenter replies too soon after post creation, which is a bot signal.

## Step 5 — Remove

`DELETE /external_api/v1/posts/{id}` returns `202 Accepted`. No confirmation webhook and no stated
retention window are published.
