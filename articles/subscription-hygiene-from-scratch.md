---
layout: article
title: Azure Subscription Hygiene from Scratch
topic: How-To
category: how-to
level: Beginner
diagram: /images/howto-subscription-hygiene-from-scratch.svg
summary: Stand up a clean Azure subscription baseline—naming, resource groups, tags, locks, budgets, and who can create what—before the sprawl starts.
author: Todd Williamsen
date: 2025-09-20
description: Beginner how-to for Azure subscription hygiene from scratch—resource group layout, required tags, delete locks, Cost Management budgets, and Contributor vs Owner boundaries.
permalink: /articles/subscription-hygiene-from-scratch/
---

Most Azure mess starts the same way: one subscription, random resource groups, no tags, nobody remembers who owns the bill. Subscription hygiene is the boring work that keeps landing zones and security work possible later.

This walkthrough sets a small, repeatable baseline you can apply to a lab or a real workload subscription. Portal first; CLI where it saves time.

<figure>
  <img src="{{ '/images/howto-subscription-hygiene-from-scratch.svg' | relative_url }}" alt="Subscription with resource groups, tags, locks, budget alert, and RBAC roles">
  <figcaption>Figure 1. Hygiene is structure: groups, tags, locks, spend alerts, and clear owners.</figcaption>
</figure>

## What you need before you start

- An Azure subscription where you can create resource groups and assign RBAC
- Owner or User Access Administrator plus Contributor (or equivalent)
- About **20–30 minutes**
- Agreement on a short naming prefix (example: `lab` or `contoso`)

## Step 1 — Pick a naming and RG layout

Use a few resource groups with jobs, not one dumping ground:

| Resource group | Purpose |
| --- | --- |
| `rg-{prefix}-net` | VNets, NSGs, Bastion, private endpoints |
| `rg-{prefix}-compute` | VMs, disks, availability sets |
| `rg-{prefix}-data` | Storage, Key Vault, SQL |
| `rg-{prefix}-ops` | Log Analytics, Automation, budgets (optional) |

### Portal

1. Search **Resource groups** → **Create**.
2. Create the four groups above in the same region you plan to use most.
3. Keep names lowercase, hyphenated, and short enough to type.

### CLI

```bash
PREFIX=lab
LOC=eastus
for rg in net compute data ops; do
  az group create --name "rg-${PREFIX}-${rg}" --location "$LOC"
done
```

## Step 2 — Require a small tag set

Tags are how Cost Management and Policy find owners. Start with three:

- `Owner` — email or Entra group name
- `Environment` — `lab`, `dev`, `prod`
- `CostCenter` — finance code or team name

### Portal

1. Open a resource group → **Tags**.
2. Add the three keys and values → **Apply**.
3. Repeat for each RG (Policy can enforce this later).

### CLI

```bash
az tag create --resource-id $(az group show -n rg-lab-net --query id -o tsv) \
  --tags Owner=you@example.com Environment=lab CostCenter=engineering
```

## Step 3 — Put a delete lock on anything hard to rebuild

Locks stop accidental deletion. Use **CanNotDelete** on shared networking and key vaults; use **ReadOnly** only when you truly mean freeze.

### Portal

1. Open `rg-lab-net` → **Locks** → **Add**.
2. Name: `lock-deny-delete`.
3. Lock type: **Delete**.
4. Notes: why it exists.

### CLI

```bash
az lock create \
  --name lock-deny-delete \
  --lock-type CanNotDelete \
  --resource-group rg-lab-net
```

Remember: locks block deletes and some updates. Remove the lock before intentional teardown.

## Step 4 — Create a Cost Management budget

A subscription without a budget is a surprise invoice waiting to happen.

### Portal

1. Search **Budgets** (under Cost Management) → **Add**.
2. Scope: this subscription.
3. Amount: pick a monthly number you will notice (example: `$200` for a lab).
4. Alert thresholds: **50%**, **90%**, **100%**.
5. Recipients: your email (and a team alias if you have one).
6. Create.

### CLI

```bash
# Create an action group first in the portal, then:
az consumption budget create \
  --budget-name budget-lab-monthly \
  --amount 200 \
  --category Cost \
  --time-grain Monthly \
  --start-date 2025-09-01 \
  --end-date 2026-09-01 \
  --resource-group rg-lab-ops
```

If the CLI shape differs in your cloud shell version, create the budget in the portal once and treat it as the source of truth.

## Step 5 — Separate who can create from who can own

Default trap: everyone is **Owner**. Prefer:

- **Contributor** on workload RGs for builders
- **Owner** or **User Access Administrator** only for a small platform set
- **Reader** for auditors and finance viewers

### Portal

1. Open `rg-lab-compute` → **Access control (IAM)** → **Add role assignment**.
2. Role: **Contributor**.
3. Assign to a user or security group (prefer groups).
4. Do **not** grant Owner unless they must assign roles.

## Step 6 — Activity log sanity check

Confirm you can see who did what:

1. Subscription → **Activity log**.
2. Filter last 24 hours; confirm create events for your RGs.
3. Optionally send Activity log to a Log Analytics workspace (covered in a later how-to).

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| Cannot delete a VNet | Delete lock still on the RG or resource |
| Budget never fires | Alert email wrong, or spend under threshold |
| Tags missing on new resources | Only tagged the RG; child resources need policy or habit |
| Sprawl returns in a week | No RG layout agreement; people create `rg1`, `test`, `new-rg` |

## What you should remember

1. **Few RGs with jobs** beat one mega-group.
2. **Three tags** (Owner, Environment, CostCenter) are enough to start.
3. **Delete locks** protect shared networking and secrets stores.
4. **Budgets** catch labs that grew into production spend.
5. **Owner is rare**; Contributor is the default builder role.

Next upgrades: Azure Policy to require tags, management groups for multi-subscription estates, and PIM for standing Owner.
