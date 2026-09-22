---
layout: article
title: VM Managed Identity to Key Vault and Storage
topic: How-To
category: how-to
level: Intermediate
diagram: /images/howto-vm-managed-identity-keyvault.svg
summary: Enable a system-assigned managed identity on a Linux VM, grant Key Vault and Storage data roles, and prove the VM can read a secret and list blobs without keys on disk.
author: Todd Williamsen
date: 2025-09-26
description: Intermediate how-to wiring a VM managed identity to Key Vault and Storage—enable identity, RBAC roles, Azure CLI on the VM, and common Forbidden fixes.
permalink: /articles/vm-managed-identity-keyvault/
---

Passwords in `/etc/app.env` are how labs become incidents. Managed identity lets the VM authenticate to Azure AD and call Key Vault and Storage with RBAC—no secret files to rotate by hand.

Assumes you have a private (or any) Linux VM, a Key Vault in RBAC mode with a secret, and a storage account with a container.

<figure>
  <img src="{{ '/images/howto-vm-managed-identity-keyvault.svg' | relative_url }}" alt="VM system-assigned identity to Key Vault and Storage via RBAC">
  <figcaption>Figure 1. The VM identity is the principal. Roles on Key Vault and Storage replace keys on disk.</figcaption>
</figure>

## What you need before you start

- Linux VM you can SSH/Bastion into
- Key Vault with at least one secret
- Storage account + container
- Rights to enable identity and assign roles
- About **30 minutes**

## Step 1 — Enable system-assigned managed identity

### Portal

1. Open the VM → **Identity** → **System assigned**.
2. Status: **On** → **Save** → Yes.
3. Copy the **Object (principal) ID** for later.

### CLI

```bash
az vm identity assign \
  --resource-group rg-lab-compute \
  --name vm-linux-private
```

## Step 2 — Grant Key Vault Secrets User

Least privilege for read:

### Portal

1. Key Vault → **IAM** → **Add role assignment**.
2. Role: **Key Vault Secrets User**.
3. Assign access to **Managed identity** → select your VM.
4. Review + assign.

### CLI

```bash
VM_PID=$(az vm identity show -g rg-lab-compute -n vm-linux-private --query principalId -o tsv)
KV_ID=$(az keyvault show -g rg-lab-data -n kv-lab-toddw001 --query id -o tsv)

az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee-object-id "$VM_PID" \
  --assignee-principal-type ServicePrincipal \
  --scope "$KV_ID"
```

## Step 3 — Grant Storage Blob Data Reader (or Contributor)

```bash
ST_ID=$(az storage account show -g rg-lab-data -n stlabtoddw001 --query id -o tsv)

az role assignment create \
  --role "Storage Blob Data Reader" \
  --assignee-object-id "$VM_PID" \
  --assignee-principal-type ServicePrincipal \
  --scope "$ST_ID"
```

Wait for RBAC (often under two minutes; sometimes longer).

## Step 4 — On the VM: login as the identity and test

Install Azure CLI if needed, then:

```bash
# Login with the VM's managed identity
az login --identity

# Read a secret
az keyvault secret show \
  --vault-name kv-lab-toddw001 \
  --name DbConnectionString \
  --query value -o tsv

# List blobs
az storage blob list \
  --account-name stlabtoddw001 \
  --container-name app-data \
  --auth-mode login \
  --output table
```

No connection strings required on the VM for these calls.

## Step 5 — Application pattern (conceptual)

Apps should use Azure SDKs with `DefaultAzureCredential` / managed identity, not shell wrappers in production. The CLI proof above validates RBAC and network path before you wire code.

## Step 6 — Network gotchas

If Key Vault or Storage firewalls are on:

- Allow the VM subnet (service endpoints) **or**
- Use private endpoints and private DNS

Identity alone does not punch through a Deny-all firewall.

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| Forbidden on secret show | Secrets User not assigned, or still propagating |
| `az login --identity` fails | Identity not enabled; or IMDS blocked (rare custom routes) |
| Storage works with key, fails with `--auth-mode login` | Missing Blob Data RBAC on the MI |
| Works from laptop, fails on VM | Firewall allows your IP only, not the VM path |

## What you should remember

1. **Enable system-assigned identity** on the VM.
2. **Data-plane roles** on Key Vault and Storage for that principal.
3. **`az login --identity`** proves the path before app code.
4. **No secrets files** for Azure resource auth.
5. **Firewalls still apply**—open the network path deliberately.

Next: Azure Policy to audit then deny public storage, and diagnostics so identity usage is visible in logs.
