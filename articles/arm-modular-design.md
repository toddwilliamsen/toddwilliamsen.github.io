---
layout: article
title: Modular ARM Design
topic: IaC Series · Part 2
summary: Parameters, dependencies, nested and linked templates, and how to compose Azure infrastructure without a single unmaintainable JSON file.
author: Todd Williamsen
date: 2026-07-08
description: Modular ARM patterns using parameters files, dependsOn, reference outputs, and nested or linked templates for network, identity, and workload composition.
permalink: /articles/arm-modular-design/
---

A single mega-template works until the second team needs a subnet, a Key Vault reference, and a different SKU in the same weekend. Modular ARM is how you keep composition explicit: parameters in, modules for one concern each, outputs to chain the next stage.

<figure>
  <img src="{{ '/images/arm-modular.svg' | relative_url }}" alt="Modular ARM composition with parameters feeding a main template that links network, identity, and workload templates">
  <figcaption>Figure 1. Main template orchestrates; linked templates own one concern.</figcaption>
</figure>

## One concern per module

Split along operational boundaries, not file-size anxiety:

- **Network module** — VNet, subnets, NSGs, private DNS links
- **Identity / access module** — managed identities, role assignments
- **Workload module** — app service, VM scale set, storage, databases
- **Observability module** — diagnostic settings, alerts, workbook hooks

If a module both creates a VNet *and* assigns subscription Owner, you have mixed blast radii.

## Parameters files are environment contracts

Treat `parameters.dev.json` / `parameters.prod.json` as reviewed artifacts:

- Same template, different inputs
- No secrets in the file
- Explicit SKUs and feature flags (`enablePrivateEndpoints`, `deployBastion`)

The template should refuse ambiguous environments. If “prod” and “dev” differ only by a comment in a PR description, the contract is incomplete.

## Dependencies without spaghetti

ARM needs ordering when resource B cannot exist before resource A.

Prefer:

- **`dependsOn`** for hard ordering you own in the same template
- **`reference()` / `resourceId()`** for reading runtime values
- **module outputs** for cross-template chaining

Avoid depending on everything “just in case.” Over-declaring dependencies slows deployments and hides real coupling.

## Nested vs linked templates

**Nested templates** embed another template deployment resource inside the parent. Useful for scoped logic and conditional sub-graphs.

**Linked templates** pull an external template URI (often from a storage account or repository artifact). Better for reuse across products and versioning modules independently.

In enterprise pipelines, linked templates usually win: version the module once, consume many times, promote intentionally.

## Outputs are the integration API

Anything another system needs should be an output:

- Subnet IDs
- Principal IDs for managed identities
- Private endpoint NIC details
- Workspace resource IDs

Hard-coding resource names across modules recreates the mega-template, just in four files.

## A composition pattern that holds

1. Deploy shared network once per landing subscription
2. Output subnet and DNS identifiers
3. Workload modules consume those outputs as parameters
4. Identity assignments happen after principals exist (race conditions are real—covered in nuances)

## Design checks

- Can you redeploy the network module without rewriting the app module?
- Can a second workload reuse the same network outputs?
- Are environment differences expressed only through parameters?
- Are module boundaries aligned to who gets paged when something breaks?

Next: copy loops, conditions, deployment modes, what-if, and subscription-scoped orchestration.

---

**IaC Series:** [← ARM Fundamentals]({{ '/articles/arm-templates-fundamentals/' | relative_url }}) · Part 2 of 4 · [Next: Advanced ARM Patterns →]({{ '/articles/arm-advanced-patterns/' | relative_url }})
