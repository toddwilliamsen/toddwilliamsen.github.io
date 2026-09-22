---
layout: article
title: SPA Plus API with Entra PKCE
topic: How-To
category: how-to
level: Advanced
diagram: /images/howto-spa-api-entra-pkce.svg
summary: Build a public SPA client with MSAL.js auth code + PKCE, request API scopes, and validate access tokens on a separate API registration—no client secret in the browser.
author: Todd Williamsen
date: 2025-10-20
description: Advanced how-to for SPA and API with Microsoft Entra PKCE—SPA platform registration, MSAL.js, API scopes, CORS, and JWT validation.
permalink: /articles/spa-api-entra-pkce/
---

Browser apps cannot keep a client secret. Treat the SPA as a **public client**: **authorization code flow with PKCE**, separate **API** registration, and **audience validation** on the API. MSAL.js handles the hard parts if you configure platforms and redirects correctly.

<figure>
  <img src="{{ '/images/howto-spa-api-entra-pkce.svg' | relative_url }}" alt="SPA uses MSAL.js PKCE with Entra then calls API with access token">
  <figcaption>Figure 1. Public SPA + confidential API boundary; secret never ships to JavaScript.</figcaption>
</figure>

## What you need before you start

- Two app registrations (SPA client + API)
- SPA hosting (Static Web Apps, App Service, storage static site)
- API that can validate JWTs
- About **45–60 minutes**

## Step 1 — API registration

1. Register `api-spa-lab` (single tenant).
2. **Expose an API**: App ID URI `api://<api-client-id>`.
3. Scope: `access_as_user`.
4. Optional app roles for daemon callers (not used by the SPA).

## Step 2 — SPA registration

1. Register `spa-contoso-lab`.
2. **Authentication** → **Add a platform** → **Single-page application**.
3. Redirect URI: `https://localhost:5173` and production origin, e.g. `https://app.contoso.com`.
4. Front-channel logout URL if you use it.
5. Do **not** create a client secret.
6. **API permissions** → My APIs → `access_as_user` → admin consent as needed.
7. On the API **Expose an API**, authorize this SPA client ID for the scope.

Disable implicit grant flows; PKCE auth code is the modern path.

## Step 3 — MSAL.js configuration

```javascript
const msalConfig = {
  auth: {
    clientId: "<spa-client-id>",
    authority: "https://login.microsoftonline.com/<tenant-id>",
    redirectUri: window.location.origin
  }
};
const apiRequest = {
  scopes: ["api://<api-client-id>/access_as_user"]
};
```

Login with `loginRedirect` or `loginPopup`, then `acquireTokenSilent(apiRequest)` (fallback to interactive). Attach the access token to API calls:

```http
Authorization: Bearer <access_token>
```

## Step 4 — API validation and CORS

- Validate `aud` = API client ID / App ID URI
- Validate `iss` for your tenant
- Require `scp` contains `access_as_user`
- CORS: allow only the SPA origin; do not use `*` with credentials

## Step 5 — Security hardening

| Control | Practice |
| --- | --- |
| Redirect URIs | Exact origins only; no wildcards in production |
| Token storage | Prefer MSAL memory/session patterns; understand XSS risk |
| CSP | Tighten script sources |
| Refresh | Rely on MSAL; no custom refresh secret |
| API | Never trust browser-only checks |

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| AADSTS9002326 / PKCE errors | Platform not set to SPA |
| Secret required errors | Registration treated as confidential Web |
| CORS preflight fail | API missing SPA origin |
| `aud` wrong | Requested Graph scopes only |
| Redirect mismatch | Trailing slash / localhost port drift |

## What you should remember

1. **SPA = public client + PKCE**—no secret in JS.
2. **Separate API registration** and scopes.
3. **Access token audience is the API**.
4. **CORS allowlist** the SPA origin.
5. **XSS is game over for tokens**—harden the SPA host.

Next: middle-tier APIs that call downstream APIs as the user (OBO).
