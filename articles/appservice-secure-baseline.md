---
layout: article
title: Secure App Service Baseline Hardening
topic: How-To
category: how-to
level: Advanced
diagram: /images/howto-appservice-secure-baseline.svg
summary: A practical hardening checklist for Azure App Service—identity, TLS, secrets, SCM access, networking, slots, and diagnostics—aligned with Entra and Key Vault best practices.
author: Todd Williamsen
date: 2025-10-25
description: Advanced secure baseline for Azure App Service—HTTPS, TLS, managed identity, Key Vault references, access restrictions, private endpoints, and logging.
permalink: /articles/appservice-secure-baseline/
---

A Web App that “works” is not the same as a Web App you would defend in an incident review. This baseline bundles the controls from earlier articles into one hardening pass you can encode in Bicep later.

<figure>
  <img src="{{ '/images/howto-appservice-secure-baseline.svg' | relative_url }}" alt="Identity, platform, and network-plus-logging layers for App Service hardening">
  <figcaption>Figure 1. Identity, platform, and network/logging—each layer has a job.</figcaption>
</figure>

## What you need before you start

- An App Service app on a paid SKU when you need slots/VNet features
- Entra app registrations designed correctly (UI vs API)
- Key Vault + ability to assign RBAC
- About **60 minutes** for a first production-like pass

## Step 1 — Identity and secrets

1. **Entra sign-in** for user-facing apps (Easy Auth or MSAL)—no local admin basic auth for the site.
2. **Separate app registrations** for UI and API; least-privilege scopes.
3. **System- or user-assigned MI** on the Web App.
4. Secrets only via **Key Vault references** (or MSI to resources directly).
5. **No secrets in git**, parameter files, or slot settings screenshots.
6. Prefer **federated credentials** for GitHub deploy; no standing client secrets.

## Step 2 — Platform TLS and SCM

```bash
az webapp update -g <rg> -n <app> --https-only true
az webapp config set -g <rg> -n <app> --min-tls-version 1.2
az webapp config set -g <rg> -n <app> --ftps-state Disabled
```

Also:

| Control | Setting |
| --- | --- |
| HTTP/2 | On when compatible |
| Remote debugging | Off in prod |
| SCM basic auth | Prefer identity-based / disable basic publishing creds |
| Client certs | Optional mutual TLS for B2B APIs |

## Step 3 — Networking

1. **Access restrictions**: allow corporate egress / Front Door / Application Gateway IPs; deny everyone else when the app is not public.
2. **Private endpoint** for the Web App when it should not be on the public internet.
3. **VNet integration** for outbound to private Key Vault/SQL.
4. If using Easy Auth + private endpoints, validate redirect hostnames still match custom domains.

## Step 4 — Deployment safety

1. **Staging slot** + sticky settings + swap.
2. Register **all** Entra redirect URIs for prod and staging hosts.
3. CI via **OIDC**; deploy to staging first.
4. Disable FTP; use zip/run-from-package.

## Step 5 — Logging and detection

1. Diagnostic settings → Log Analytics: AppServiceHTTPLogs, AppServiceAuditLogs, AppServicePlatformLogs.
2. Application Insights with connection string in Key Vault reference.
3. Entra **sign-in logs** for the enterprise app; alert on unexpected failures.
4. Key Vault audit logs for secret access anomalies.

## Step 6 — Quick verification checklist

| Check | Pass? |
| --- | --- |
| HTTPS only + TLS 1.2+ | |
| FTPS disabled | |
| MI on; Key Vault refs green | |
| No client secret in GitHub | |
| UI/API separate regs; aud validated | |
| CA policy on enterprise app (if licensed) | |
| Access restrictions or private endpoint | |
| Diagnostics to Log Analytics | |
| Staging slot sticky settings | |

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| Key Vault red X after private endpoint | DNS/VNet integration missing |
| Auth breaks on custom domain | Redirect URI not updated |
| Still public SCM | Publishing credentials left enabled |
| Secret regenerated into pipeline | Old deploy template |

## What you should remember

1. **Identity first**: Entra users, MI for Azure resources.
2. **No long-lived secrets** in config or git—KV + federation.
3. **HTTPS/TLS/FTPS/SCM** hardened by default.
4. **Network least exposure** + slots for safe deploy.
5. **Logs that someone will actually alert on**.

Encode this baseline in infrastructure-as-code so the next app does not start from a permissive portal wizard.
