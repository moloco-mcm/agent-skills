---
name: embedded-sso
description: Generate signed Embedded SSO URLs that authenticate your platform's users into Moloco MCM's embedded Campaign Manager. Use when implementing the HMAC-SHA256 signed-URL handoff (the sso-examples flow), constructing the /sso URL, or debugging signature mismatches.
---

# MCM Embedded SSO

You are helping a partner developer implement **Embedded SSO** for Moloco MCM. Embedded SSO lets a partner embed Moloco's RMP Campaign Manager inside their own product and seamlessly sign their own users into it — no separate Moloco password. The partner's backend signs a URL describing the user with a shared secret; Moloco verifies the signature, provisions or looks up the user, and loads the embedded Campaign Manager already authenticated.

This is implementable in **any language** with an HMAC-SHA256 library. The official `moloco-mcm/sso-examples` repo has reference implementations in Node, Go, Java, and Ruby.

## How it works

1. Your backend builds a signed URL for the logged-in user: gather the parameters, generate a fresh `nonce` and current Unix `timestamp`, compute the HMAC-SHA256 signature over the canonical message, and assemble `{baseUrl}/sso?<params>&signature=<sig>`.
2. Your app redirects the browser (or loads an iframe) to that `/sso` URL.
3. Moloco's `/sso` page verifies the signature, timestamp, and nonce, provisions or looks up the user, grants the requested role, issues a short-lived session token, and redirects to the embedded view at `path`.

You only build and serve the signed `/sso` URL. The token exchange happens inside the Moloco portal — there is no separate REST endpoint for you to call.

## The signed URL

`GET {baseUrl}/sso?<query params>&signature=<base64 HMAC>`

- `baseUrl` — your platform-specific portal URL, provided by your Moloco account manager. Do not hardcode a generic Moloco URL.
- `path` — the deep link inside the portal to land on after auth (see [Roles](#roles-and-targeting) for the value per role).

## Canonical signature message (v1.1.0)

This is the single most error-prone part. The signature is computed over **these 14 fields, in this exact order, joined by newline (`\n`)**. Every field is always present in the string — use an empty string `""` for any field you are not sending. Getting the order or an empty-field wrong produces a signature mismatch with no other diagnostic.

```
ad_account_id
ad_account_ids            # array joined with ";", "" if none
ad_account_title
ad_manager_account_id
ad_manager_account_title
email
external_user_id
name
nonce
path
platform_id
role                      # "" if no role
timestamp                 # Unix seconds, as a string
version                   # "1.1.0"
```

Then: `signature = base64( HMAC_SHA256( message, sso_secret ) )` using **standard** base64 (not URL-safe).

Use version `1.1.0`. (A legacy `1.0.0` exists with a different, shorter field set — do not mix the two; the field list and order differ.)

## Signing recipe (language-agnostic)

1. Collect `platform_id`, `path`, `email`, `name`, `external_user_id`, `role`, and the target — `ad_account_id` (single account), `ad_account_ids[]` (agency), or `ad_manager_account_id` (ad manager account). Set `version = "1.1.0"`.
2. `timestamp` = current Unix time in **seconds**, as a string. `nonce` = a fresh random string per request (16 alphanumeric chars is typical; any unique value works).
3. Build the canonical string = the 14 fields above joined with `\n`, using `""` for absent fields and `";"`-join for `ad_account_ids`.
4. `signature = base64(HMAC_SHA256(canonical_string, sso_secret))`.
5. URL-encode every query param and assemble `{baseUrl}/sso?...&signature=...`. Send `ad_account_ids` as repeated query params (`ad_account_ids=a&ad_account_ids=b`), even though they are `;`-joined in the signature.
6. Redirect the browser to that URL promptly — the server validates `timestamp` against a **±5-minute window** around its own clock (see [What Moloco verifies](#what-moloco-verifies)).

Reference (Node, from `moloco-mcm/sso-examples` — Go / Java / Ruby are 1:1 equivalents):

```js
const { createHmac } = require("crypto");

// fields in the exact canonical order; "" for anything absent
const message = [
  adAccountId || "",
  adAccountIds && adAccountIds.length ? adAccountIds.join(";") : "",
  adAccountTitle || "",
  adManagerAccountId || "",
  adManagerAccountTitle || "",
  email,
  externalUserId,
  name,
  nonce,
  path,
  platformId,
  role || "",
  timestamp,
  version, // "1.1.0"
].join("\n");

const signature = createHmac("sha256", ssoSecret).update(message).digest("base64");
```

## Roles and targeting

Role values: `AD_ACCOUNT_OWNER`, `AD_ACCOUNT_USER`, `AD_ACCOUNT_VIEWER`, `AD_ACCOUNT_AGENCY`, `AD_MANAGER_ACCOUNT_OWNER`, `AD_MANAGER_ACCOUNT_USER`. To grant no role, send `role` as an empty string `""` (the reference implementations use the `role || ""` fallback).

| Scenario | Role(s) | Target field | `path` |
|---|---|---|---|
| Single ad account | `AD_ACCOUNT_OWNER` / `_USER` / `_VIEWER` | `ad_account_id` | `/embed/sponsored-ads/cm/a/{adAccountId}` |
| Agency (multiple accounts) | `AD_ACCOUNT_AGENCY` | `ad_account_ids` | `/embed/sponsored-ads` |
| Ad manager account | `AD_MANAGER_ACCOUNT_OWNER` / `_USER` | `ad_manager_account_id` | `/embed/sponsored-ads/cm/ama/{adManagerAccountId}` |

Field rules:
- `ad_account_id` and `ad_account_ids` cannot be used together.
- Ad manager roles use `ad_manager_account_id` and must not send `ad_account_id` / `ad_account_ids`.
- For `AD_ACCOUNT_AGENCY`, all listed ad accounts must already exist. A server-side limit applies to the number of accounts in `ad_account_ids` — confirm the current cap with your account manager.

## Display options

`config:color_mode` (`light` | `dark` | `useDeviceSetting`) and `config:language` (`en` | `ko`) can be added to the URL. **These are not part of the signed message** — including them in the signature breaks it.

## Configuration

- **SSO secret** — the per-platform HMAC key. Managed in Campaign Manager → Admin → Credential Management; only the **Platform Owner** role can generate/view/change it; **one secret per platform**. Store it in a vault, never commit it, rotate periodically.
- **`platform_id`** — your unique platform ID, provided by Moloco. Send it consistently; the value is signed as-sent.
- **`baseUrl`** — your platform-specific portal URL from your account manager.
- **`external_user_id`** — your stable identifier for the user; it becomes the delegated-user key on Moloco's side.

## What Moloco verifies

1. Required fields and role/version rules.
2. **Timestamp freshness** — rejected if more than ±5 minutes from Moloco server time.
3. **Signature** — recomputed from the canonical message and compared; mismatch → permission denied.
4. **Nonce** — anti-replay; reusing a nonce is rejected.

## Common pitfalls

1. **Wrong field order or wrong version's field set** → silent signature mismatch. Copy the canonical order exactly; don't mix v1.0.0 and v1.1.0.
2. **Omitting absent fields instead of sending `""`** — every one of the 14 positions must be present in the signed string; a skipped line shifts everything.
3. **`ad_account_ids` formatting** — `;`-joined in the signature, but repeated `ad_account_ids=` params in the URL.
4. **`config:*` in the signature** — they are display-only; never include them in the signed message.
5. **Stale or pre-generated URLs** — the timestamp must be within ±5 minutes; generate the URL at request time.
6. **Nonce reuse** — generate a fresh nonce every request.
7. **URL-safe base64** — use standard base64; the server expects it.
8. **`platform_id` casing** — send it the same way every time; it is signed as-sent.

## Sources

This skill is derived from Moloco's public SSO examples and documentation. The `sso-examples` repo is the canonical reference — match its canonical-message field order exactly. If any of these change, re-check the affected section here.

- Reference implementations (Node / Go / Java / Ruby): <https://github.com/moloco-mcm/sso-examples>
- SSO secret & credential management: <https://mcm-docs.moloco.com/docs/api-and-sso-credential-management>
