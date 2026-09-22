---
layout: article
title: "Azure Policy: Audit Then Deny"
topic: How-To
category: how-to
level: Intermediate
diagram: /images/howto-azure-policy-audit-then-deny.svg
summary: Assign a built-in Azure Policy in Audit mode, read compliance results, then flip to Deny so new non-compliant resources fail fast without surprise breakages.
author: Todd Williamsen
date: 2025-09-27
description: Intermediate how-to for Azure Policy audit-then-deny—assign built-in policy, review compliance, change effect to Deny, exemptions, and CLI assignment tips.
permalink: /articles/azure-policy-audit-then-deny/
---

Deny policies without a reconnaissance pass create helpdesk tickets and shadow IT. The safe pattern: **Audit** (or AuditIfNotExists) until you understand the blast radius, fix or exempt what must remain, then **Deny** so new sprawl cannot ignore the rule.

This walkthrough uses a common built-in: require storage accounts to deny public blob access (or similar secure baseline). Adjust the definition ID to the exact built-in you choose in your tenant.

<figure>
  <img src="{{ '/images/howto-azure-policy-audit-then-deny.svg' | relative_url }}" alt="Policy assignment flow from Audit compliance review to Deny">
  <figcaption>Figure 1. Audit first. Read compliance. Then Deny new non-compliant creates.</figcaption>
</figure>

## What you need before you start

- Owner or Resource Policy Contributor on the subscription (or MG)
- A few existing storage accounts to observe (optional but useful)
- About **30–40 minutes**
- Agreement on who handles exemptions

## Step 1 — Find a built-in policy definition

### Portal

1. Search **Policy** → **Definitions**.
2. Filter Type: **Built-in**, Category: **Storage** (example).
3. Pick a clear control, e.g. policies that audit/deny public blob access or require secure transfer.
4. Open it and note the **Definition ID**.

### CLI

```bash
az policy definition list \
  --query "[?contains(displayName, 'public') && contains(displayName, 'Storage')].{Name:displayName, Id:id}" \
  --output table
```

Pick one definition and copy its `id`.

## Step 2 — Assign with effect Audit

### Portal

1. Policy → **Assignments** → **Assign policy**.
2. Scope: your subscription (or a resource group for a narrower lab).
3. Policy definition: the built-in you chose.
4. Parameters: set **effect** to **Audit** if parameterized.
5. Create a managed identity only if the policy needs DeployIfNotExists (skip for pure Audit/Deny).
6. **Review + create**.

### CLI sketch

```bash
DEF="/providers/Microsoft.Authorization/policyDefinitions/YOUR-DEFINITION-GUID"
az policy assignment create \
  --name audit-storage-public-blob \
  --display-name "Audit storage public blob access" \
  --scope "/subscriptions/YOUR-SUB-ID" \
  --policy "$DEF" \
  --params '{"effect":{"value":"Audit"}}'
```

Parameter names vary by definition—check `az policy definition show` for the exact schema.

## Step 3 — Wait for compliance evaluation

Compliance is not always instant.

1. Policy → **Compliance**.
2. Open your assignment.
3. Review **Non-compliant** resources.
4. Optionally trigger a scan: Compliance → **Scan**.

Document every non-compliant resource: fix, replace, or mark for exemption.

## Step 4 — Remediate or exempt

- **Fix**: change the resource to match (disable public blob access).
- **Exempt**: Policy → assignment → **Exemptions** for approved exceptions with an expiry and ticket number.
- Do **not** leave Audit forever and call it governance.

## Step 5 — Flip to Deny

### Portal

1. Open the assignment → **Edit assignment**.
2. Change parameter **effect** to **Deny**.
3. Save.

### Prove it

Try creating a storage account that violates the policy (in a lab). Creation should fail with a policy error. That failure is success.

```bash
# Expect failure when Deny is active and the create violates the rule
az storage account create \
  --resource-group rg-lab-data \
  --name stshouldfail001 \
  --location eastus \
  --sku Standard_LRS \
  --allow-blob-public-access true
```

## Step 6 — Operational habits

1. Prefer assignment at **management group** for org baselines (later how-to).
2. Keep a change window when flipping Audit → Deny on busy subscriptions.
3. Watch Activity log for `Policy` failures after the flip.

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| Always 100% compliant | Wrong scope; or effect not evaluating that property |
| Deny breaks CI suddenly | Flipped without remediating existing automation |
| Cannot create anything storage-related | Over-broad Deny; review parameters and exemptions |
| Compliance stale | Scan not run; wait for next evaluation cycle |

## What you should remember

1. **Audit before Deny** on live subscriptions.
2. **Read compliance** and fix or expire exemptions.
3. **Parameterized effect** makes the flip controlled.
4. **Prove Deny** with a deliberate failing create.
5. **Exemptions need owners and end dates**.

Next: send diagnostics to Log Analytics so policy and resource health share one investigation pane.
