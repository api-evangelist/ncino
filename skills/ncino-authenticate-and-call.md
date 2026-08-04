---
name: Authenticate against the nCino Mortgage API
description: Mint an OAuth 2.0 client-credentials token, send it correctly, pick an API version, and handle the 401/403 failure modes that dominate this API.
api: openapi/ncino-mortgage-openapi.yml
operations: [authentication-create, authentication-show, authentication-actions]
---

# Authenticate against the nCino Mortgage API

Every nCino Mortgage call is bearer-token authenticated. Tokens are short-lived, so
mint one at the start of each batch of work rather than caching it across a session.

## 1. Mint a token

Call `authentication-create` — `POST https://api.ncinomortgage.com/oauth/token` — with:

- `grant_type: client_credentials`
- `client_id`: the API Key from API Settings
- `client_secret`: the API Secret from API Settings

The response carries `expires_in: 900`. The documentation says "five minutes"; treat
900 seconds as the contract and re-mint per batch either way. The granted scope is
`external`.

Never put the API Secret in a prompt, a log line, or a tool argument that is echoed
back to a user. nCino's own guidance is that the secret is not to be shared with
anyone outside the organization, including nCino.

## 2. Send it

Set `Authorization: Bearer <access_token>` on every subsequent request. Optionally set
`X-Api-Version: 1.0`; when omitted, the version configured on the API credential is
used. Every response echoes `X-Api-Version` and `X-Api-Supported-Versions` — read them
rather than assuming.

## 3. Inspect or revoke

`authentication-show` (`GET /oauth/token`) returns the current token's context.
`authentication-actions` (`POST /oauth/token/actions`) performs token actions.

## Failure modes

- **401** — the token is invalid or has expired. Re-mint; do not retry the same token.
- **403 "The client does not have access to the requested resource"** — the endpoint
  is not toggled on for this credential. This is a configuration change in API
  Settings, not something to retry or work around.
- **403 "The client is not authorized to perform this action"** — the underlying
  workflow is disabled in the tenant. Stop and tell the operator to contact support.
- **403 "This feature requires additional enablement"** — the capability needs a CSM.

Full envelope and remediation detail: `errors/ncino-problem-types.yml`.
