---
layout: article
title: Web App Plus API with Separate App Registrations
topic: How-To
category: how-to
level: Intermediate
diagram: /images/howto-webapp-api-separate-app-regs.svg
summary: Register a client app and an API app separately, expose a scope, grant consent, and request tokens for the API audience—not the client ID.
author: Todd Williamsen
date: 2025-10-13
description: Intermediate how-to for separate Entra app registrations for a web UI and API—App ID URI, scopes, authorized client apps, and audience validation.
permalink: /articles/webapp-api-separate-app-regs/
---

One registration for “the whole product” mixes redirect URIs, secrets, and API audiences until debugging is guesswork. **Best practice:** a **client** registration for the UI and an **API** registration that exposes scopes. The access token’s **audience** is the API.

<figure>
  <img src="{{ '/images/howto-webapp-api-separate-app-regs.svg' | relative_url }}" alt="Browser UI client app registration requests scopes from a separate API app registration">
  <figcaption>Figure 1. Client gets tokens; API validates audience and scopes.</figcaption>
</figure>

## What you need before you start

- Rights to create two app registrations
- A web UI and a protected API (App Service or local)
- About **35–45 minutes**
- Single-tenant for this lab

## Step 1 — Create the API registration

1. **App registrations** → **New registration** → `api-contoso-lab`.
2. Single tenant. No redirect URI required for a pure API.
3. **Expose an API** → set **Application ID URI** (accept `api://<api-client-id>` or a verified domain URI).
4. **Add a scope**:
   - Name: `access_as_user`
   - Who can consent: Admins and users (or Admins only for sensitive APIs)
   - Admin/user consent display names: clear English
5. Copy the full scope: `api://<api-client-id>/access_as_user`.

Optional: define **App roles** for app-only callers (daemon apps).

## Step 2 — Create the client (UI) registration

1. New registration → `app-contoso-ui-lab`.
2. Single tenant.
3. Redirect URI: Web → your UI callback (`https://app.../signin-oidc` or Easy Auth callback).
4. **API permissions** → **Add a permission** → **My APIs** → `api-contoso-lab` → delegated `access_as_user`.
5. **Grant admin consent** if your tenant requires it.
6. Under the API registration **Expose an API** → **Authorized client applications**, add the UI client ID and pre-authorize the scope to reduce consent prompts (still least privilege).

## Step 3 — Client acquires a token for the API

Authority:

```text
https://login.microsoftonline.com/<tenant-id>/v2.0
```

Scopes requested by the UI (example MSAL):

```text
api://<api-client-id>/access_as_user
```

Do **not** request only `User.Read` and expect the API to accept that token. Graph tokens have Graph audiences.

## Step 4 — API validates audience and issuer

Validate JWT:

| Check | Expected |
| --- | --- |
| Signature | Entra JWKS |
| `iss` | `https://login.microsoftonline.com/<tenant-id>/v2.0` (v2) |
| `aud` | API client ID or App ID URI (match your config) |
| `scp` | contains `access_as_user` (delegated) |

ASP.NET example settings shape:

```text
AzureAd:Instance = https://login.microsoftonline.com/
AzureAd:TenantId = <tenant-id>
AzureAd:ClientId = <api-client-id>
```

## Step 5 — Keep credentials straight

| App | Secrets? |
| --- | --- |
| SPA public client | No client secret |
| Confidential web UI | Cert / secret / MI as appropriate |
| API | Validates tokens; may have its own outbound creds |

Never put the API’s client secret in the browser.

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| `aud` is Graph / wrong GUID | Client requested wrong scope |
| 401 invalid audience | API ClientId config points at UI app |
| Consent loop | Scope not admin-consented; preauth missing |
| Works locally only | Redirect URI not registered for prod host |

## What you should remember

1. **Two registrations:** UI client + API resource.
2. **Scope string is the contract** for delegated access.
3. **Audience must be the API**, not the UI client ID.
4. **Pre-authorize known clients** when appropriate.
5. **Least privilege scopes**—one purpose per scope if you grow.

Next: call Microsoft Graph from App Service with managed identity instead of a client secret.
