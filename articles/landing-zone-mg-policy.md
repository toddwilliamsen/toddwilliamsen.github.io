---
layout: article
title: "Landing Zone Starter: Management Groups and Policy"
topic: How-To
category: how-to
level: Advanced
diagram: /images/howto-landing-zone-mg-policy.svg
summary: Create a simple management group tree, nest subscriptions, and assign a baseline Azure Policy at the MG so every child subscription inherits guardrails.
author: Todd Williamsen
date: 2025-10-03
description: Advanced how-to landing zone starter with management groups and policy—MG hierarchy, move subscription, assign policy at MG, inheritance checks.
permalink: /articles/landing-zone-mg-policy/
---

Subscriptions without a management group tree become snowflakes. A starter landing zone is not CAF Enterprise Scale overnight—it is a **sensible MG hierarchy** plus a few **policies at the parent** so identity, security, and platform teams share one inheritance path.

<figure>
  <img src="{{ '/images/howto-landing-zone-mg-policy.svg' | relative_url }}" alt="Tenant root to platform and landing zones management groups with policy inheritance">
  <figcaption>Figure 1. Management groups parent subscriptions. Policy at the MG flows down.</figcaption>
</figure>

## What you need before you start

- Permissions to create management groups (often Management Group Contributor / User Access Administrator at tenant root)
- At least one subscription you can move
- About **40–50 minutes**
- Change window: moving subscriptions can unsettle teams if surprise

## Step 1 — Sketch a tiny hierarchy

Example starter (names are yours to choose):

```text
Tenant Root Group
  mg-contoso-root
    mg-platform
    mg-landing-zones
      mg-corp
      mg-sandboxes
```

Keep depth shallow until you need more.

## Step 2 — Create management groups

### Portal

1. Search **Management groups**.
2. **Create** under Tenant Root (or your org’s intermediate root).
3. Create `mg-contoso-root`, then children `mg-platform` and `mg-landing-zones`, then `mg-corp` and `mg-sandboxes` under landing zones.

### CLI

```bash
az account management-group create --name mg-contoso-root --display-name "Contoso Root"
az account management-group create --name mg-platform --display-name "Platform" --parent mg-contoso-root
az account management-group create --name mg-landing-zones --display-name "Landing Zones" --parent mg-contoso-root
az account management-group create --name mg-corp --display-name "Corp" --parent mg-landing-zones
az account management-group create --name mg-sandboxes --display-name "Sandboxes" --parent mg-landing-zones
```

## Step 3 — Move a subscription under a MG

### Portal

1. Management groups → select `mg-sandboxes` → **Add subscription**.
2. Or open the subscription → **Parent management group** change (wording varies).

### CLI

```bash
SUB=$(az account show --query id -o tsv)
az account management-group subscription add \
  --name mg-sandboxes \
  --subscription "$SUB"
```

Confirm the subscription appears under `mg-sandboxes` in the hierarchy view.

## Step 4 — Assign a baseline policy at the MG

Pick a low-risk Audit policy first (tags required, allowed locations, storage public access audit).

### Portal

1. **Policy** → **Assignments** → **Assign policy**.
2. Scope: select management group `mg-sandboxes` (or `mg-contoso-root` for wider blast radius—start narrow).
3. Definition: a built-in Audit policy you already understand.
4. Effect: **Audit**.
5. Create.

Inheritance: child subscriptions should show the assignment under Policy compliance.

### CLI sketch

```bash
az policy assignment create \
  --name audit-require-tag-owner \
  --display-name "Audit require Owner tag" \
  --scope "/providers/Microsoft.Management/managementGroups/mg-sandboxes" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/YOUR-DEFINITION-GUID"
```

## Step 5 — Verify inheritance

1. Open the nested subscription → **Policy** → **Assignments**.
2. Confirm the MG assignment is visible.
3. Create a non-compliant resource (for Audit) and watch Compliance.

When ready, follow the audit-then-deny pattern at MG scope—carefully, with exemptions process defined.

## Step 6 — RBAC at MG (light touch)

Assign **Reader** at `mg-landing-zones` for auditors, and keep Owner-like roles rare and PIM-eligible. MG RBAC multiplies power—treat it like tenant-level privilege.

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| Cannot create MG | Missing tenant-level permissions |
| Subscription move fails | Existing MG permissions / quotas / policies |
| Policy not visible | Wrong scope; or viewing different tenant directory |
| Accidental Deny at root | Assigned Deny too high—start sandboxes MG |

## What you should remember

1. **Shallow MG tree** beats a perfect diagram you never finish.
2. **Sandboxes vs corp** separation early.
3. **Policy at MG** inherits to subscriptions.
4. **Audit first** at MG the same as at subscription.
5. **MG Owner is powerful**—PIM and break-glass still apply.

Next: Conditional Access for Azure portals/CLI so the humans under this tree authenticate the way you expect.
