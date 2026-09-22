---
layout: article
title: PIM for Azure RBAC from Scratch
topic: How-To
category: how-to
level: Intermediate
diagram: /images/howto-pim-azure-rbac-from-scratch.svg
summary: Move standing Azure Owner/Contributor into Privileged Identity Management eligible assignments so admins activate just-in-time with MFA and a reason code.
author: Todd Williamsen
date: 2025-09-30
description: Intermediate how-to for PIM on Azure RBAC—eligible Contributor, activation settings, MFA, remove standing roles, and activation walkthrough.
permalink: /articles/pim-azure-rbac-from-scratch/
---

Standing Owner on a subscription is convenient until it is compromised. Privileged Identity Management (PIM) makes privileged Azure roles **eligible**: users activate for a limited window with MFA and justification, then lose the role again.

Requires Microsoft Entra ID P2 or Governance licenses for the users in scope.

<figure>
  <img src="{{ '/images/howto-pim-azure-rbac-from-scratch.svg' | relative_url }}" alt="Eligible Azure role activates JIT with MFA then expires">
  <figcaption>Figure 1. Eligible until needed. Activate with MFA. Expires automatically.</figcaption>
</figure>

## What you need before you start

- Entra ID P2 (or trial) for participating admins
- Privilege to configure PIM for Azure resources (Privileged Role Administrator / Owner patterns vary—use an account that can manage the subscription)
- About **35 minutes**
- A test user who currently has standing Contributor (lab)

## Step 1 — Open PIM for Azure resources

### Portal

1. Entra admin center → **Identity Governance** → **Privileged Identity Management**.
2. Select **Azure resources**.
3. Discover/manage the subscription if prompted (PIM may need to onboard the subscription once).
4. Open your subscription → **Roles**.

## Step 2 — Make Contributor eligible (not active)

1. Select **Contributor** → **Add assignments**.
2. Assignment type: **Eligible**.
3. Select members: your test admin user or group.
4. Duration: eligible permanently for a lab, or time-bound for production hygiene.
5. Assign.

Repeat for **Owner** or **User Access Administrator** only for the few who need them—prefer Contributor-eligible for builders.

## Step 3 — Configure activation requirements

1. Still under the role → **Settings** → **Edit**.
2. Activation maximum duration: e.g. **8 hours**.
3. Require **MFA** on activation.
4. Require **justification**.
5. Optional: require approval for Owner activation (good for production).
6. Update.

## Step 4 — Remove standing assignments

PIM eligibility does nothing if the user still has a permanent IAM assignment.

1. Subscription → **Access control (IAM)**.
2. Remove the standing **Contributor** assignment for that user.
3. Confirm they only show as eligible in PIM.

## Step 5 — Activate as the user

Sign in as the eligible user:

1. Azure portal → search **Privileged Identity Management** → **My roles** → **Azure resources**.
2. Eligible assignment → **Activate**.
3. Complete MFA; enter a ticket/reason.
4. Wait for activation (seconds to a minute).
5. Perform the privileged work; confirm role disappears after the window.

CLI activation exists via Microsoft Graph / PIM APIs; portal is enough for a first lab.

## Step 6 — Audit

1. PIM → Azure resource → **Resource audit** / **Assignments** history.
2. Optionally send Entra audit logs to Log Analytics and alert on frequent activations.

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| Activate succeeds but portal still Forbidden | Standing role removed but activation not finished; or wrong subscription scope |
| PIM menu missing | License not assigned; or wrong admin center blade |
| User never needs to activate | Standing IAM assignment still present |
| Approvals stuck | Approver group empty or users offline—have a backup approver |

## What you should remember

1. **Eligible > standing** for Owner/Contributor on shared subscriptions.
2. **Remove IAM permanent roles** after eligibility exists.
3. **MFA + justification** on activation.
4. **Short windows** beat all-day standing elevation.
5. **Audit activations** like break-glass sign-ins.

Next: encode the resource baseline in Bicep so privileged creates are also repeatable.
