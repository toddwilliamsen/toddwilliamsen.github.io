---
layout: article
title: ARM Templates Fundamentals
topic: ARM / IaC
category: arm
diagram: /images/arm-fundamentals.svg
summary: What Azure Resource Manager templates actually are, how a deployment converges, and the building blocks worth learning before the JSON starts nesting itself.
author: Todd Williamsen
date: 2026-07-01
description: A practical, human introduction to ARM templates—schema, parameters, variables, resources, outputs, API versions, and incremental deployment.
permalink: /articles/arm-templates-fundamentals/
---

Infrastructure as Code in Azure starts with a slightly arrogant idea: you write down what you want, hit deploy, and Azure Resource Manager (ARM) makes reality catch up. ARM templates are that wish list in JSON—still the native language under Bicep, a lot of Terraform’s Azure provider behavior, and every “Export template” surprise the portal has ever handed you at 4:47 p.m. on a Friday.

This series goes from fundamentals to modular design, advanced patterns, and the nuances that show up right after your first successful deploy—when you get brave and try a second one.

<figure>
  <img src="{{ '/images/arm-fundamentals.svg' | relative_url }}" alt="ARM template anatomy showing schema, parameters, variables, resources, and outputs feeding a Resource Manager deployment">
  <figcaption>Figure 1. An ARM template is declarative desired state—not a bash script wearing a JSON costume.</figcaption>
</figure>

## Why ARM still matters

Even if you author in Bicep day to day (sensible), the runtime still thinks in ARM:

- Deployment history, what-if, and policy evaluations reason about ARM resources
- Portal exports and half the enterprise modules you’ll inherit still ship as ARM JSON
- Debugging a failed deployment usually means squinting at the ARM expression language like it’s a ransom note

Skipping ARM because “we use Bicep now” is like refusing to learn SQL because the ORM is friendly—until it isn’t.

## Template anatomy

A minimal template has five ideas, and only one of them is where the excitement (and the outages) live:

| Section | Role |
| --- | --- |
| `$schema` / `contentVersion` | Declares the template dialect |
| `parameters` | Inputs that change per environment |
| `variables` | Derived values you do not want callers inventing at 11 p.m. |
| `resources` | The Azure objects to create or update |
| `outputs` | Values the next template or pipeline will beg for |

Everything security-relevant eventually lives in `resources`—or fails to, which is how public storage accounts and open management ports appear while everyone swears the design was Zero Trust.

## Resources are typed contracts

Each resource declares:

- `type` — e.g. `Microsoft.Storage/storageAccounts`
- `apiVersion` — the contract version for that type
- `name` — identity within the scope
- `location` / `properties` — configuration
- optional `dependsOn` — “please create A before B, I am not kidding”

**API version is not cosmetic.** Same resource type, different year, different personality. Pin deliberately. “Latest” is a strategy in the same way “YOLO” is a backup plan.

## Parameters vs variables

Use **parameters** for decisions a human (or pipeline) must make: environment name, region, SKU tier, whether diagnostics are required.

Use **variables** for values computed from parameters: naming conventions, concatenated strings, repeated objects.

If a value is a secret, it does not belong in a casual parameter default or a checked-in `parameters.json`. We’ll get dramatic about that in part 4. For now: `secureString` exists because someone, somewhere, committed `Password123!` and called it temporary.

## Incremental is the default mental model

ARM’s default mode is **incremental**: create or update what’s in the template; leave everything else alone.

That’s usually what you want. It’s also how drift accumulates—one “quick portal fix,” then another, until the template is a polite suggestion and production is a scrapbook.

## A first practical pattern

For any workload template, start with three non-negotiables:

1. **Naming and tags** — owner, cost center, data classification (future-you during an incident will send present-you a thank-you note)
2. **Diagnostics** — logs to a known workspace, not “we’ll add monitoring after go-live” (narrator: they did not)
3. **Network posture** — private endpoints or an *explicit* public access decision—never the accidental default

If those three are optional, your “secure” template is more of a vibe.

## What “done” looks like at this level

You can:

- Read a template and predict what Azure will create
- Separate environment inputs from derived naming
- Explain why `apiVersion` is pinned without hand-waving
- Redeploy the same template without inventing new resources by accident

## Sample template

A small storage account template that makes the boring-but-important decisions explicit: tags, TLS 1.2, public blob access as a parameter (default `false`), and network ACLs that deny by default.

Samples live in **[toddwilliamsen/arm-templates](https://github.com/toddwilliamsen/arm-templates)**:

- [fundamentals/storage-account.json](https://github.com/toddwilliamsen/arm-templates/blob/main/fundamentals/storage-account.json)
- [fundamentals/parameters.dev.json](https://github.com/toddwilliamsen/arm-templates/blob/main/fundamentals/parameters.dev.json)

```json
"properties": {
  "minimumTlsVersion": "TLS1_2",
  "allowBlobPublicAccess": "[parameters('allowBlobPublicAccess')]",
  "supportsHttpsTrafficOnly": true,
  "networkAcls": {
    "defaultAction": "Deny",
    "bypass": "AzureServices"
  }
}
```

```bash
git clone https://github.com/toddwilliamsen/arm-templates.git
cd arm-templates

az deployment group create \
  -g rg-arm-samples \
  -f fundamentals/storage-account.json \
  -p @fundamentals/parameters.dev.json
```

Next up: composing templates so networking, identity, and workloads stay modular—without summoning a single 4,000-line JSON boss fight.

