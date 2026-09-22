---
layout: article
title: Azure Bastion from Scratch
topic: How-To
category: how-to
diagram: /images/bastion-setup.svg
summary: A start-to-finish walkthrough for standing up Azure Bastion the first time—VNet, the special subnet name, public IP, connect to a private VM, and what to tear down when you’re done.
author: Todd Williamsen
date: 2026-09-15
description: Step-by-step guide to set up Azure Bastion from scratch for beginners—resource group, VNet, AzureBastionSubnet, Standard public IP, Bastion host, private VM, portal connect, costs, and cleanup.
permalink: /articles/azure-bastion-from-scratch/
---

Azure Bastion is Microsoft’s managed jump box: you open a browser (or the Azure CLI / native client on Standard SKU), pick a VM that has **no public IP**, and RDP or SSH lands on the private NIC. Nothing inbound from the internet to that VM. That’s the whole point.

This walkthrough assumes you have never done it before. We will build a tiny lab: one VNet, Bastion, and one Linux VM with only a private IP. Portal clicks first; Azure CLI equivalents at the end of each major step if you prefer typing.

<figure>
  <img src="{{ '/images/bastion-setup.svg' | relative_url }}" alt="Browser to Azure Bastion public endpoint into AzureBastionSubnet, then private RDP or SSH to a VM with no public IP">
  <figcaption>Figure 1. You connect to Bastion. Bastion connects to the VM. The VM never needs a public IP.</figcaption>
</figure>

## What you need before you start

- An Azure subscription where you can create networking and compute resources
- Owner or Contributor on a resource group (or permission to create one)
- About **30–45 minutes** the first time
- Willingness to delete the lab afterward—Bastion is **not free** while it exists

Rough cost reality check (varies by region and SKU): a Bastion host plus its public IP often runs on the order of **tens of dollars per day** if you leave Basic sitting there. Treat this as a lab you tear down, not a pet you forget.

## Step 1 — Create a resource group

1. In the Azure portal, search for **Resource groups** → **Create**.
2. Subscription: pick yours.
3. Resource group name: `rg-bastion-lab` (or anything you’ll recognize).
4. Region: pick one close to you (example: **East US**). Bastion, the VNet, and the VM must all live in **this same region**.
5. **Review + create** → **Create**.

CLI:

```bash
az group create \
  --name rg-bastion-lab \
  --location eastus
```

## Step 2 — Create a virtual network (with the right subnet)

Bastion will not deploy into a random subnet. You need a subnet whose name is **exactly**:

```text
AzureBastionSubnet
```

Capitalization matters. Size must be **at least /26** (64 addresses). Smaller prefixes fail with a validation error that sends first-timers on a wild goose chase.

### Portal

1. Search **Virtual networks** → **Create**.
2. Resource group: `rg-bastion-lab`.
3. Name: `vnet-bastion-lab`.
4. Region: **same as the resource group**.
5. On **IP addresses**:
   - Address space: `10.60.0.0/16` (example—any RFC1918 space is fine).
   - Delete the default subnet if the wizard created `default`, or rename/edit as follows.
6. Add subnet **AzureBastionSubnet**:
   - Name: `AzureBastionSubnet` (exact spelling)
   - Subnet address range: `10.60.0.0/26`
7. Add a second subnet for your VM:
   - Name: `snet-workload`
   - Subnet address range: `10.60.1.0/24`
8. **Review + create** → **Create**.

You do **not** need to attach an NSG to `AzureBastionSubnet` for a first lab. Azure manages Bastion’s control plane. Putting a “helpful” deny-all NSG on that subnet is a common way to break the deployment.

### CLI

```bash
az network vnet create \
  --resource-group rg-bastion-lab \
  --name vnet-bastion-lab \
  --address-prefix 10.60.0.0/16 \
  --subnet-name AzureBastionSubnet \
  --subnet-prefix 10.60.0.0/26

az network vnet subnet create \
  --resource-group rg-bastion-lab \
  --vnet-name vnet-bastion-lab \
  --name snet-workload \
  --address-prefix 10.60.1.0/24
```

## Step 3 — Create a Standard public IP for Bastion

Bastion requires a **Standard** SKU public IP with **static** allocation. Basic public IPs will not work for current Bastion deployments.

### Portal

1. Search **Public IP addresses** → **Create**.
2. Resource group: `rg-bastion-lab`.
3. Name: `pip-bas-lab`.
4. Region: same region again.
5. SKU: **Standard**.
6. Assignment: **Static**.
7. **Review + create** → **Create**.

### CLI

```bash
az network public-ip create \
  --resource-group rg-bastion-lab \
  --name pip-bas-lab \
  --sku Standard \
  --allocation-method Static \
  --location eastus
```

## Step 4 — Create the Bastion host

### Portal

1. Search **Bastions** → **Create**.
2. Resource group: `rg-bastion-lab`.
3. Name: `bas-lab`.
4. Region: same region.
5. Tier: **Basic** is enough for this walkthrough (browser RDP/SSH). Choose **Standard** later if you need native client, shareable links, or IP-based connect.
6. Virtual network: `vnet-bastion-lab`. The portal should auto-select `AzureBastionSubnet`.
7. Public IP: **Use existing** → `pip-bas-lab`.
8. **Review + create** → **Create**.

Deployment often takes **5–15 minutes**. Do not refresh frantically and click Create again—you will double-bill yourself into two half-broken attempts.

### CLI

```bash
az network bastion create \
  --resource-group rg-bastion-lab \
  --name bas-lab \
  --public-ip-address pip-bas-lab \
  --vnet-name vnet-bastion-lab \
  --location eastus \
  --sku Basic
```

When the portal shows the Bastion resource as **Succeeded**, move on.

## Step 5 — Create a VM with no public IP

If the VM has a public IP, you can still use Bastion, but you have not proven the private-path story. Build it the “secure by default” way.

### Portal

1. Search **Virtual machines** → **Create** → **Azure virtual machine**.
2. Resource group: `rg-bastion-lab`.
3. Name: `vm-private-lab`.
4. Region: same region.
5. Image: **Ubuntu Server 22.04 LTS** (or Windows Server if you prefer RDP).
6. Size: something small (`Standard_B2s` is fine for a lab).
7. Authentication: Password (simplest for a first Bastion test) or SSH public key.
8. On **Networking**:
   - Virtual network: `vnet-bastion-lab`
   - Subnet: `snet-workload`
   - Public IP: **None**
   - NIC network security group: **Basic** is fine for a first pass, or **Advanced** with an NSG that allows **22** (Linux) or **3389** (Windows) from the Bastion subnet `10.60.0.0/26`. If you lock the NSG to deny everything and forget Bastion’s source, connect will hang.
9. Leave inbound ports “None” if the wizard offers opening 22/3389 to the internet—you do not want that.
10. **Review + create** → **Create**.

Wait until the VM shows **Running**.

### CLI (Linux example)

```bash
az vm create \
  --resource-group rg-bastion-lab \
  --name vm-private-lab \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --vnet-name vnet-bastion-lab \
  --subnet snet-workload \
  --public-ip-address "" \
  --admin-username azureadmin \
  --authentication-type password \
  --admin-password 'ReplaceWithAComplexPass1!'
```

## Step 6 — Connect through Bastion

1. Open the VM blade: `vm-private-lab`.
2. Click **Connect** → **Bastion**.
3. Enter the VM username and password (or paste your SSH private key if you used key auth and the portal prompts for it).
4. Click **Connect**.

A browser session should open. For Linux you get an SSH terminal in the browser. For Windows you get an RDP canvas.

If it fails, work this checklist in order:

| Symptom | Likely cause |
| --- | --- |
| Bastion option missing on Connect | Bastion not in the same VNet (or peered correctly), or still deploying |
| “Bastion is not deployed in this VNet” | Wrong VNet selected when you created Bastion |
| Connect spins forever | NSG on the VM subnet blocking 22/3389 from `AzureBastionSubnet` |
| Create Bastion failed on subnet | Subnet not named `AzureBastionSubnet`, or smaller than /26 |
| Public IP SKU error | Public IP is Basic instead of Standard |

## Step 7 — Optional NSG hardening on the workload subnet

Once Bastion works, tighten the workload NSG:

1. Allow **TCP 22** (and/or **3389**) **from** `10.60.0.0/26` (your Bastion subnet) **to** the workload subnet.
2. Deny other VNet inbound if you want to be strict.
3. Do **not** open 22/3389 to `Internet` “just in case.” That undoes the exercise.

Bastion itself still needs its managed path to the control plane; leave `AzureBastionSubnet` alone unless you know you are following Microsoft’s documented NSG rules for Bastion.

## Step 8 — Tear it down

When you are done learning:

1. Delete the resource group `rg-bastion-lab` (portal: Resource group → **Delete resource group**), **or**
2. CLI:

```bash
az group delete --name rg-bastion-lab --yes --no-wait
```

That removes Bastion, the public IP, the VM, disks, NICs, and the VNet together. Confirm in **Cost Management** the next day that the meter stopped.

## What you should remember

1. **Subnet name is a contract:** `AzureBastionSubnet`, /26 or larger.
2. **Public IP is Standard + static** for Bastion.
3. **Same region** for VNet, Bastion, and targets.
4. **Target VMs do not need public IPs**—that is the design.
5. **Bastion costs money while it exists**—labs get deleted.

Once this path is boring, the next upgrades are Standard SKU features, hub-and-spoke peering (Bastion in the hub, VMs in spokes), and encoding the same layout in Bicep or ARM so nobody recreates `AzureBastionSubnet` as `bastion-subnet` at 4:59 p.m. on a Friday.
