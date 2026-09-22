---
layout: article
title: Private Endpoint for Azure Storage
topic: How-To
category: how-to
level: Intermediate
diagram: /images/howto-private-endpoint-storage.svg
summary: Expose Blob storage on a private IP in your VNet with a private endpoint, Private DNS zone, and a final cutover that disables public network access.
author: Todd Williamsen
date: 2025-09-29
description: Intermediate how-to for Storage private endpoints—subnet, private DNS zone, blob PE, portal and CLI, disable public access, and connectivity checks.
permalink: /articles/private-endpoint-storage/
---

Public storage endpoints with IP allowlists are better than open, but private endpoints put the data plane on your VNet. This walkthrough creates a private endpoint for Blob, wires `privatelink.blob.core.windows.net`, and verifies name resolution from a VM.

<figure>
  <img src="{{ '/images/howto-private-endpoint-storage.svg' | relative_url }}" alt="VM to private endpoint NIC to Storage via Private DNS">
  <figcaption>Figure 1. Private DNS resolves the blob FQDN to a private IP on your PE subnet.</figcaption>
</figure>

## What you need before you start

- VNet with `snet-priv-endpoints` (dedicated subnet recommended)
- Storage account already created
- A VM in the same VNet (or peered) for testing
- About **35–45 minutes**

## Step 1 — Create the Private DNS zone and link

### Portal

1. Search **Private DNS zones** → **Create**.
2. Name: `privatelink.blob.core.windows.net`.
3. Resource group: `rg-lab-net`.
4. After create → **Virtual network links** → **Add**.
5. Link name: `link-vnet-lab`.
6. VNet: `vnet-lab`. Enable auto-registration only if you intend it (often off for PE zones).
7. OK.

### CLI

```bash
az network private-dns zone create \
  --resource-group rg-lab-net \
  --name privatelink.blob.core.windows.net

az network private-dns link vnet create \
  --resource-group rg-lab-net \
  --zone-name privatelink.blob.core.windows.net \
  --name link-vnet-lab \
  --virtual-network vnet-lab \
  --registration-enabled false
```

## Step 2 — Create the private endpoint

### Portal

1. Storage account → **Networking** → **Private endpoint connections** → **+ Private endpoint**.
2. Resource group / name: `pe-stlab-blob`.
3. Resource sub-resource: **blob**.
4. VNet / subnet: `snet-priv-endpoints`.
5. Integrate with private DNS zone: **Yes** → select `privatelink.blob.core.windows.net`.
6. Review + create.

### CLI

```bash
ST_ID=$(az storage account show -g rg-lab-data -n stlabtoddw001 --query id -o tsv)

az network private-endpoint create \
  --resource-group rg-lab-net \
  --name pe-stlab-blob \
  --vnet-name vnet-lab \
  --subnet snet-priv-endpoints \
  --private-connection-resource-id "$ST_ID" \
  --group-id blob \
  --connection-name pe-stlab-blob-conn

az network private-endpoint dns-zone-group create \
  --resource-group rg-lab-net \
  --endpoint-name pe-stlab-blob \
  --name default \
  --private-dns-zone privatelink.blob.core.windows.net \
  --zone-name privatelink.blob.core.windows.net
```

## Step 3 — Verify DNS from the VM

On a VM in the linked VNet:

```bash
nslookup stlabtoddw001.blob.core.windows.net
```

You want a **private** A record (10.x), not only the public answer. From outside the VNet you may still see public resolution—that is expected until you disable public access.

## Step 4 — Disable public network access

Once the VM path works with Entra/RBAC:

### Portal

1. Storage → **Networking** → **Firewalls and virtual networks**.
2. Public network access: **Disabled**.
3. Save.

### CLI

```bash
az storage account update \
  --resource-group rg-lab-data \
  --name stlabtoddw001 \
  --public-network-access Disabled
```

Portal and public internet clients will fail by design. Admin from home needs VPN, Bastion jump, or a temporary exception process.

## Step 5 — Retest data plane

From the VM (managed identity or logged-in CLI):

```bash
az storage blob list \
  --account-name stlabtoddw001 \
  --container-name app-data \
  --auth-mode login \
  --output table
```

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| nslookup still public inside VNet | DNS zone not linked; or VM uses custom DNS that does not forward |
| PE created but connection Pending | Manual approval required; approve the connection |
| Works then breaks after public disable | Client still on public internet path |
| Wrong sub-resource | Created `dfs` or `file` PE but testing blob |

## What you should remember

1. **Private DNS zone name is a contract** (`privatelink.blob.core.windows.net`).
2. **Link the zone to every VNet** that must resolve the PE.
3. **Separate PE subnet** keeps address planning clean.
4. **Disable public access** only after private path is proven.
5. **Operators need a private admin path** (Bastion/VPN) afterward.

Next: PIM for Azure RBAC so standing Contributor on these resources is temporary.
