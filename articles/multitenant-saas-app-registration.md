---
layout: article
title: Multi-Tenant SaaS App Registration Design
topic: How-To
category: how-to
level: Advanced
diagram: /images/howto-multitenant-saas-app-registration.svg
summary: Design a multi-tenant Entra app for SaaS—publisher verification, admin consent, per-tenant service principals, and issuer/tenant allowlisting in your API.
author: Todd Williamsen
date: 2025-10-22
description: Advanced guide to multi-tenant Microsoft Entra app registration for SaaS—account types, consent, tenant allowlists, and token issuer validation.
permalink: /articles/multitenant-saas-app-registration/
---

Single-tenant apps trust one directory. **Multi-tenant SaaS** apps let other organizations sign in after **consent**, creating a **service principal** in each customer tenant. Mis-handle issuer validation and you accept tokens from the wrong tenants—or the whole world.

<figure>
  <img src="{{ '/images/howto-multitenant-saas-app-registration.svg' | relative_url }}" alt="Publisher tenant multi-tenant app consented into customer tenants with API tenant checks">
  <figcaption>Figure 1. One registration, many customer service principals—validate tid and iss.</figcaption>
</figure>

## What you need before you start

- A publisher tenant where the app lives
- Willingness to support admin consent UX
- A tenant allowlist or onboarding workflow
- About **45 minutes** for the registration design (longer for product UX)

## Step 1 — Registration settings

1. App registration → **Accounts in any organizational directory** (multi-tenant). Avoid “personal Microsoft accounts” unless you truly need them.
2. Keep **separate** client and API registrations even in SaaS.
3. Redirect URIs: only your SaaS HTTPS endpoints.
4. Publisher verification (Microsoft verified publisher) to reduce consent warnings.
5. Least-privilege Graph permissions; prefer your own API scopes over broad Graph.

## Step 2 — Consent strategy

| Approach | Use |
| --- | --- |
| User consent | Low-privilege delegated scopes only |
| Admin consent URL | Required for privileged permissions |
| Customer-create SP | Some enterprises deploy via their own automation |

Admin consent URL shape:

```text
https://login.microsoftonline.com/organizations/v2.0/adminconsent
  ?client_id=<client-id>
  &redirect_uri=<encoded>
  &scope=https://graph.microsoft.com/.default
```

Prefer scoping consent to **your API scopes** plus minimal Graph.

## Step 3 — Token validation for multi-tenant APIs

Do **not** set `TenantId = common` and accept every issuer blindly.

Recommended pattern:

1. Metadata: use multi-tenant capable middleware carefully.
2. Validate signature against Entra.
3. Validate `iss` matches `https://login.microsoftonline.com/{tid}/v2.0`.
4. Enforce **`tid` allowlist** (customers you onboarded).
5. Optionally map `tid` → customer record before authorization.

Reject tokens from tenants not in your customer table—even if Entra signed them.

## Step 4 — Per-tenant enterprise apps

After consent, each customer tenant has an enterprise app you do not fully control. Document:

- Required app roles / groups in the customer tenant
- Conditional Access expectations (customers apply their own CA)
- Offboarding: customer removes consent; you revoke allowlist

## Step 5 — Operational concerns

| Topic | Practice |
| --- | --- |
| Secrets | Federated creds / certs; rotate per environment |
| National clouds | Separate registrations and endpoints |
| Testing | Use a second test tenant; never dogfood only in publisher |
| Break-glass | Publisher admin accounts with CA hardened |

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| AADSTS50020 user account mismatch | Wrong account type / personal account |
| Consent scary warnings | Unverified publisher; overbroad scopes |
| Accepting any tenant | No tid allowlist |
| Single-tenant issuer hard-coded | Forgot multi-tenant `iss` pattern |

## What you should remember

1. **Multi-tenant ≠ trust all tenants**—allowlist `tid`.
2. **Separate UI and API regs**; least privilege scopes.
3. **Admin consent** for privileged permissions.
4. **Validate iss + aud + tid**.
5. **Publisher verification** and clean redirect URIs reduce friction and risk.

Next: workload identity on Azure Container Apps.
