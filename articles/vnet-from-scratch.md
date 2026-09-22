---
layout: article
title: Build an Azure VNet the Right Way
topic: How-To
category: how-to
level: Beginner
diagram: /images/howto-vnet-from-scratch.svg
summary: Create a production-minded Azure virtual network—address space, subnets, NSG basics, DNS notes, and the mistakes that force painful renumbering later.
author: Todd Williamsen
date: 2025-09-21
description: Beginner how-to to build an Azure VNet the right way—CIDR planning, workload and Bastion subnets, NSG association, DNS, and CLI create steps.
permalink: /articles/vnet-from-scratch/
---

A VNet is not just “10.0.0.0/16 and hope.” Bad address plans collide with on-prem, peer poorly, and force rebuilds. This walkthrough builds a small but deliberate VNet you can grow into hub-and-spoke later.

<figure>
  <img src="{{ '/images/howto-vnet-from-scratch.svg' | relative_url }}" alt="VNet with address space, AzureBastionSubnet, workload subnet, and NSG">
  <figcaption>Figure 1. Plan the CIDR, carve subnets with jobs, attach NSGs to the workload path.</figcaption>
</figure>

## What you need before you start

- A subscription and resource group (example: `rg-lab-net`)
- Permission to create networking resources
- About **25 minutes**
- A CIDR that will not fight your corp network (avoid `10.0.0.0/8` blindly)

## Step 1 — Choose an address space you can live with

Rules of thumb for a first lab that might become real:

- Prefer something like `10.60.0.0/16` over the portal default `10.0.0.0/16` if you ever peer with other VNets that also used defaults.
- Leave headroom. Do not pack every /24 on day one.
- Document the space in a one-line comment in your runbook.

### Portal

1. Search **Virtual networks** → **Create**.
2. Resource group: `rg-lab-net`.
3. Name: `vnet-lab`.
4. Region: pick one and stick to it for peer resources.
5. On **IP addresses**, set address space to `10.60.0.0/16`.

### CLI

```bash
az network vnet create \
  --resource-group rg-lab-net \
  --name vnet-lab \
  --address-prefix 10.60.0.0/16 \
  --location eastus
```

## Step 2 — Create subnets with jobs

Minimum useful layout:

| Subnet | CIDR | Job |
| --- | --- | --- |
| `AzureBastionSubnet` | `10.60.0.0/26` | Bastion only (exact name) |
| `snet-workload` | `10.60.1.0/24` | VMs / app NICs |
| `snet-priv-endpoints` | `10.60.2.0/24` | Private endpoints (later) |

### Portal

1. Still on **IP addresses**, add or edit subnets with the names and ranges above.
2. Do not put VMs in `AzureBastionSubnet`.
3. **Review + create** → **Create**.

### CLI

```bash
az network vnet subnet create \
  --resource-group rg-lab-net \
  --vnet-name vnet-lab \
  --name AzureBastionSubnet \
  --address-prefix 10.60.0.0/26

az network vnet subnet create \
  --resource-group rg-lab-net \
  --vnet-name vnet-lab \
  --name snet-workload \
  --address-prefix 10.60.1.0/24

az network vnet subnet create \
  --resource-group rg-lab-net \
  --vnet-name vnet-lab \
  --name snet-priv-endpoints \
  --address-prefix 10.60.2.0/24
```

## Step 3 — Create and attach a workload NSG

Start closed enough to teach habits, open enough to not brick a lab.

### Portal

1. Search **Network security groups** → **Create** → name `nsg-workload`.
2. After create, open **Inbound security rules**.
3. Add allow **SSH (22)** or **RDP (3389)** from your Bastion subnet `10.60.0.0/26` only (not Internet).
4. Open the VNet → **Subnets** → `snet-workload` → **Network security group** → `nsg-workload` → **Save**.

### CLI

```bash
az network nsg create \
  --resource-group rg-lab-net \
  --name nsg-workload

az network nsg rule create \
  --resource-group rg-lab-net \
  --nsg-name nsg-workload \
  --name allow-ssh-from-bastion \
  --priority 100 \
  --access Allow \
  --protocol Tcp \
  --direction Inbound \
  --source-address-prefixes 10.60.0.0/26 \
  --destination-port-ranges 22

az network vnet subnet update \
  --resource-group rg-lab-net \
  --vnet-name vnet-lab \
  --name snet-workload \
  --network-security-group nsg-workload
```

## Step 4 — DNS: know the default

New VNets use Azure-provided DNS unless you set custom DNS. For a first VNet:

- Leave Azure DNS unless you have an AD / private DNS plan.
- When you add private endpoints later, you will create **Private DNS zones** and link them to this VNet.

Check: VNet blade → **DNS servers** → **Default (Azure-provided)**.

## Step 5 — Verify the layout

```bash
az network vnet subnet list \
  --resource-group rg-lab-net \
  --vnet-name vnet-lab \
  --output table
```

Confirm three subnets, correct prefixes, and NSG on `snet-workload`.

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| Cannot peer later | Overlapping address spaces with another VNet or on-prem |
| Bastion deploy fails | Subnet not named `AzureBastionSubnet` or smaller than /26 |
| VM unreachable | NSG denies Bastion/source; or wrong subnet selected |
| Out of IPs in a year | /24 packed with private endpoints and scale sets |

## What you should remember

1. **Pick a non-default CIDR** when peering is in your future.
2. **Subnets have jobs**—Bastion, workload, private endpoints.
3. **NSGs live on workload subnets**; be careful with Bastion’s subnet.
4. **Azure DNS is fine** until private endpoints force Private DNS zones.
5. **Document the plan**—address space changes are expensive socially, not just technically.

Next: put a private Linux VM in `snet-workload`, then Bastion in the hub pattern when you outgrow a single VNet.
