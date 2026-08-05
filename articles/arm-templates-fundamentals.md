---
layout: article
title: ARM Templates Fundamentals
topic: IaC Series · Part 1
summary: What Azure Resource Manager templates actually are, how a deployment converges, and the building blocks you need before anything advanced matters.
author: Todd Williamsen
date: 2026-07-01
description: A practical introduction to ARM templates—schema, parameters, variables, resources, outputs, API versions, and incremental deployment.
permalink: /articles/arm-templates-fundamentals/
---

Infrastructure as Code in Azure starts with a simple contract: you describe the desired state, and Azure Resource Manager (ARM) converges the subscription toward it. ARM templates are that contract in JSON—still the native deployment language under Bicep, Terraform providers, and portal exports.

This series walks from fundamentals through modular design, advanced patterns, and the nuances that show up after the first successful deploy.

<figure>
  <img src="{{ '/images/arm-fundamentals.svg' | relative_url }}" alt="ARM template anatomy showing schema, parameters, variables, resources, and outputs feeding a Resource Manager deployment">
  <figcaption>Figure 1. An ARM template is declarative desired state—not an imperative script.</figcaption>
</figure>

## Why ARM still matters

Even if you author in Bicep day to day, the runtime still thinks in ARM:

- Deployment history, what-if, and policy evaluations reason about ARM resources
- Portal “Export template” and many enterprise modules still ship as ARM JSON
- Debugging a failed deployment usually means reading the ARM expression language

Understanding ARM is understanding Azure’s control plane.

## Template anatomy

A minimal template has five ideas:

| Section | Role |
| --- | --- |
| `$schema` / `contentVersion` | Declares the template dialect |
| `parameters` | Inputs that change per environment |
| `variables` | Derived values you do not want callers to invent |
| `resources` | The Azure objects to create or update |
| `outputs` | Values other templates or pipelines need next |

Everything security-relevant eventually lives in `resources`—or fails to, which is how public storage accounts and open management ports appear under delivery pressure.

## Resources are typed contracts

Each resource declares:

- `type` — e.g. `Microsoft.Storage/storageAccounts`
- `apiVersion` — the contract version for that type
- `name` — identity within the scope
- `location` / `properties` — configuration
- optional `dependsOn` — explicit ordering

**API version is not cosmetic.** The same resource type can accept different properties, defaults, and validation rules across versions. Pin deliberately; “latest” is not a strategy.

## Parameters vs variables

Use **parameters** for decisions the caller must make: environment name, region, SKU tier, whether diagnostics are required.

Use **variables** for values computed from parameters: naming conventions, concatenated strings, repeated objects.

If a value is a secret, it is neither a casual parameter default nor a checked-in `parameters.json` entry—that nuance gets its own article later. For now: `secureString` exists for a reason.

## Incremental is the default mental model

ARM’s default deployment mode is **incremental**: resources in the template are created or updated; resources that exist in the resource group but are absent from the template are left alone.

That is usually what you want for day-to-day delivery. It is also how drift accumulates when people click in the portal and never bring those changes back into source control.

## A first practical pattern

For any workload template, start with three non-negotiables:

1. **Naming and tags** — owner, cost center, data classification
2. **Diagnostics** — logs to a known workspace, not “we’ll add later”
3. **Network posture** — private endpoints or explicit public access decisions, never accidental defaults

If those three are optional, the “secure” template is a suggestion, not a platform.

## What “done” looks like at this level

You can:

- Read a template and predict what Azure will create
- Separate environment inputs from derived naming
- Explain why `apiVersion` is pinned
- Redeploy the same template safely (idempotent intent)

Next: composing templates so networking, identity, and workloads stay modular without turning into a JSON swamp.

---

**IaC Series:** Part 1 of 4 · [Next: Modular ARM Design →]({{ '/articles/arm-modular-design/' | relative_url }})
