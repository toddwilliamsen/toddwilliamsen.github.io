---
layout: article
title: Protect an API with Entra Token Validation
topic: How-To
category: how-to
level: Intermediate
diagram: /images/howto-protect-api-entra-tokens.svg
summary: Accept only valid Entra access tokens—verify signature, issuer, audience, then scopes or app roles before your API business logic runs.
author: Todd Williamsen
date: 2025-10-16
description: Intermediate how-to for protecting an API with Microsoft Entra JWT validation—JWKS, issuer, audience, scopes, and common 401 causes.
permalink: /articles/protect-api-entra-tokens/
---

A public HTTP API without token validation is a data breach waiting on a scanner. With Entra, clients send `Authorization: Bearer <access_token>`. Your API must **validate** that JWT—not merely decode it.

<figure>
  <img src="{{ '/images/howto-protect-api-entra-tokens.svg' | relative_url }}" alt="Client bearer token validated by API using Entra metadata and JWKS">
  <figcaption>Figure 1. Signature, issuer, audience, then scope/role—fail closed.</figcaption>
</figure>

## What you need before you start

- API app registration with App ID URI and scopes/roles
- Clients that request the correct scope
- Middleware or library that validates JWTs (do not hand-roll crypto)
- About **30 minutes**

## Step 1 — Know which token you need

| Token | Use |
| --- | --- |
| ID token | Who signed in (UI session); **not** for API authZ by default |
| Access token | Sent to APIs; **aud** must be your API |

Reject Graph-audience tokens at your API.

## Step 2 — Configure authority metadata

OpenID config (v2):

```text
https://login.microsoftonline.com/<tenant-id>/v2.0/.well-known/openid-configuration
```

Libraries fetch JWKS from the `jwks_uri` and cache keys.

ASP.NET (shape):

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
  .AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAd"));
```

`AzureAd:ClientId` = **API** application ID. TenantId = your tenant (or org for multi-tenant with care).

## Step 3 — Validate the critical claims

| Claim | Check |
| --- | --- |
| `iss` | Matches your tenant issuer (or approved multi-tenant pattern) |
| `aud` | Equals API client ID or App ID URI—**configure one and stick to it** |
| `exp` / `nbf` | Not expired; allow small clock skew |
| `scp` | Delegated: required scope present |
| `roles` | App-only: required app role present |

Example policy: require scope `access_as_user` **or** role `Api.Read`.

## Step 4 — Wire authorization in the API

```csharp
[Authorize]
[HttpGet("orders")]
public IActionResult List() => Ok();

[Authorize(Roles = "Admin")] // maps from roles claim when configured
[HttpPost("orders")]
public IActionResult Create() => Ok();
```

For scopes, use scope-based policies (`scp`/`http://schemas.microsoft.com/identity/claims/scope`).

## Step 5 — Test with a real token

1. Acquire a token for `api://<api-id>/access_as_user` via MSAL or a test harness.
2. Call the API with the bearer header.
3. Decode at jwt.ms **only with lab tokens**—confirm `aud` and `scp`.
4. Flip `aud` expectation and confirm you get 401.

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| 401 `invalid_token` aud | Client used UI client ID as audience |
| 401 issuer | Wrong tenant; v1 vs v2 issuer mismatch |
| Scope missing | Client did not request API scope |
| Accepts any JWT | Validation disabled; “decode only” bug |
| Works with ID token | API incorrectly configured to accept ID tokens |

## What you should remember

1. **Validate signature + iss + aud**—always.
2. **Access token audience = API**.
3. **Authorize on `scp` or `roles`** after authentication.
4. **Use maintained libraries**, not custom JWT parsers.
5. **Fail closed**—no anonymous fallback on protected routes.

Next: decide Easy Auth at the platform vs MSAL inside your code.
