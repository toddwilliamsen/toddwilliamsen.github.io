---
layout: article
title: Register an Entra App the Right Way
topic: How-To
category: how-to
level: Beginner
diagram: /images/howto-entra-app-registration-basics.svg
summary: Create your first Microsoft Entra app registration with the right account type, redirect URIs, and zero secrets in git—plus what Object ID vs Client ID mean.
author: Todd Williamsen
date: 2025-10-08
description: Beginner walkthrough to register an Entra ID application correctly—single-tenant vs multi-tenant, redirect URIs, API permissions, secrets vs certificates, and enterprise app basics.
permalink: /articles/entra-app-registration-basics/
---

An **app registration** is how Entra ID knows your application. You get a **Application (client) ID**, a **Directory (tenant) ID**, and (optionally) credentials. The tenant also gets an **enterprise application** (service principal)—the local instance you assign users to and apply Conditional Access against.

This walkthrough is for a first single-tenant web app. Portal first; CLI where it saves time.

<figure>
  <img src="{{ '/images/howto-entra-app-registration-basics.svg' | relative_url }}" alt="Entra app registration with client ID, redirect URIs, and enterprise app service principal">
  <figcaption>Figure 1. Registration is the blueprint; the enterprise app is the tenant-local instance.</figcaption>
</figure>

## What you need before you start

- Entra ID permissions: **Application Developer** (or higher) to create app registrations
- A clear hostname for redirects (local `https://localhost:5001` and/or an App Service URL)
- About **15–20 minutes**
- Decision: **single-tenant** for internal apps unless you truly need multi-tenant SaaS

## Step 1 — Create the app registration

### Portal

1. Open **Microsoft Entra ID** → **App registrations** → **New registration**.
2. Name: `app-contoso-web-lab` (something you will recognize later).
3. Supported account types: **Accounts in this organizational directory only** (single tenant).
4. Redirect URI: platform **Web**, URI `https://localhost:5001/signin-oidc` for a first ASP.NET Core lab (change later for App Service).
5. **Register**.

Copy three values from **Overview**:

| Value | Use |
| --- | --- |
| Application (client) ID | Configure in your app |
| Directory (tenant) ID | Authority / issuer |
| Object ID | Graph / admin operations on *this* app object |

### CLI

```bash
az ad app create \
  --display-name "app-contoso-web-lab" \
  --sign-in-audience AzureADMyOrg \
  --web-redirect-uris "https://localhost:5001/signin-oidc"
```

## Step 2 — Set redirect URIs correctly

Wrong redirect URIs are the #1 beginner failure. Rules:

1. Scheme + host + path must match **exactly** (trailing slash matters).
2. Use **Web** for confidential server apps; **SPA** for browser-only MSAL.js; **Mobile and desktop** for public native clients.
3. Prefer HTTPS. Localhost HTTP is allowed only for development in some platforms—do not ship it.
4. For App Service Easy Auth, the URI is typically `https://<app>.azurewebsites.net/.auth/login/aad/callback`.

Portal: **Authentication** → add platform / redirect URIs → **Save**.

## Step 3 — API permissions (least privilege)

1. Open **API permissions**.
2. Keep **Microsoft Graph** → **User.Read** (delegated) if you only need the signed-in profile.
3. Do **not** add `Directory.ReadWrite.All` “just in case.”
4. Click **Grant admin consent** only if your org requires it and you understand the permission.

For your own API later, you will **Expose an API** on a *separate* registration and request *that* scope from the client. Do not conflate UI and API into one registration if you can avoid it.

## Step 4 — Credentials (prefer none yet)

For a user-sign-in web app using the authorization code flow with a confidential client, you eventually need a **client secret** or **certificate**. Best practice order:

1. **Managed identity** or **federated credentials** when the host is Azure or GitHub Actions
2. **Certificate** stored in Key Vault
3. **Client secret** with short expiry—never commit to git

Portal: **Certificates & secrets** → create only when your stack requires it. Store the value in Key Vault or a secret store immediately; the portal shows it once.

```bash
# Example: create a secret (lab only) — prefer cert/federation in real work
az ad app credential reset \
  --id <client-id> \
  --append \
  --display-name "lab-secret" \
  --years 1
```

## Step 5 — Find the enterprise application

1. Entra ID → **Enterprise applications**.
2. Search for the same display name.
3. Here you assign users/groups, review sign-in logs, and target Conditional Access.

If **Assignment required** is Yes, users must be assigned or they get “access denied” after a successful login—confusing until you know to look here.

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| AADSTS50011 redirect mismatch | URI not registered or wrong platform type |
| AADSTS700016 application not found | Wrong client ID or wrong tenant |
| Users cannot sign in | Assignment required + no assignment |
| Secret “lost” | Value only shown once; recreate and store properly |
| Over-permissioned Graph | Admin consented broad roles for a simple site |

## What you should remember

1. **Client ID ≠ Object ID**—configure Client ID in apps; Object ID is for directory operations.
2. **Single-tenant first** unless you are building SaaS.
3. **Redirect URIs are exact**—treat them like public contracts.
4. **Least privilege permissions**—start with `User.Read`.
5. **No secrets in git**—prefer MI / federation / Key Vault.
6. **Separate app regs** for UI vs API when you add a backend.

Next: deploy a simple App Service site, then wire Entra sign-in with a correct production redirect URI.
