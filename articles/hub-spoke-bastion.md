---
layout: article
title: Hub-and-Spoke with Bastion in the Hub
topic: How-To
category: how-to
level: Advanced
diagram: /images/howto-hub-spoke-bastion.svg
summary: Build a hub VNet that hosts Bastion and peer spoke VNets so private VMs in spokes are reachable through one Bastion without public IPs.
author: Todd Williamsen
date: 2025-10-02
description: Advanced how-to for hub-and-spoke with Bastion in the hub—hub VNet, spoke peering both directions, Bastion connect to spoke VMs, NSG notes.
permalink: /articles/hub-spoke-bastion/
---

One Bastion per spoke gets expensive and noisy. The usual pattern: **Bastion in the hub**, spokes peer to the hub, workloads stay private. This walkthrough builds a minimal hub, one spoke, peering both ways, and a connect path to a spoke VM.

<figure>
  <img src="{{ '/images/howto-hub-spoke-bastion.svg' | relative_url }}" alt="Hub VNet with Bastion peered to spoke VNet with private VM">
  <figcaption>Figure 1. Bastion lives in the hub. Spokes peer in. VMs never need public IPs.</figcaption>
</figure>

## What you need before you start

- Rights to create two VNets, peering, Bastion, and a VM
- Non-overlapping CIDRs (example hub `10.60.0.0/16`, spoke `10.70.0.0/16`)
- About **45–60 minutes**
- Willingness to delete Bastion afterward (cost)

## Step 1 — Create the hub VNet with AzureBastionSubnet

```bash
az group create -n rg-hub -l eastus

az network vnet create \
  --resource-group rg-hub \
  --name vnet-hub \
  --address-prefix 10.60.0.0/16 \
  --subnet-name AzureBastionSubnet \
  --subnet-prefix 10.60.0.0/26
```

## Step 2 — Deploy Bastion in the hub

```bash
az network public-ip create \
  --resource-group rg-hub \
  --name pip-bas-hub \
  --sku Standard \
  --allocation-method Static

az network bastion create \
  --resource-group rg-hub \
  --name bas-hub \
  --public-ip-address pip-bas-hub \
  --vnet-name vnet-hub \
  --location eastus \
  --sku Standard
```

**Standard** (or higher) is typically required for reliable spoke connectivity scenarios; confirm current SKU requirements for your connect method. Wait until Bastion succeeds.

## Step 3 — Create the spoke VNet and workload subnet

```bash
az group create -n rg-spoke1 -l eastus

az network vnet create \
  --resource-group rg-spoke1 \
  --name vnet-spoke1 \
  --address-prefix 10.70.0.0/16 \
  --subnet-name snet-workload \
  --subnet-prefix 10.70.1.0/24
```

## Step 4 — Peer hub and spoke (both directions)

Peering is not fully useful until **both** sides exist.

```bash
# Hub -> Spoke
az network vnet peering create \
  --resource-group rg-hub \
  --name hub-to-spoke1 \
  --vnet-name vnet-hub \
  --remote-vnet $(az network vnet show -g rg-spoke1 -n vnet-spoke1 --query id -o tsv) \
  --allow-vnet-access true \
  --allow-forwarded-traffic true

# Spoke -> Hub
az network vnet peering create \
  --resource-group rg-spoke1 \
  --name spoke1-to-hub \
  --vnet-name vnet-spoke1 \
  --remote-vnet $(az network vnet show -g rg-hub -n vnet-hub --query id -o tsv) \
  --allow-vnet-access true \
  --allow-forwarded-traffic true
```

Confirm both peerings show **Connected**.

## Step 5 — Spoke VM with no public IP

```bash
az network nsg create -g rg-spoke1 -n nsg-spoke1-workload

az network nsg rule create \
  --resource-group rg-spoke1 \
  --nsg-name nsg-spoke1-workload \
  --name allow-ssh-from-bastion \
  --priority 100 \
  --access Allow \
  --protocol Tcp \
  --direction Inbound \
  --source-address-prefixes 10.60.0.0/26 \
  --destination-port-ranges 22

az network vnet subnet update \
  --resource-group rg-spoke1 \
  --vnet-name vnet-spoke1 \
  --name snet-workload \
  --network-security-group nsg-spoke1-workload

az vm create \
  --resource-group rg-spoke1 \
  --name vm-spoke1 \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --vnet-name vnet-spoke1 \
  --subnet snet-workload \
  --public-ip-address "" \
  --nsg "" \
  --admin-username azureadmin \
  --generate-ssh-keys
```

## Step 6 — Connect via hub Bastion

### Portal

1. Open `vm-spoke1` → **Connect** → **Bastion**.
2. If prompted, select the hub Bastion / ensure Bastion VNet peering path is valid.
3. Authenticate and confirm a shell opens.

### Notes that bite people

- Spoke NSG must allow 22/3389 **from the hub Bastion subnet**, not only from the spoke.
- Do not place Bastion in the spoke “just for this one VM” if hub is the standard.
- Gateway transit / firewalls in the middle need extra design (NVA, Azure Firewall hub)—out of scope for this minimal lab.

## Step 7 — Tear down

Delete `rg-spoke1` and `rg-hub` when finished (hub first or together once peerings are gone). Bastion is the cost driver.

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| Bastion option missing on spoke VM | Peering not Connected both ways; or SKU/feature limits |
| Connect hangs | NSG blocks Bastion subnet `10.60.0.0/26` |
| Overlapping address space | Hub and spoke CIDRs collide |
| Two Bastions accidentally | Created Bastion in spoke as well—delete the spare |

## What you should remember

1. **Bastion in hub**, workloads in spokes.
2. **Peering both directions** and Connected status.
3. **NSG allows Bastion hub CIDR** into spoke workloads.
4. **Non-overlapping CIDRs** planned up front.
5. **Delete labs**—hub Bastion is not a souvenir.

Next: management groups and policy so every subscription lands with this pattern encoded.
