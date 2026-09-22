---
layout: article
title: Conditional Access for Azure Portals and CLI
topic: How-To
category: how-to
level: Advanced
diagram: /images/howto-conditional-access-azure-portals.svg
summary: Build a Conditional Access policy that requires MFA (and optionally compliant devices) for Azure management apps, while excluding break-glass accounts.
author: Todd Williamsen
date: 2025-10-04
description: Advanced how-to Conditional Access for Azure portal and CLI—target Microsoft Azure Management, MFA, break-glass exclude, report-only then on.
permalink: /articles/conditional-access-azure-portals/
---

Azure RBAC does not care how someone authenticated—only that they have a token. Conditional Access closes that gap for **Azure management**: portal, Azure CLI, PowerShell, and related resource manager clients.

Have break-glass accounts ready and excluded before you enforce. Report-only first.

<figure>
  <img src="{{ '/images/howto-conditional-access-azure-portals.svg' | relative_url }}" alt="Conditional Access on Azure Management with MFA and break-glass exclusion">
  <figcaption>Figure 1. Azure management apps require MFA. Emergency accounts stay excluded and monitored.</figcaption>
</figure>

## What you need before you start

- Entra ID P1/P2 for CA
- Conditional Access Administrator (or Global Admin)
- Working **break-glass** accounts + exclude group
- About **35–45 minutes**
- A pilot security group of admins

## Step 1 — Create a pilot group

1. Entra → **Groups** → New security group `sg-ca-azure-admins-pilot`.
2. Add a few volunteer admins (not the whole company).

## Step 2 — Create the policy in report-only

### Portal

1. Entra → **Protection** → **Conditional Access** → **Create new policy**.
2. Name: `CA-Azure-Management-MFA`.
3. **Users**: Include `sg-ca-azure-admins-pilot`. **Exclude** `sg-break-glass`.
4. **Target resources** (Cloud apps): select **Microsoft Azure Management** (covers portal / Resource Manager clients used for Azure management).
5. **Grant**: **Require multifactor authentication**. Optionally add compliant device later—not on day one for CLI users without device join.
6. **Enable policy**: **Report-only**.
7. Create.

Exact app display names can vary slightly; search “Azure Management” in the cloud apps picker and confirm Microsoft’s current label.

## Step 3 — Exercise portal and CLI

As a pilot user:

```bash
az logout
az login
az group list -o table
```

Also open https://portal.azure.com and perform a read-only action.

Then open CA **Insights and reporting** / sign-in logs → Conditional Access tab. Confirm the policy would apply (report-only results).

## Step 4 — Fix false positives before On

Common issues:

- Service accounts / automation using user credentials (move to managed identity / workload identity)
- Break-glass accidentally included
- Guest admins without MFA methods registered

Register MFA methods for pilots **before** flipping to On.

## Step 5 — Turn On for the pilot

1. Edit policy → Enable: **On**.
2. Retest `az login` and portal—MFA should challenge as configured.
3. Expand Include users from pilot group to all Azure admins (or all users if that is your standard)—slowly.

## Step 6 — Optional hardening (second policy)

After MFA is boring:

- Require **phishing-resistant** MFA for privileged directory roles
- Block legacy auth tenant-wide (separate policy)
- Location conditions only if you truly understand travel and cloud shell egress

Do not stack five conditions in the first policy.

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| Locked out of portal | Break-glass not excluded; or MFA not registered |
| CLI fails, portal works | Different client; check Azure Management app targeting |
| Automation broken | User-based SP passwords under CA; move to OIDC/MI |
| Report-only empty | User not in Include; or wrong cloud app |

## What you should remember

1. **Break-glass excluded** before enforce.
2. **Report-only** with real pilot sign-ins first.
3. Target **Microsoft Azure Management** for portal/CLI-style management.
4. **MFA first**; device compliance second.
5. **Automation identities** are not human CA targets—use workload identity.

Next: Sentinel on your Log Analytics workspace so CA and Azure activity become detections, not just logs.
