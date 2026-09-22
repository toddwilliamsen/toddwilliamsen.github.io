---
layout: article
title: App Service Easy Auth vs MSAL in Code
topic: How-To
category: how-to
level: Intermediate
diagram: /images/howto-easyauth-vs-msal.svg
summary: Choose platform Authentication (Easy Auth) when you need a simple gate; choose MSAL when you need token acquisition, silent refresh, OBO, or SPA PKCE control.
author: Todd Williamsen
date: 2025-10-17
description: Intermediate comparison of Azure App Service Easy Auth and MSAL-in-code for Entra sign-in, token handling, and API calls.
permalink: /articles/easyauth-vs-msal/
---

Both Easy Auth and MSAL can put Entra in front of users. They solve different layers. **Easy Auth** authenticates at the App Service front door. **MSAL** authenticates inside your process (or browser) with full OAuth control.

<figure>
  <img src="{{ '/images/howto-easyauth-vs-msal.svg' | relative_url }}" alt="Side-by-side Easy Auth platform auth versus MSAL in application code">
  <figcaption>Figure 1. Platform gate vs SDK control—pick based on token needs.</figcaption>
</figure>

## What you need before you start

- An App Service app (for Easy Auth) and/or a codebase that can host MSAL
- An Entra app registration with correct redirect URIs for the path you choose
- About **25 minutes** to map your scenario

## Easy Auth — when it shines

Use when:

- You need “must sign in” for a server-rendered site
- You want config in portal/Bicep, not middleware
- Downstream APIs are minimal or you only need identity headers

Traits:

- Redirect URI ends with `/.auth/login/aad/callback`
- Identity via `X-MS-CLIENT-PRINCIPAL*` headers / Easy Auth endpoints
- Token store and refresh are platform-managed—less flexible

## MSAL — when you need it

Use when:

- SPA with PKCE
- Calling multiple APIs with incremental consent
- On-behalf-of middle tier
- Custom token caches, step-up, or BFF patterns
- Non-App Service hosts (containers, on-prem) with the same code path

Traits:

- You own redirect URIs (`/signin-oidc`, `/auth/callback`, etc.)
- You request explicit scopes
- You validate API tokens in your API project

## Side-by-side

| Need | Easy Auth | MSAL |
| --- | --- | --- |
| Fast protect site | Yes | Possible, more code |
| Call custom API with scopes | Limited / awkward | First-class |
| SPA public client | Not ideal alone | MSAL.js |
| OBO | No | Yes |
| Infra-as-code auth | Strong | App settings + code |
| Local parity | Differs from Azure | Same libraries locally |

## Step — Pick a pattern (practical)

1. **Internal admin site, little API usage** → Easy Auth.
2. **Web UI + separate API** → MSAL (or OpenID Connect middleware) on UI; JWT bearer on API. Optional Easy Auth only if you accept platform limits.
3. **SPA + API** → MSAL.js PKCE + API validation (advanced article).
4. **Do not stack blindly**: Easy Auth + MSAL both fighting redirects causes loops. If both exist, document which layer owns login.

## Redirect URI checklist

| Mode | Register |
| --- | --- |
| Easy Auth | `https://<app>.azurewebsites.net/.auth/login/aad/callback` |
| ASP.NET OIDC | `https://<host>/signin-oidc` |
| SPA | SPA platform + exact origin redirect |

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| Redirect loop | Easy Auth + app middleware both enforcing login |
| No access token for API | Easy Auth session without API scope acquisition |
| Works in Azure only | Easy Auth not reproducible locally without emulator |
| Secret in frontend | Treating SPA as confidential client |

## What you should remember

1. **Easy Auth = platform gate**; **MSAL = token engine**.
2. **API-heavy apps usually need MSAL** (or equivalent OIDC middleware).
3. **One owner for login redirects**.
4. **Correct platform type + redirect URI** for the chosen path.
5. **Never put client secrets in SPAs**.

Next: ship safely with App Service deployment slots.
