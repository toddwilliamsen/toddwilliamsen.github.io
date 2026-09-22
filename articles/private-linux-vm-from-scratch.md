---
layout: article
title: Private Linux VM in Azure from Scratch
topic: How-To
category: how-to
level: Beginner
diagram: /images/howto-private-linux-vm-from-scratch.svg
summary: Deploy an Ubuntu VM with no public IP, SSH key auth, a tight NSG, and a path to reach it via Bastion or a jump host you already trust.
author: Todd Williamsen
date: 2025-09-22
description: Beginner how-to for a private Linux VM in Azure—no public IP, SSH keys, NSG from Bastion subnet, portal and CLI create, connect checklist.
permalink: /articles/private-linux-vm-from-scratch/
---

A VM with a public IP and port 22 open to the world is a training exercise for attackers. This walkthrough builds a private Ubuntu VM: private NIC only, SSH key auth, NSG that allows SSH from your Bastion (or admin) subnet—not from the internet.

Assumes you already have a VNet with `snet-workload` and preferably Bastion (or another private path). If not, create the VNet first, then come back.

<figure>
  <img src="{{ '/images/howto-private-linux-vm-from-scratch.svg' | relative_url }}" alt="Private Linux VM with no public IP, NSG allow from Bastion subnet only">
  <figcaption>Figure 1. No public IP. SSH only from the admin path you control.</figcaption>
</figure>

## What you need before you start

- Resource group and VNet with `snet-workload`
- Bastion in the same VNet, or VPN / jump box already working
- An SSH public key on your machine (`~/.ssh/id_rsa.pub` or similar)
- About **20–30 minutes**

## Step 1 — Confirm the subnet and NSG

Workload subnet should allow TCP 22 from `AzureBastionSubnet` (example `10.60.0.0/26`). If the NSG allows Internet:22, fix that before you call the VM “private.”

```bash
az network nsg rule list \
  --resource-group rg-lab-net \
  --nsg-name nsg-workload \
  --output table
```

## Step 2 — Create the VM with no public IP

### Portal

1. Search **Virtual machines** → **Create** → **Azure virtual machine**.
2. Resource group: `rg-lab-compute` (or your compute RG).
3. Name: `vm-linux-private`.
4. Region: same as the VNet.
5. Image: **Ubuntu Server 22.04 LTS**.
6. Size: `Standard_B2s` for a lab.
7. Authentication type: **SSH public key**.
8. Username: `azureadmin` (or your standard).
9. Paste your public key.
10. **Networking**:
    - VNet / subnet: `snet-workload`
    - Public IP: **None**
    - NIC NSG: use your existing `nsg-workload` (Advanced)
    - Public inbound ports: **None**
11. **Review + create** → **Create**.

### CLI

```bash
az vm create \
  --resource-group rg-lab-compute \
  --name vm-linux-private \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --vnet-name vnet-lab \
  --subnet snet-workload \
  --nsg "" \
  --public-ip-address "" \
  --admin-username azureadmin \
  --ssh-key-values ~/.ssh/id_rsa.pub
```

If the NIC needs the subnet NSG already associated at the subnet level, `--nsg ""` avoids a second NSG on the NIC. Prefer subnet NSGs for consistency.

## Step 3 — Verify no public IP

### Portal

VM → **Networking** → confirm Public IP is blank / none.

### CLI

```bash
az vm list-ip-addresses \
  --resource-group rg-lab-compute \
  --name vm-linux-private \
  --output table
```

You should see a private IP only (example `10.60.1.4`).

## Step 4 — Connect the private way

### Via Bastion (portal)

1. Open the VM → **Connect** → **Bastion**.
2. Username: `azureadmin`.
3. Authentication: SSH private key (upload/paste as prompted).
4. **Connect**.

### Via Azure CLI Bastion tunnel (Standard SKU)

```bash
az network bastion ssh \
  --name bas-lab \
  --resource-group rg-bastion-lab \
  --target-resource-id $(az vm show -g rg-lab-compute -n vm-linux-private --query id -o tsv) \
  --auth-type ssh-key \
  --username azureadmin \
  --ssh-key ~/.ssh/id_rsa
```

Basic Bastion is browser-only; use portal Connect if you are on Basic.

## Step 5 — First-login checklist on the box

Once inside:

```bash
whoami
ip -br a
sudo apt-get update
```

Optional hardening habits (keep short for a lab):

- Disable password SSH if you enabled passwords by mistake
- Keep the OS patched on a schedule
- Do not install random inbound agents that open ports

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| Bastion connect hangs | NSG blocks 22 from Bastion subnet |
| “No Bastion in this VNet” | Bastion in another VNet without peering |
| CLI create added a public IP | Forgot `--public-ip-address ""` |
| Auth fails | Wrong username or private key does not match pasted public key |

## What you should remember

1. **Public IP: None** is the default you want for private workloads.
2. **SSH keys**, not passwords, for Linux labs that might linger.
3. **NSG source = Bastion/admin subnet**, never `Internet` for 22.
4. **Same region / VNet path** as Bastion or your jump host.
5. **Prove privacy** with `list-ip-addresses` before you move on.

Next: lock down a storage account the same way—no public data plane by default—and wire the VM to Key Vault with a managed identity.
