---
layout: article
title: Advanced ARM Patterns
topic: IaC Series · Part 3
summary: Copy loops, conditions, what-if, and incremental vs complete mode—advanced ARM with fewer clever tricks and fewer 2 a.m. rollbacks.
author: Todd Williamsen
date: 2026-07-15
description: Advanced Azure ARM techniques with a human tone—copy, conditions, deployment scripts, what-if, and incremental versus complete deployment modes.
permalink: /articles/arm-advanced-patterns/
---

Once modules exist, the next problems are repetition, optionality, and the quiet terror of deploying to production without knowing what will change. Advanced ARM is less “look what JSON can do” and more “please don’t delete the wrong resource group.”

<figure>
  <img src="{{ '/images/arm-advanced.svg' | relative_url }}" alt="Advanced ARM patterns for copy loops, conditions, deployment modes, and a validation layer with what-if">
  <figcaption>Figure 1. Advanced patterns should increase control—not win a code-golf contest.</figcaption>
</figure>

## Copy loops: repetition with intent

Use `copy` when you need N of the same shape:

- Subnets from an array of CIDR definitions
- NIC / disk attachments
- Role assignments across a list of principal IDs
- Diagnostic settings across a resource list (the grown-up move)

Watch for:

- **Serial vs parallel** copy mode when Azure has creation limits or ordering needs
- **Name uniqueness** — collisions fail loudly in CI and mysteriously in “it worked on my machine” retries
- **Index expressions** that look like a crossword puzzle; prefer named objects in arrays

If the loop encodes business logic nobody can explain in a standup, that logic wants to be parameters—not interpretive dance inside `copyIndex()`.

## Conditions: deploy only when required

`condition` keeps prod-only controls out of non-prod without maintaining two templates that slowly become distant cousins:

- Private Endpoints only when `usePrivateNetwork` is true
- Diagnostic settings only when a workspace ID actually exists
- Skip Bastion in sandboxes (your finance team will notice; so will attackers, eventually)

Conditionals should be boring. Nested conditions that recreate a programming language inside JSON are a maintenance tax with interest.

## Deployment scopes beyond the resource group

ARM can deploy at:

- **Resource group** — most app workloads
- **Subscription** — role assignments, diagnostics at scale, platform glue
- **Management group** — guardrails that must inherit whether app teams feel like it or not

Subscription and MG scoped templates are how platform teams encode “every subscription gets logging and these deny policies” without relying on hope as a control.

## Incremental vs complete mode

| Mode | Behavior | Typical use |
| --- | --- | --- |
| Incremental | Create/update what’s in the template; leave the rest alone | Day-to-day app delivery |
| Complete | Make the scope match the template; remove extras | Tightly controlled platform RGs |

**Complete mode’s blast radius is the scope.** A missing resource in the template is a delete. That’s powerful when you own the whole resource group. It’s a horror movie when someone else’s “temporary” jump box counts as drift.

Use complete mode only when the pipeline fully owns the scope—and you’ve run what-if, unless you enjoy storytelling in the incident channel.

## What-if before promote

`az deployment group what-if` answers the only question that matters before prod: *what would actually change?*

Make it a gate:

1. Validate template
2. What-if against the target scope
3. Human review on destructive or “wait, why is this public?” changes
4. Deploy
5. Redeploy once more to confirm you’re not inventing resources for sport

If what-if is optional, production surprises aren’t bugs—they’re calendar invites.

## Deployment Scripts when ARM is not enough

Some bootstrap steps are awkward in pure declarative form: seeding Key Vault from a pipeline-managed rotation, one-time DNS glue, calling an API Azure doesn’t model cleanly.

**Deployment Scripts** run during deployment and can save the day. They can also become hidden imperative snowflakes that only one engineer trusts. Prefer native resources when they exist. Use scripts when you must—and document them like the landmine they are.

## A promotion path that scales

- **Dev** — iterate fast; complete mode allowed on disposable RGs
- **Canary** — what-if required; incremental only
- **Prod** — what-if plus a change record for network, identity, or public-access diffs

Advanced ARM isn’t collecting every language feature like Pokémon. It’s repeatable convergence with fewer pages at 2 a.m.

## Sample template

[vnet-copy-and-condition.json]({{ '/samples/arm/advanced/vnet-copy-and-condition.json' | relative_url }}) builds a VNet from a subnet array and optionally deploys a Bastion-related public IP—only when you set the flag. No accidental public IPs “because the sample had one.”

```json
{
  "condition": "[parameters('deployBastionPublicIp')]",
  "type": "Microsoft.Network/publicIPAddresses",
  "apiVersion": "2023-09-01",
  "name": "[variables('pipName')]",
  ...
}
```

What-if this against a disposable RG before you get comfortable. Complete mode is still waiting in the wings with scissors.

Next: the nuances—API versions, secrets, RBAC races, drift, and all the ways a green deployment can still be lying to you.

---

**IaC Series:** [← Modular ARM Design]({{ '/articles/arm-modular-design/' | relative_url }}) · Part 3 of 4 · [Next: ARM Nuances →]({{ '/articles/arm-nuances/' | relative_url }})
