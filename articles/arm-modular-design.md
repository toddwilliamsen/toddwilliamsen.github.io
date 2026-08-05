---
layout: article
title: Modular ARM Design
topic: IaC Series · Part 2
summary: Parameters, dependencies, nested and linked templates—and how to stop shipping one gigantic JSON file that only one person understands (and that person is on PTO).
author: Todd Williamsen
date: 2026-07-08
description: Modular ARM patterns with a human tone—parameters files, dependsOn, reference outputs, and nested or linked templates for network, identity, and workload composition.
permalink: /articles/arm-modular-design/
---

A single mega-template works beautifully—right up until the second team needs a subnet, a Key Vault reference, and a different SKU before lunch. Suddenly your “simple” file has opinions about networking, identity, compute, and whether Bastion deserves to exist. Modular ARM is how you keep composition explicit: parameters in, one concern per module, outputs to chain the next stage without telepathy.

<figure>
  <img src="{{ '/images/arm-modular.svg' | relative_url }}" alt="Modular ARM composition with parameters feeding a main template that links network, identity, and workload templates">
  <figcaption>Figure 1. Main template conducts; linked templates play one instrument each.</figcaption>
</figure>

## One concern per module

Split along operational boundaries, not “this file is getting long and my scrollbar is sad”:

- **Network module** — VNet, subnets, NSGs, private DNS links
- **Identity / access module** — managed identities, role assignments
- **Workload module** — app service, VM scale set, storage, databases
- **Observability module** — diagnostic settings, alerts, the stuff people skip until the outage

If one module both creates a VNet *and* hands out subscription Owner, congratulations: you’ve mixed blast radii. That’s not modularity—that’s a plot twist.

## Parameters files are environment contracts

Treat `parameters.dev.json` / `parameters.prod.json` like reviewed artifacts, not sticky notes:

- Same template, different inputs
- No secrets in the file (Git has a long memory and no chill)
- Explicit SKUs and feature flags (`enablePrivateEndpoints`, `deployBastion`)

If “prod” and “dev” differ only by a comment in a PR—“remember to turn on private endpoints”—the contract is incomplete. Azure will not read the comment. Azure will deploy the vibe.

## Dependencies without spaghetti

ARM needs ordering when resource B cannot exist before resource A. Fair.

Prefer:

- **`dependsOn`** for hard ordering you own in the same template
- **`reference()` / `resourceId()`** for reading runtime values
- **module outputs** for cross-template chaining

Avoid depending on everything “just in case.” Over-declaring dependencies slows deployments and hides real coupling—like listing every coworker as an emergency contact.

## Nested vs linked templates

**Nested templates** embed another deployment inside the parent. Handy for scoped logic and conditional sub-graphs.

**Linked templates** pull an external template URI (storage account, pipeline artifact, that shared modules repo someone finally created). Better for reuse and versioning modules on their own schedule.

In enterprise pipelines, linked templates usually win: version once, consume many times, promote on purpose—not because someone copied a file into seven repos and changed one of them.

## Outputs are the integration API

Anything another system needs should be an output:

- Subnet IDs
- Principal IDs for managed identities
- Private endpoint details
- Workspace resource IDs

Hard-coding resource names across modules recreates the mega-template—you just distributed the chaos into four files and called it architecture.

## A composition pattern that holds

1. Deploy shared network once per landing subscription
2. Output subnet and DNS identifiers
3. Workload modules consume those outputs as parameters
4. Identity assignments happen *after* principals exist (race conditions are real; part 4 has the scars)

## Design checks

- Can you redeploy the network module without rewriting the app module?
- Can a second workload reuse the same network outputs?
- Are environment differences expressed only through parameters?
- Are module boundaries aligned to who gets paged when something breaks? (If the answer is “everyone,” the boundaries are decorative.)

## Sample templates

Two files in **[toddwilliamsen/arm-templates](https://github.com/toddwilliamsen/arm-templates)** that show the composition idea without requiring a private module registry:

- [modular/network.json](https://github.com/toddwilliamsen/arm-templates/blob/main/modular/network.json) — VNet module with subnet `copy` and outputs
- [modular/main.json](https://github.com/toddwilliamsen/arm-templates/blob/main/modular/main.json) — nests a network deployment, then deploys storage and surfaces `vnetId` via outputs

The interesting bit in `main.json` is treating nested deployment outputs as the integration API:

```json
"outputs": {
  "vnetId": {
    "type": "string",
    "value": "[reference('deploy-network').outputs.vnetId.value]"
  }
}
```

In a real pipeline you’d usually **link** `network.json` from a versioned artifact URI instead of inlining—same contract, less copy-paste.

Next: copy loops, conditions, deployment modes, what-if, and the special joy of complete mode.

---

**IaC Series:** [← ARM Fundamentals]({{ '/articles/arm-templates-fundamentals/' | relative_url }}) · Part 2 of 4 · [Next: Advanced ARM Patterns →]({{ '/articles/arm-advanced-patterns/' | relative_url }})
