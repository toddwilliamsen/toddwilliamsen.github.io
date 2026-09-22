---
layout: article
title: Client Secrets vs Certificates vs Federated Credentials
topic: How-To
category: how-to
level: Intermediate
diagram: /images/howto-client-secret-vs-cert-vs-federation.svg
summary: Choose the right Entra credential type—why long-lived client secrets lose, when certificates help, and why federated credentials or managed identity should be your default.
author: Todd Williamsen
date: 2025-10-15
description: Intermediate comparison of Entra ID client secrets, certificates, and federated credentials for apps and CI, with practical selection guidance.
permalink: /articles/client-secret-vs-cert-vs-federation/
---

Every confidential client eventually asks: “How does this process prove it is the app?” Entra offers **client secrets**, **certificates**, and **federated credentials**. Pick in this order when you can: **managed identity / federation → certificate → short-lived secret**.

<figure>
  <img src="{{ '/images/howto-client-secret-vs-cert-vs-federation.svg' | relative_url }}" alt="Three columns comparing client secrets, certificates, and federated credentials">
  <figcaption>Figure 1. Standing secrets are liabilities; federation removes them.</figcaption>
</figure>

## What you need before you start

- An app registration you control
- Clarity on where the workload runs (Azure, GitHub Actions, on-prem)
- About **20–30 minutes** to configure one preferred path

## Option A — Client secret (last resort)

Portal: **Certificates & secrets** → **New client secret**.

Pros: simple samples. Cons: string that can be copied; often lands in git, screenshots, and pipeline variables forever.

Rules if you must:

1. Max short lifetime (days/weeks for labs; policy-limited in enterprise).
2. Store only in Key Vault or a secret store.
3. Dual secrets during rotation.
4. Alert on near-expiry.
5. Never commit to source control.

```bash
az ad app credential reset --id <client-id> --append --display-name rotate --years 1
```

## Option B — Certificate

You register the **public** cert; the app holds the **private** key.

Pros: private key need not travel as a password string; fits Key Vault + MI download patterns. Cons: you still manage issuance, renewal, and key protection.

Portal: **Certificates & secrets** → **Upload certificate**.

```bash
az ad app credential reset \
  --id <client-id> \
  --cert "@./public.cer" \
  --append
```

Keep private keys in Key Vault (HSM-backed when warranted). Apps use assertion-based client credentials (MSAL supports cert auth).

## Option C — Federated credentials (preferred for CI and many cloud cases)

Entra trusts an **external issuer** (GitHub OIDC, Kubernetes, etc.). The workload presents a short-lived token; Entra exchanges it—**no Entra client secret**.

Portal: **Certificates & secrets** → **Federated credentials** → **Add credential**.

GitHub example fields:

| Field | Example |
| --- | --- |
| Issuer | `https://token.actions.githubusercontent.com` |
| Subject | `repo:org/name:environment:prod` or `repo:org/name:ref:refs/heads/main` |
| Audience | `api://AzureADTokenExchange` |

Then grant that app registration **Azure RBAC** (e.g. Website Contributor) scoped tightly.

## Option D — Managed identity (when the host is Azure)

Not a credential on the app registration blade—**the Azure resource identity** is the principal. Prefer this for App Service → Key Vault / Graph / Azure APIs.

## Decision table

| Workload | Prefer |
| --- | --- |
| App Service calling Azure | Managed identity |
| GitHub Actions deploy | Federated credential |
| On-prem daemon | Cert in HSM/KV; avoid long secrets |
| Quick throwaway lab | Short secret OK if deleted after |

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| AADSTS7000215 invalid secret | Expired or wrong secret |
| Federation subject mismatch | Branch/env subject string typo |
| Cert auth fails | Uploaded private key by mistake; wrong thumbprint |
| Secret in `appsettings.json` | Sample defaults left in repo |

## What you should remember

1. **Secrets are shared passwords**—treat them as temporary.
2. **Certificates improve secret-string problems** but still need lifecycle.
3. **Federated credentials** remove standing Entra secrets for CI.
4. **Managed identity** is ideal inside Azure.
5. **Least privilege RBAC/scopes** matters as much as credential type.

Next: enforce token validation so only the right audiences reach your API.
