---
layout: article
title: Sign Users Into App Service with Entra
topic: How-To
category: how-to
level: Beginner
diagram: /images/howto-appservice-entra-signin.svg
summary: Wire Microsoft Entra ID sign-in to an App Service Web App using Easy Auth—correct redirect URI, client registration, and first successful login.
author: Todd Williamsen
date: 2025-10-10
description: Beginner how-to for App Service Authentication (Easy Auth) with Entra ID—app registration, redirect URI, issuer URL, and portal configuration.
permalink: /articles/appservice-entra-signin/
---

**App Service Authentication** (Easy Auth) sits in front of your code. Unauthenticated requests redirect to Entra; after login, App Service injects identity headers. You do not have to write OIDC middleware for a simple site.

Use this when you want “users must sign in” quickly. Use MSAL in code when you need fine-grained token control (covered later).

<figure>
  <img src="{{ '/images/howto-appservice-entra-signin.svg' | relative_url }}" alt="Browser hits App Service, redirects to Entra, returns to Easy Auth callback">
  <figcaption>Figure 1. Entra issues tokens; Easy Auth completes the redirect on your app hostname.</figcaption>
</figure>

## What you need before you start

- A running Web App (see the App Service from-scratch article)
- Permission to create an Entra app registration
- About **25 minutes**
- Prefer a **separate** app registration for this UI (do not reuse a random Graph tutorial app)

## Step 1 — Register the Entra app

1. Entra ID → **App registrations** → **New registration**.
2. Name: `app-web-easyauth-lab`.
3. Account type: **Single tenant**.
4. Redirect URI: platform **Web**, value:

```text
https://<your-app>.azurewebsites.net/.auth/login/aad/callback
```

5. **Register**. Copy **Application (client) ID** and **Directory (tenant) ID**.

Optional client secret: Easy Auth can use a secret for the confidential client. Create one under **Certificates & secrets**, store it somewhere safe (not git), and paste it into App Service only.

Better long-term: use a certificate or managed identity patterns where supported; for a first Easy Auth lab, a short-lived secret is common.

## Step 2 — Enable Authentication on the Web App

### Portal

1. Open the Web App → **Authentication** → **Add identity provider**.
2. Identity provider: **Microsoft**.
3. App registration type: **Provide existing** (use the app you just created).
4. Application (client) ID: paste yours.
5. Issuer URL (v2):

```text
https://login.microsoftonline.com/<tenant-id>/v2.0
```

6. Client secret: paste if you created one (or configure client secret setting name).
7. Restrict access: **Require authentication**.
8. Unauthenticated requests: **HTTP 302** find account (redirect to login).
9. **Add**.

### CLI (outline)

```bash
az webapp auth update \
  --resource-group rg-appservice-lab \
  --name <your-app> \
  --enabled true \
  --action LoginWithAzureActiveDirectory \
  --aad-client-id <client-id> \
  --aad-client-secret <secret> \
  --aad-token-issuer-url "https://login.microsoftonline.com/<tenant-id>/v2.0"
```

Exact flags vary by CLI version; the portal path is reliable for beginners.

## Step 3 — Confirm reply URL and logout

1. In the app registration → **Authentication**, ensure the callback URI is present.
2. Optional front-channel logout URL: `https://<your-app>.azurewebsites.net/.auth/logout`.
3. Save.

## Step 4 — Test sign-in

1. Open an InPrivate/Incognito window.
2. Browse `https://<your-app>.azurewebsites.net`.
3. You should bounce to Microsoft login, then return to the site.
4. In App Service → **Authentication**, confirm the provider shows healthy.

To see identity in code without MSAL, Easy Auth exposes headers such as `X-MS-CLIENT-PRINCIPAL-NAME`. For production APIs, prefer validating tokens yourself (later article).

## Step 5 — Tighten the registration

| Setting | Recommendation |
| --- | --- |
| Implicit grant ID tokens | Off for modern auth code + Easy Auth |
| Supported accounts | Single tenant unless SaaS |
| API permissions | `User.Read` only unless needed |
| Client secret expiry | Short; calendar reminder to rotate |

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| AADSTS50011 | Callback URI missing or typo |
| Infinite redirect loop | Auth enabled but wrong issuer/tenant |
| 401 after login | “Require authentication” + app not returning correctly |
| Works for you only | Enterprise app assignment required |
| Secret in repo | Publish profile or settings committed |

## What you should remember

1. **Redirect URI is the Easy Auth callback path**, not your home page.
2. **Issuer includes your tenant ID** for single-tenant apps.
3. **Easy Auth ≠ full MSAL**—great for gatekeeping; limited for complex token scenarios.
4. **No secrets in git.**
5. **Separate UI app registration** from any API registration you add later.

Next: choose app roles vs group claims for authorization inside the app.
