---
layout: article
title: Lock Down an Azure Storage Account from Scratch
topic: How-To
category: how-to
level: Beginner
diagram: /images/howto-storage-locked-down-from-scratch.svg
summary: Create a storage account that refuses anonymous public access, prefers Entra RBAC over keys, and is ready for network rules or private endpoints next.
author: Todd Williamsen
date: 2025-09-23
description: Beginner how-to to lock down Azure Storage from scratch—disable public blob access, TLS, shared key preference, RBAC data plane, and network firewall basics.
permalink: /articles/storage-locked-down-from-scratch/
---

Default storage wizards still tempt people into “public blob container” and long-lived account keys. This walkthrough creates a storage account you would not be embarrassed to put near real data: no anonymous public access, TLS enforced, RBAC for data plane, and a network stance you can tighten.

<figure>
  <img src="{{ '/images/howto-storage-locked-down-from-scratch.svg' | relative_url }}" alt="Storage account with public access off, RBAC, TLS, and network firewall">
  <figcaption>Figure 1. Identity over keys. No anonymous public containers. Network closed when you are ready.</figcaption>
</figure>

## What you need before you start

- Resource group `rg-lab-data` (or similar)
- Rights to create storage and assign roles
- About **25 minutes**
- A test user or your own identity for RBAC checks

## Step 1 — Create the storage account with safe defaults

### Portal

1. Search **Storage accounts** → **Create**.
2. Resource group: `rg-lab-data`.
3. Name: globally unique, e.g. `stlabtoddw001`.
4. Region: same as your workloads.
5. Performance: **Standard**. Redundancy: **LRS** for a lab.
6. On **Advanced**:
   - **Allow enabling anonymous access on individual containers**: **Disabled**
   - Prefer **Enable storage account key access** off if your tooling supports Entra-only (turn off once you confirm Azure CLI / portal RBAC works for your scenario)
7. On **Networking** (first create): you can leave public endpoint enabled for the lab, then lock firewall in Step 4.
8. **Review + create** → **Create**.

### CLI

```bash
az storage account create \
  --resource-group rg-lab-data \
  --name stlabtoddw001 \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2 \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false \
  --https-only true
```

## Step 2 — Create a private container

### Portal

1. Storage account → **Containers** → **+ Container**.
2. Name: `app-data`.
3. Anonymous access level: **Private (no anonymous access)**.
4. Create.

### CLI

```bash
az storage container create \
  --account-name stlabtoddw001 \
  --name app-data \
  --auth-mode login \
  --public-access off
```

`--auth-mode login` uses your Entra identity instead of the account key.

## Step 3 — Grant data-plane RBAC instead of sharing keys

For yourself or an app identity:

1. Storage account → **Access control (IAM)** → **Add role assignment**.
2. Role: **Storage Blob Data Contributor** (read/write) or **Storage Blob Data Reader**.
3. Assign to your user (or a managed identity later).
4. Wait a minute for propagation.

Test upload with Azure CLI (Entra):

```bash
echo "hello" > /tmp/hello.txt
az storage blob upload \
  --account-name stlabtoddw001 \
  --container-name app-data \
  --name hello.txt \
  --file /tmp/hello.txt \
  --auth-mode login
```

Avoid pasting **Access keys** into chat, tickets, or git. If a key leaked, rotate it under **Access keys** → **Rotate**.

## Step 4 — Tighten networking (lab-friendly path)

Once RBAC upload works:

### Portal

1. Storage → **Networking** → **Firewalls and virtual networks**.
2. Public network access: **Enabled from selected virtual networks and IP addresses**.
3. Add your client public IP for break-glass admin, **or** add the VNet/subnet that will use a service endpoint / private endpoint.
4. **Save**.

For a stronger end state, use a **private endpoint** (separate how-to) and disable public network access entirely.

### CLI (allow your current public IP)

```bash
MYIP=$(curl -s https://api.ipify.org)
az storage account update \
  --resource-group rg-lab-data \
  --name stlabtoddw001 \
  --default-action Deny \
  --public-network-access Enabled

az storage account network-rule add \
  --resource-group rg-lab-data \
  --account-name stlabtoddw001 \
  --ip-address "$MYIP"
```

## Step 5 — Soft delete and versioning (quick wins)

Storage account → **Data protection**:

- Enable **Blob soft delete** (7–14 days for a lab)
- Enable **Versioning** if you overwrite blobs often

These are not a backup strategy by themselves, but they save you from “oops overwrite.”

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| `AuthorizationFailure` with CLI | Missing Blob Data RBAC; still using key auth incorrectly |
| Portal lists containers but upload fails | Firewall blocked your IP after Deny default |
| “Public access is not permitted” | Account-level anonymous access disabled—good; use RBAC |
| App still works with keys after you “locked down” | Shared key access still enabled; rotate and disable when ready |

## What you should remember

1. **Disable anonymous public access** at the account.
2. **TLS 1.2+** and HTTPS-only.
3. **Blob Data RBAC** over sharing account keys.
4. **Firewall or private endpoint** before real data lands.
5. **Soft delete** is cheap insurance for blob mistakes.

Next: private endpoint for Storage, then Key Vault so connection strings are not littered across VMs.
