---
layout: article
title: Send Azure Diagnostics to Log Analytics
topic: How-To
category: how-to
level: Intermediate
diagram: /images/howto-diagnostics-to-log-analytics.svg
summary: Create a Log Analytics workspace and wire diagnostic settings from a Key Vault, NSG, or Activity log so you can query real platform signals with Kusto.
author: Todd Williamsen
date: 2025-09-28
description: Intermediate how-to sending Azure diagnostics to Log Analytics—create workspace, diagnostic settings, Activity log, sample KQL, and retention notes.
permalink: /articles/diagnostics-to-log-analytics/
---

Resources without diagnostic settings are mute during incidents. This walkthrough creates a Log Analytics workspace, attaches diagnostic settings from a sample resource (Key Vault) and the subscription Activity log, then runs a first query.

<figure>
  <img src="{{ '/images/howto-diagnostics-to-log-analytics.svg' | relative_url }}" alt="Resources and Activity log flowing into a Log Analytics workspace">
  <figcaption>Figure 1. Diagnostic settings push platform logs into one workspace you can query.</figcaption>
</figure>

## What you need before you start

- Resource group `rg-lab-ops`
- A resource that supports diagnostics (Key Vault, NSG, Storage, etc.)
- Rights to create Operational Insights / Log Analytics
- About **30 minutes**

## Step 1 — Create a Log Analytics workspace

### Portal

1. Search **Log Analytics workspaces** → **Create**.
2. Resource group: `rg-lab-ops`.
3. Name: `law-lab-001`.
4. Region: same region as most resources when possible.
5. Pricing: pay-as-you-go (watch ingest cost in busy tenants).
6. **Review + create** → **Create**.

### CLI

```bash
az monitor log-analytics workspace create \
  --resource-group rg-lab-ops \
  --workspace-name law-lab-001 \
  --location eastus
```

## Step 2 — Diagnostic settings on Key Vault

### Portal

1. Open Key Vault → **Diagnostic settings** → **Add diagnostic setting**.
2. Name: `diag-to-law`.
3. Categories: at least **AuditEvent** (and metrics if you want).
4. Destination: **Send to Log Analytics workspace** → `law-lab-001`.
5. Save.

### CLI

```bash
LAW_ID=$(az monitor log-analytics workspace show \
  -g rg-lab-ops -n law-lab-001 --query id -o tsv)
KV_ID=$(az keyvault show -g rg-lab-data -n kv-lab-toddw001 --query id -o tsv)

az monitor diagnostic-settings create \
  --name diag-to-law \
  --resource "$KV_ID" \
  --workspace "$LAW_ID" \
  --logs '[{"category":"AuditEvent","enabled":true}]'
```

Category names differ by resource type—use the portal once to learn the list, then automate.

## Step 3 — Send Activity log to the same workspace

Subscription Activity log catches control-plane changes (who created what).

### Portal

1. **Monitor** → **Activity log** → **Export Activity Logs** (or Diagnostic settings at subscription).
2. Add diagnostic setting → Categories: Administrative (minimum); add Security, Policy, Alert as needed.
3. Destination: `law-lab-001`.
4. Save.

### CLI

```bash
SUB=$(az account show --query id -o tsv)
az monitor diagnostic-settings subscription create \
  --name activity-to-law \
  --location global \
  --workspace "$LAW_ID" \
  --logs '[{"category":"Administrative","enabled":true}]'
```

Exact subscription diagnostic CLI syntax can vary; portal is reliable for the first wire-up.

## Step 4 — Generate a signal and query

1. Read a Key Vault secret (portal or CLI) to generate AuditEvent.
2. Open `law-lab-001` → **Logs**.
3. Dismiss the schema splash if shown; run:

```kusto
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.KEYVAULT"
| project TimeGenerated, OperationName, ResultType, CallerIPAddress, identity_claim_upn_s
| order by TimeGenerated desc
| take 20
```

Activity log table (when configured):

```kusto
AzureActivity
| where TimeGenerated > ago(1d)
| project TimeGenerated, Caller, OperationNameValue, ActivityStatusValue, ResourceGroup
| order by TimeGenerated desc
| take 20
```

If results are empty, wait a few minutes—ingest is near-real-time but not always instant.

## Step 5 — Retention and cost hygiene

1. Workspace → **Usage and estimated costs**.
2. Set interactive retention appropriately (default may be fine for labs).
3. Do not enable every category on every resource on day one—start with audit/security-relevant logs.

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| Empty `AzureDiagnostics` | Wrong workspace; category not enabled; wait for ingest |
| Huge bill | All metrics + verbose logs on high-churn resources |
| Query column missing | Resource-specific columns; use `search` or schema explorer |
| Activity missing | Subscription diagnostic setting never created |

## What you should remember

1. **One workspace per lab/env** is enough to start.
2. **Diagnostic settings** are per resource (plus subscription Activity log).
3. **Audit/security categories first**, vanity metrics later.
4. **Prove with a query** after generating traffic.
5. **Watch ingest cost** as you scale categories.

Next: private endpoint for Storage, then Sentinel on top of this workspace for analytic rules.
