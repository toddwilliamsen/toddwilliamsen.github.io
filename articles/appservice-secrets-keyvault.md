---
layout: article
title: Store Web App Secrets in Key Vault
topic: How-To
category: how-to
level: Beginner
diagram: /images/howto-appservice-secrets-keyvault.svg
summary: Turn on a managed identity for App Service, put secrets in Key Vault, and reference them from app settings—no secrets in git.
author: Todd Williamsen
date: 2025-10-12
description: Beginner walkthrough for Azure App Service Key Vault references—system-assigned managed identity, RBAC, secret URI, and app setting syntax.
permalink: /articles/appservice-secrets-keyvault/
---

Plain text **Application settings** are convenient and dangerous. They show up in export templates, screenshots, and support tickets. Put secrets in **Key Vault** and let App Service resolve them with a **managed identity**.

<figure>
  <img src="{{ '/images/howto-appservice-secrets-keyvault.svg' | relative_url }}" alt="App Service managed identity reads secrets from Key Vault; git stays clean">
  <figcaption>Figure 1. The identity of the app gets Get/List on secrets—not your developers' personal keys in the repo.</figcaption>
</figure>

## What you need before you start

- An App Service Web App
- Rights to create a Key Vault and assign RBAC
- About **25 minutes**
- Decision: use **Azure RBAC** on Key Vault (recommended) rather than legacy access policies when you can

## Step 1 — Enable system-assigned managed identity

### Portal

1. Web App → **Identity** → **System assigned** → **On** → **Save**.
2. Copy the **Object (principal) ID**.

### CLI

```bash
az webapp identity assign \
  --resource-group rg-appservice-lab \
  --name <your-app>
```

## Step 2 — Create Key Vault and a secret

```bash
az keyvault create \
  --name kv-contoso-lab-UNIQUE \
  --resource-group rg-appservice-lab \
  --location eastus \
  --enable-rbac-authorization true

az keyvault secret set \
  --vault-name kv-contoso-lab-UNIQUE \
  --name DbConnection \
  --value "Server=...;User=...;Password=...;"
```

Portal path: **Key vaults** → create → **Secrets** → **Generate/Import**.

## Step 3 — Grant the Web App access

With RBAC:

1. Key Vault → **Access control (IAM)** → **Add role assignment**.
2. Role: **Key Vault Secrets User**.
3. Assign access to: **Managed identity** → select your Web App.
4. **Review + assign**.

```bash
# Resolve principal id from the webapp identity
PRINCIPAL=$(az webapp identity show -g rg-appservice-lab -n <your-app> --query principalId -o tsv)
VAULT_ID=$(az keyvault show -n kv-contoso-lab-UNIQUE --query id -o tsv)

az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee-object-id $PRINCIPAL \
  --assignee-principal-type ServicePrincipal \
  --scope $VAULT_ID
```

Do **not** grant yourself “do everything” roles to the app identity. Secrets User is enough for references.

## Step 4 — Add a Key Vault reference app setting

Secret URI form (version optional—omit version to always get latest):

```text
@Microsoft.KeyVault(SecretUri=https://kv-contoso-lab-UNIQUE.vault.azure.net/secrets/DbConnection/)
```

### Portal

1. Web App → **Configuration** → **Application settings** → **New application setting**.
2. Name: `DbConnection` (or whatever your code reads).
3. Value: the `@Microsoft.KeyVault(...)` string.
4. **Save** → restart if prompted.

### CLI

```bash
az webapp config appsettings set \
  --resource-group rg-appservice-lab \
  --name <your-app> \
  --settings \
DbConnection='@Microsoft.KeyVault(SecretUri=https://kv-contoso-lab-UNIQUE.vault.azure.net/secrets/DbConnection/)'
```

In **Configuration**, a healthy reference shows a green check. Red means identity, network, or URI problems.

## Step 5 — Verify from the app

Your code still reads `Environment.GetEnvironmentVariable("DbConnection")` (or configuration). App Service resolves the reference before your process sees it—no Key Vault SDK required for basic cases.

Rotate the secret in Key Vault; with no version pinned, the app picks up the new value on refresh/restart behavior documented for references.

## Networking note

If the vault is locked down with private endpoints or firewalls, App Service must reach it (VNet integration + private DNS, or allow trusted Microsoft services carefully). For a first public lab, public vault + RBAC is enough; harden later.

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| Red X on reference | MI not granted Secrets User; wrong secret name |
| 403 from vault | Firewall blocking App Service outbound |
| Still seeing old value | Pinned secret version in URI |
| Secret in ARM output | Someone still put plaintext in template parameters |
| Access policies vs RBAC | Mixed model confusion—pick one |

## What you should remember

1. **Managed identity + Key Vault reference** beats plaintext app settings.
2. **Least privilege**: Secrets User for the app identity.
3. **No secrets in git**—including publish profiles and parameter files.
4. **Omit version** in the URI unless you need pin-for-rollback.
5. **Confirm the green check** in Configuration before debugging app code.

Next (intermediate): split UI and API into separate app registrations and scopes.
