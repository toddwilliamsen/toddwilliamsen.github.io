---
layout: article
title: On-Behalf-Of Flow for a Middle-Tier API
topic: How-To
category: how-to
level: Advanced
diagram: /images/howto-on-behalf-of-middle-tier.svg
summary: Exchange a user's access token at your middle-tier API for a new Entra token to call Graph or a second API while preserving the user context—OBO done safely.
author: Todd Williamsen
date: 2025-10-21
description: Advanced walkthrough of the OAuth 2.0 On-Behalf-Of flow with Microsoft Entra—middle-tier API, scopes, client credentials, and consent.
permalink: /articles/on-behalf-of-middle-tier/
---

Your browser calls **API A**. API A must call **Microsoft Graph** or **API B** **as that user**. App-only MI is the wrong tool—you need the **On-Behalf-Of (OBO)** grant: user token in, downstream token out.

<figure>
  <img src="{{ '/images/howto-on-behalf-of-middle-tier.svg' | relative_url }}" alt="Client user token to middle-tier API, OBO exchange at Entra, downstream API call">
  <figcaption>Figure 1. User context flows through the middle tier via OBO—not a shared app-only identity.</figcaption>
</figure>

## What you need before you start

- Three logical apps: client, middle-tier API, downstream resource (Graph or API B)
- Middle tier is a **confidential client** (secret, cert, or federated cred—prefer cert/federation)
- Admin consent for delegated permissions the middle tier will request
- About **45–60 minutes**

## Step 1 — Registrations and permissions

1. **Client** (SPA or web): requests scope on **middle-tier API** (`api://middle/access_as_user`).
2. **Middle-tier API**: exposes `access_as_user`; also has **API permissions** for downstream delegated scopes (e.g. Graph `Mail.Read` or `api://downstream/access_as_user`).
3. **Downstream**: exposes its scope or is Graph.
4. Grant **admin consent** for middle-tier → downstream delegated permissions.
5. Pre-authorize the client on the middle-tier API scope.

Middle tier needs a credential to perform OBO (certificate preferred).

## Step 2 — Client calls middle tier

Client acquires an access token **audience = middle-tier API** and calls:

```http
GET /api/profile
Authorization: Bearer <user-token-for-middle>
```

Middle tier validates that token (iss, aud, scp) **before** any OBO call.

## Step 3 — Middle tier performs OBO

Using MSAL.NET / MSAL node confidential client:

1. Build `userAssertion` from the inbound bearer token.
2. Request downstream scopes, e.g. `https://graph.microsoft.com/Mail.Read` or `api://downstream/access_as_user`.
3. MSAL uses grant `urn:ietf:params:oauth:grant-type:jwt-bearer` (OBO) under the hood.

Pseudo-shape:

```csharp
var result = await app.AcquireTokenOnBehalfOf(
  scopes: new[] { "https://graph.microsoft.com/Mail.Read" },
  userAssertion: new UserAssertion(incomingToken))
  .ExecuteAsync();
```

Call Graph/API B with `result.AccessToken`.

## Step 4 — Consent and CA realities

- If the user never consented to downstream scopes, OBO fails until consent/admin consent is complete.
- Conditional Access that requires interactive MFA may block pure OBO—design CA exclusions carefully or use step-up patterns.
- Do not cache downstream tokens longer than MSAL guidance; bind cache to user.

## Step 5 — Hardening

| Control | Practice |
| --- | --- |
| Credential | Cert or federation for middle tier |
| Validation | Always validate inbound token first |
| Scopes | Downstream least privilege |
| Logging | Correlate user oid/tid; never log raw tokens |
| Network | Middle tier private where possible |

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| AADSTS50013 / assertion errors | Inbound token aud is not the middle tier |
| Consent required | Downstream delegated perm not consented |
| Used client credentials instead | App-only token loses user context |
| Secret in SPA | OBO must run on confidential middle tier |

## What you should remember

1. **OBO preserves the user**—MI/app-only does not.
2. **Inbound aud must be the middle tier**.
3. **Middle tier is confidential** with least-privilege downstream scopes.
4. **Validate then exchange**.
5. **CA and consent** are part of the design, not afterthoughts.

Next: multi-tenant SaaS registration patterns.
