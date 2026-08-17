---
name: Authenticate against the Kardinal ARO API and keep the token alive
description: Exchange a username and password for a Kardinal JWT access token, complete MFA or SSO where configured, refresh before the one-hour expiry, and verify tokens offline against the published signing key. Use before any other Kardinal call.
api: openapi/kardinal-aro-openapi-original.yml
generated: '2026-08-17'
method: generated
source: Grounded in operationIds verified against openapi/kardinal-aro-openapi-original.yml and https://developers.kardinal.ai/guides/authentication
operations:
  - postLogin
  - postLoginOTP
  - postLoginRefresh
  - postLoginWithAzureSSO
  - postLoginWithGoogleSSO
  - getLoginMethods
  - requestNewLoginOTPCode
  - getPublicKey
---

# Authenticate and stay authenticated

Kardinal issues **short-lived JWTs, not API keys**. There is no dashboard where you generate a long-lived credential — a username and password are exchanged for tokens, and those tokens expire in an hour.

## Know which environment you are on

Credentials are bound to one environment host (`https://<env>.kardinal.ai`). A sandbox token fails authentication against production rather than silently reading the wrong data. Store sandbox and production credentials as **separate secrets** scoped to separate deployments, and point your refresh logic at the same host the original login used.

Access is provisioned by invitation — there is no self-serve sign-up. If you have no credentials, that is not a bug to debug; contact `api@kardinal.ai` or the Account Executive.

## Step 1 — Discover the login method (`getLoginMethods`)

`GET /login/methods` tells you which methods are available for a given username, so you can branch to password, Azure SSO or Google SSO instead of guessing.

## Step 2 — Log in

**Password** (`postLogin`) — `POST /login` with `{"username", "password"}`.

- If MFA is **not** configured you get `access_token` and `refresh_token` directly.
- If MFA **is** configured you get an **OTP token** instead. Redeem it with `postLoginOTP` (`POST /login/otp`) to get the real tokens. `requestNewLoginOTPCode` (`POST /login/resendOTP`) resends the code — it is rate limited and returns **429** when called too often, and that 429 has **no structured JSON body**, so do not try to parse it as an `EnvelopedErrors`.

**SSO** — `postLoginWithAzureSSO` (`POST /login/sso/azure`) or `postLoginWithGoogleSSO` (`POST /login/sso/google`).

## Step 3 — Send the token

```
Authorization: Bearer <access_token>
```

Spelling matters: use the standard `Authorization`. Some older Kardinal examples show `Authorisation`, which the API does not accept.

## Step 4 — Refresh before you expire (`postLoginRefresh`)

The `access_token` is valid for **one hour**. `POST /login/refresh` with the `refresh_token` returns a fresh `access_token` without re-sending the password.

Refresh **proactively on a ~50-minute timer.** Do not wait for a request to fail — the short lifetime is a security property, not an error condition, and a reactive refresh turns every hour into a user-visible failure.

## Step 5 — Verify tokens offline (`getPublicKey`)

`GET /public_key` is **unauthenticated** and returns the signing key as a JWK (or PEM via `Accept: text/plain`). Kardinal signs with **ES384** on an **EC P-384** key, so you can verify tokens locally instead of calling back on every request.

This is also the one endpoint you can call before you have credentials — use it as a reachability smoke test for an environment host.

## Token classes

Five distinct bearer token types appear in the spec; do not use one where another is expected:

| Token | What it is for |
|---|---|
| `access_token` | Every business operation. One hour. |
| `refresh_token` | Only `postLoginRefresh`. |
| `otp_token` | Interim token from `postLogin` when MFA is on; redeemed at `postLoginOTP`. |
| `password_token` | The password-reset flow, redeemed at `resetPassword`. |
| `gdpr_token` | GDPR-scoped access. |

## Failure modes

| Symptom | Cause | Fix |
|---|---|---|
| Works fine, then everything 401s after ~1 hour | `access_token` expired | Refresh proactively, before the hour |
| Every request fails, including the first | Wrong credentials, or wrong `<env>` host | Re-check both — a token from the wrong environment fails auth |
| 401 with no `Authorization` sent | Header missing or misspelled | Exactly `Authorization: Bearer <token>` |
| 429 on OTP resend | Resend called too often | Back off; there is no `Retry-After` header and no JSON body to read |

Authentication errors use `code: NOT_AUTHENTICATED` (401) and `code: NOT_ALLOWED` (403). Note that a 403 appears under both "Forbidden" and "Unauthorized" response names in the API reference depending on the endpoint — both behave identically. Match on `code`.

**There is no self-service revocation endpoint.** If a password or refresh token is compromised, contact `api@kardinal.ai` or the Account Executive to have access reset. Never log the `Authorization` header value; log the response body instead when debugging a 401.
