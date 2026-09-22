---
layout: article
title: Workload Identity for Azure Container Apps
topic: How-To
category: how-to
level: Advanced
diagram: /images/howto-workload-identity-container-apps.svg
summary: Enable workload identity on Container Apps, bind a user-assigned managed identity with federated credentials, and reach Key Vault or Azure APIs without Kubernetes secrets.
author: Todd Williamsen
date: 2025-10-23
description: Advanced how-to for Azure Container Apps workload identity—user-assigned managed identity, federated identity credential, RBAC, and DefaultAzureCredential.
permalink: /articles/workload-identity-container-apps/
---

Container images should not embed client secrets. On **Azure Container Apps**, **workload identity** federates a **user-assigned managed identity (UAMI)** so your revision obtains Entra tokens like other Azure MI workloads—without a long-lived secret in the container.

<figure>
  <img src="{{ '/images/howto-workload-identity-container-apps.svg' | relative_url }}" alt="Container Apps with UAMI federated to Entra calling Key Vault">
  <figcaption>Figure 1. Platform federation replaces kube secrets for Azure auth.</figcaption>
</figure>

## What you need before you start

- Container Apps environment and app
- Rights to create UAMI, federated credentials, and RBAC
- Azure CLI recent enough for workload identity flags
- About **40–50 minutes**

## Step 1 — Create a user-assigned managed identity

```bash
az identity create \
  --name uami-aca-lab \
  --resource-group rg-aca-lab \
  --location eastus
```

Note `clientId` and `principalId`.

## Step 2 — Enable workload identity on the environment

Follow current Microsoft docs for your CLI version—conceptually:

1. Container Apps **environment** with workload identities enabled.
2. Bind the UAMI to the **container app**.
3. Set the app to use that identity for Azure credential flow.

```bash
# Example shape — verify flag names for your CLI version
az containerapp identity assign \
  --name ca-api-lab \
  --resource-group rg-aca-lab \
  --user-assigned <uami-resource-id>
```

Enable workload identity / set `AZURE_CLIENT_ID` to the UAMI client ID in the app’s environment variables when required by the SDK.

## Step 3 — Federated credential on the UAMI

Container Apps (and AKS-style workload identity) register a federated identity credential so the platform token exchanges for an Entra token for the UAMI.

Portal: UAMI → **Federated credentials** → add per Microsoft Container Apps guidance (issuer/subject for your environment).

If federation is created automatically by `az` extensions for your scenario, still verify the credential exists before debugging apps.

## Step 4 — Grant RBAC on the target resource

```bash
az role assignment create \
  --assignee-object-id <uami-principal-id> \
  --assignee-principal-type ServicePrincipal \
  --role "Key Vault Secrets User" \
  --scope <key-vault-resource-id>
```

Least privilege per resource. Do not grant Contributor on the subscription for app runtime.

## Step 5 — Code with Azure Identity

```csharp
var credential = new DefaultAzureCredential();
var client = new SecretClient(new Uri(vaultUri), credential);
```

Inside Container Apps with workload identity configured, the credential chain finds the UAMI. Locally, use Azure CLI login or a dev UAMI pattern—never bake prod secrets into the image.

## Step 6 — Confirm

1. Revision starts healthy.
2. Call a path that reads Key Vault.
3. Sign-in logs / resource logs show the UAMI principal—not a client secret app.

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| Credential unavailable | Workload identity not enabled / wrong client ID env |
| 403 on vault | RBAC missing; firewall blocks egress |
| Image still has secrets | Old pattern left in env vars |
| System-assigned confusion | Docs path expects UAMI + federation |

## What you should remember

1. **Workload identity > secrets in containers**.
2. **UAMI + federation + RBAC** is the usual ACA pattern.
3. **DefaultAzureCredential** in code; configure platform identity outside the image.
4. **Least privilege** on Key Vault/Storage/SQL.
5. **No secrets in git or image layers**.

Next: Conditional Access policies targeting your enterprise app.
