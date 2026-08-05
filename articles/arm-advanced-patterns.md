---
layout: article
title: Advanced ARM Patterns
topic: IaC Series · Part 3
summary: Copy loops, conditions, nested deployments, what-if validation, and the difference between incremental delivery and complete-mode remediation.
author: Todd Williamsen
date: 2026-07-15
description: Advanced Azure ARM techniques including copy, conditions, deployment scripts, what-if, and incremental versus complete deployment modes.
permalink: /articles/arm-advanced-patterns/
---

Once modules exist, the next problems are repetition, optionality, and confidence before you touch production. Advanced ARM is less about clever JSON and more about controlling blast radius while the platform converges.

<figure>
  <img src="{{ '/images/arm-advanced.svg' | relative_url }}" alt="Advanced ARM patterns for copy loops, conditions, deployment modes, and a validation layer with what-if">
  <figcaption>Figure 1. Advanced patterns should increase control—not cleverness for its own sake.</figcaption>
</figure>

## Copy loops: repetition with intent

Use `copy` when you need N of the same shape:

- Subnets from an array of CIDR definitions
- Nic / disk attachments
- Role assignments across a list of principal IDs
- Diagnostic settings across a resource list

Watch for:

- **Serial vs parallel** copy mode when Azure has creation limits or ordering needs
- **Name uniqueness** — collisions fail loudly in CI and quietly in partial retries
- **Index expressions** that become unreadable; prefer named objects in arrays

If the loop encodes business logic nobody can explain, move that logic up into parameters design.

## Conditions: deploy only when required

`condition` keeps prod-only controls out of non-prod without maintaining two templates:

- Deploy Private Endpoints only when `usePrivateNetwork` is true
- Attach diagnostic settings only when a workspace ID is provided
- Skip Bastion in sandboxes

Conditionals should be boring. Nested conditions that recreate a programming language inside JSON are a maintenance tax.

## Deployment scopes beyond the resource group

ARM can deploy at:

- **Resource group** — most app workloads
- **Subscription** — role assignments, diagnostic settings at scale, policy assignments (depending on design)
- **Management group** — guardrails that must inherit

Subscription and MG scoped templates are how platform teams encode “every subscription gets logging and these deny policies” without hoping each app team remembers.

## Incremental vs complete mode

| Mode | Behavior | Typical use |
| --- | --- | --- |
| Incremental | Create/update resources in the template; leave others alone | Day-to-day app delivery |
| Complete | Make the target scope match the template; remove extras | Tightly controlled platform RGs |

**Complete mode’s blast radius is the scope.** A missing resource in the template is a delete. Use it only when the resource group (or scope) is fully owned by that template and the pipeline is trusted.

## What-if before promote

`az deployment group what-if` (and equivalents) answers: *what would change?*

Make what-if a gate:

1. Validate template
2. What-if against the target scope
3. Human review on destructive or public-exposure changes
4. Deploy
5. Redeploy to confirm idempotency

If what-if is optional, production surprises are scheduled.

## Deployment Scripts when ARM is not enough

Some bootstrap steps are awkward in pure declarative form: seeding a Key Vault value from a pipeline-managed secret rotation, one-time DNS glue, invoking an API Azure does not model cleanly.

**Deployment Scripts** run in a managed context during deployment. Use sparingly—they are powerful and easy to turn into hidden imperative snowflakes. Prefer native resources when they exist.

## A promotion path that scales

- **Dev subscription** — fast iteration, complete mode allowed on disposable RGs
- **Canary** — what-if required, incremental only
- **Prod** — what-if + change ticket for network / identity / public access diffs

Advanced ARM is not maximal language features. It is repeatable convergence with fewer 2 a.m. rollbacks.

Next: the nuances—API versions, secrets, RBAC races, drift, and the failure modes templates do not catch alone.

---

**IaC Series:** [← Modular ARM Design]({{ '/articles/arm-modular-design/' | relative_url }}) · Part 3 of 4 · [Next: ARM Nuances →]({{ '/articles/arm-nuances/' | relative_url }})
