---
layout: article
title: Sentinel Workspace and Your First Analytic Rule
topic: How-To
category: how-to
level: Advanced
diagram: /images/howto-sentinel-first-workspace.svg
summary: Enable Microsoft Sentinel on a Log Analytics workspace, confirm a data connector path, and create a scheduled analytic rule that opens an incident you can triage.
author: Todd Williamsen
date: 2025-10-05
description: Advanced how-to for Sentinel first workspace—enable SIEM on LAW, connector basics, scheduled analytic rule, incident validation, and cost awareness.
permalink: /articles/sentinel-first-workspace/
---

Log Analytics stores data. Sentinel turns it into detections and incidents. This walkthrough enables Sentinel on an existing workspace, notes a connector, and creates one scheduled analytic rule so you see a full alert → incident path.

<figure>
  <img src="{{ '/images/howto-sentinel-first-workspace.svg' | relative_url }}" alt="Log Analytics with Sentinel analytic rule creating an incident">
  <figcaption>Figure 1. Workspace data in. Analytic rule evaluates. Incident out.</figcaption>
</figure>

## What you need before you start

- Log Analytics workspace with some logs (Activity log or Entra sign-ins recommended)
- Rights to enable Sentinel (Microsoft Sentinel Contributor-style permissions)
- About **40 minutes**
- Awareness that Sentinel has licensing/pricing beyond raw LAW ingest

## Step 1 — Enable Sentinel on the workspace

### Portal

1. Search **Microsoft Sentinel**.
2. **Create** / add Sentinel to `law-lab-001`.
3. Confirm the Sentinel blade opens on that workspace.

### CLI

```bash
LAW_ID=$(az monitor log-analytics workspace show -g rg-lab-ops -n law-lab-001 --query id -o tsv)
az sentinel workspace create \
  --resource-group rg-lab-ops \
  --workspace-name law-lab-001
```

If the CLI extension surface differs, use the portal enablement once—functionally you are onboarding Sentinel to the LAW.

## Step 2 — Confirm data is present

Sentinel → **Logs** (or LAW Logs):

```kusto
AzureActivity
| where TimeGenerated > ago(1d)
| take 10
```

Or for Entra (if connected):

```kusto
SigninLogs
| where TimeGenerated > ago(1d)
| take 10
```

No data means fix connectors/diagnostic settings before writing rules.

## Step 3 — Enable a content path (connector / content hub)

### Portal

1. Sentinel → **Content management** → **Content hub** (or **Data connectors**).
2. Install a starter pack relevant to your data (Azure Activity, Entra ID).
3. Open the connector page and complete any connection prerequisites shown.

You do not need every connector. One clean source beats twenty empty tiles.

## Step 4 — Create a scheduled analytic rule

Example: noisy but educational rule on failed Azure Activity operations (tune for your tenant).

### Portal

1. Sentinel → **Analytics** → **+ Create** → **Scheduled query rule**.
2. Name: `LAB - AzureActivity Failures`.
3. Severity: Low (lab).
4. Rule query:

```kusto
AzureActivity
| where ActivityStatusValue == "Failed"
| where TimeGenerated > ago(5m)
| summarize FailureCount=count() by Caller, OperationNameValue, bin(TimeGenerated, 5m)
| where FailureCount >= 1
```

5. Entity mapping: map `Caller` to Account if offered.
6. Query scheduling: every **5 minutes**, lookback **5 minutes** (lab-friendly).
7. Alert threshold: greater than 0.
8. Incident settings: **Create incidents** from alerts enabled.
9. Create.

Tighten thresholds before production; this rule is intentionally chatty for learning.

## Step 5 — Generate traffic and open the incident

1. Perform a failing Azure operation as a test user (denied action, bad deploy).
2. Wait for the schedule cycle.
3. Sentinel → **Incidents** → open the new incident.
4. Confirm entities, raw events, and owner assignment workflow.

## Step 6 — Automation note (optional)

Automation rules can assign owner or run a playbook. Skip until one manual triage feels natural—otherwise you automate noise.

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| Rule never fires | No matching data; schedule not elapsed; query logic wrong |
| Incidents flood | Threshold too low for production volumes |
| Empty connector | Permissions or diagnostic settings incomplete |
| Cost spike | High-churn logs + many rules + long retention |

## What you should remember

1. **Sentinel rides on LAW**—fix ingest first.
2. **One connector well** beats many half-connected.
3. **Scheduled rule → incident** is the core loop to learn.
4. **Tune severity and thresholds** before wide enablement.
5. **Cost and noise** are security problems too.

Next: Defender for Cloud plans and secure score so posture findings feed the same operational muscle.
