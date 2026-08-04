---
name: Register and verify nCino Mortgage webhooks
description: Register a listener, subscribe it to events, verify the HMAC signature on delivery, and test the whole path without producing real platform activity.
api: openapi/ncino-mortgage-openapi.yml
operations: [webhooks-create, webhooks-index, webhooks-show, webhooks-update, webhooks-destroy, webhooks-actions, webhooks-test_events, subscriptions-create, subscriptions-index, subscriptions-show, subscriptions-update, subscriptions-destroy]
---

# Register and verify nCino Mortgage webhooks

Authenticate first — see `ncino-authenticate-and-call.md`.

## 1. Register a listener

`webhooks-create` — `POST /webhooks` — with the listener URL. Registering the same URL
twice returns **409** ("Webhook already exists for the given URL"), so call
`webhooks-index` first if you are not certain.

## 2. Subscribe it to events

`subscriptions-create` — `POST /webhooks/{webhook_id}/subscriptions` — once per event.
Subscribing twice to the same event returns **409**. The 35 available events are listed
in `asyncapi/ncino-mortgage-webhooks.yml` and declared machine-readably in
`openapi/ncino-mortgage-webhooks-openapi.json` under OpenAPI 3.1 `webhooks`.

Common choices: `loan_created`, `loan_updated`, `loan_app_submitted`,
`loan_milestone_completed`, `document_uploaded`, `job_status_changed`,
`verification_available`.

## 3. Verify every delivery

Each delivery carries an `x-api-signature` header. Recompute HMAC-SHA256 over the
**raw** request body — the exact bytes received, not a re-serialized copy — keyed with
the webhook secret, and compare in constant time. Reject on mismatch. Do not act on an
unverified payload.

## 4. Respond correctly

Return a success status. The nCino docs name both `200` and `201` in different places;
either is safe. If nCino does not receive a success status it will resend later, so the
listener must be idempotent on its own side — the API gives you no delivery id guarantee
to lean on beyond the event body.

## 5. Test without real activity

`webhooks-test_events` — `POST /webhooks/{webhook_id}/test_events` — fires a test
delivery so the signature check and the listener can be exercised end to end.
`webhooks-actions` (`POST /webhooks/{webhook_id}/actions`) carries enable/disable style
transitions.

## 6. Clean up

`subscriptions-destroy` removes one event binding; `webhooks-destroy` removes the
listener. Always remove test listeners when done.
