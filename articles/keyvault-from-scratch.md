---
layout: article
title: Azure Key Vault from Scratch
topic: How-To
category: how-to
level: Intermediate
diagram: /images/howto-keyvault-from-scratch.svg
summary: Create a Key Vault with RBAC authorization, soft delete and purge protection, a first secret, and network defaults that do not leave the vault wide open forever.
author: Todd Williamsen
date: 2025-09-25
description: Intermediate how-to for Azure Key Vault from scratch—RBAC model, soft delete, purge protection, secrets, access policies vs RBAC, and portal plus CLI steps.
permalink: /articles/keyvault-from-scratch/
---

Key Vault is where secrets, keys, and certificates should live instead of config files and wiki pages. This walkthrough creates a vault using the **RBAC** permission model, turns on soft delete and purge protection, and stores a first secret you can retrieve with your identity.

<figure>
  <img src="{{ '/images/howto-keyvault-from-scratch.svg' | relative_url }}" alt="Key Vault with RBAC, soft delete, purge protection, and a secret">
  <figcaption>Figure 1. RBAC at the vault. Soft delete and purge protection on. Secrets retrieved by identity.</figcaption>
</figure>

## What you need before you start

- Resource group `rg-lab-data`
- Rights to create Key Vault and assign roles
- About **25–35 minutes**
- Azure CLI logged in (`az login`)

## Step 1 — Create the vault (RBAC authorization)

Prefer **Azure role-based access control** over the older vault access-policy model for new vaults.

### Portal

1. Search **Key vaults** → **Create**.
2. Resource group: `rg-lab-data`.
3. Name: globally unique, e.g. `kv-lab-toddw001`.
4. Region: same as workloads that will call it.
5. Pricing tier: **Standard** for secrets labs.
6. On **Access configuration**: **Azure role-based access control**.
7. On **Networking**: start with public endpoint for the lab; lock down later (private endpoint how-to).
8. **Review + create** → **Create**.

### CLI

```bash
az keyvault create \
  --resource-group rg-lab-data \
  --name kv-lab-toddw001 \
  --location eastus \
  --enable-rbac-authorization true \
  --enable-purge-protection true \
  --retention-days 90
```

Soft delete is standard on current vaults; purge protection stops permanent deletion during the retention window—turn it on for anything that is not a disposable toy.

## Step 2 — Grant yourself data-plane rights

Control plane (create vault) does not automatically let you read secrets.

### Portal

1. Key Vault → **Access control (IAM)** → **Add role assignment**.
2. Role: **Key Vault Secrets Officer** (manage secrets) or **Key Vault Administrator** for a lab admin.
3. Assign to your user → Review + assign.

### CLI

```bash
USER_ID=$(az ad signed-in-user show --query id -o tsv)
KV_ID=$(az keyvault show -g rg-lab-data -n kv-lab-toddw001 --query id -o tsv)

az role assignment create \
  --role "Key Vault Secrets Officer" \
  --assignee-object-id "$USER_ID" \
  --assignee-principal-type User \
  --scope "$KV_ID"
```

Wait 1–2 minutes for RBAC propagation.

## Step 3 — Add a secret

### Portal

1. Key Vault → **Secrets** → **Generate/Import**.
2. Name: `DbConnectionString` (use a naming convention).
3. Value: a dummy string for the lab.
4. Create.

### CLI

```bash
az keyvault secret set \
  --vault-name kv-lab-toddw001 \
  --name DbConnectionString \
  --value "Server=tcp:demo.database.windows.net;..."
```

## Step 4 — Read it back (prove RBAC)

```bash
az keyvault secret show \
  --vault-name kv-lab-toddw001 \
  --name DbConnectionString \
  --query value -o tsv
```

If this fails with Forbidden, RBAC has not propagated or you assigned a control-plane role only (e.g. Contributor on the RG without Key Vault data roles).

## Step 5 — Soft delete and recovery awareness

1. Delete a test secret in the portal.
2. **Secrets** → manage deleted secrets / recover.
3. Note purge protection: you cannot instantly purge the vault while protection is enabled.

Practice recover once so incident response is not theoretical.

## Step 6 — Logging (do not skip)

Key Vault → **Diagnostic settings** → send **AuditEvent** to a Log Analytics workspace. Without this, you will not see who read secrets when it matters.

(Full diagnostics walkthrough is a separate how-to; at least create the setting now.)

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| Forbidden on secret show | Missing Key Vault data-plane RBAC role |
| Access policies UI empty / confusing | Vault is RBAC mode; use IAM, not access policies |
| Cannot purge vault | Purge protection working as designed |
| App works in portal but not from VM | VM identity lacks role; or network firewall blocks |

## What you should remember

1. **RBAC authorization** for new vaults.
2. **Data-plane roles** are separate from Contributor.
3. **Purge protection** for anything non-ephemeral.
4. **Diagnostic AuditEvent** to Log Analytics early.
5. **Never** commit secret values; commit Key Vault references.

Next: VM managed identity reading this vault (and Storage) without keys on disk.
