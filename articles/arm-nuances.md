---
layout: article
title: ARM Nuances in Production
topic: IaC Series · Part 4
summary: API versions, secrets handling, idempotency, RBAC races, complete-mode risk, and drift—the gaps between a green deployment and a safe platform.
author: Todd Williamsen
date: 2026-07-22
description: Production nuances for Azure ARM templates covering API version pinning, Key Vault secret references, role assignment races, drift, and complete deployment mode.
permalink: /articles/arm-nuances/
---

Most ARM failures in enterprise environments are not syntax errors. They are semantic ones: the template deployed, the portal looks fine, and three weeks later security or cost finds the gap. These are the nuances that separate “it works” from “it converges safely.”

<figure>
  <img src="{{ '/images/arm-nuances.svg' | relative_url }}" alt="ARM production nuances including API versions, secrets, idempotency, RBAC, complete mode, and drift">
  <figcaption>Figure 1. Treat templates as contracts—explicit versions, secrets handling, and ownership.</figcaption>
</figure>

## API versions change behavior

Pinning `apiVersion` freezes more than a date string. Properties appear, disappear, and change defaults.

Practical rules:

- Pin per resource type in code review, not by accident from a portal export
- When upgrading an API version, what-if in a non-prod scope first
- Do not mix “whatever the sample used last year” across a landing-zone library

Silent default changes are how public blob access and weaker TLS settings re-enter otherwise locked-down modules.

## Secrets never belong in source

Anti-patterns that still ship:

- Passwords in `parameters.json` committed to Git
- Connection strings in plain `string` parameters logged by pipelines
- Template outputs that echo secrets into deployment history

Prefer:

- `secureString` parameters supplied at deploy time
- Key Vault references where supported
- Managed identities instead of long-lived keys when the resource allows it

Deployment history is durable. Assume anything non-secure in parameters or outputs is readable later.

## Idempotency is a design requirement

A second deploy of the same template should be boring.

Watch for:

- **Guid-based names generated each run** — creates duplicates or conflicts
- **Child resources** with implicit names that clash on update
- **Role assignments** that must use a deterministic name (often a GUID derived from principal + role + scope)

If CI redeploy fails after a successful first deploy, the template is not done.

## RBAC races and principal timing

Managed identities and role assignments are a classic race:

1. Identity resource creates
2. Role assignment runs before the principal is usable everywhere
3. Intermittent failures that “pass on retry”

Mitigations:

- Explicit `dependsOn` to the identity
- Deterministic role assignment resource names
- Sometimes a short Deployment Script or retry policy in the pipeline when Azure eventual consistency bites

Also remember: assigning roles requires permissions at the scope. A template can be perfect and still fail because the deploying identity lacks `Microsoft.Authorization/roleAssignments/write`.

## Complete mode is a sharp tool

Complete mode remediates drift by deletion. That is valuable for locked platform resource groups and catastrophic for shared RGs where someone else’s jump box is “extra.”

Rules of thumb:

- Complete mode only on scopes fully owned by one pipeline
- Always what-if first
- Document the scope ownership in the repo README—not in tribal memory

## Drift will happen—plan for it

Portal clicks, hotfix hotfixes, and “temporary” firewall rules accumulate.

Combine:

- **Source control as intended state**
- **What-if in PRs** against representative subscriptions
- **Azure Policy** for guardrails templates should not be solely trusted to remember (deny public storage, require diagnostics)
- **Periodic redeploy** of platform modules so incremental updates reassert configuration

IaC without policy is documentation with a deploy button. Policy without IaC is a ticket queue.

## Naming and tagging are security controls

Unowned resources do not get patched, do not get billed correctly, and do not get isolated cleanly during an incident.

Require tags in the template (and reinforce with policy):

- `owner`
- `costCenter`
- `dataClassification`
- `environment`

If ownership is optional, incident response invents it under pressure.

## Expression language footguns

A few that waste real hours:

- `reference()` needs the right API version and can fail when the resource is not yet created
- `resourceId()` mistakes (wrong subscription/RG) fail late
- `uniqueString()` is stable for a seed—but changing the seed renames the world
- Nested templates change evaluation scope in ways that surprise people coming from imperative scripts

When an expression is hard to explain in a PR, simplify the parameters model instead of adding another layer of JSON cleverness.

## Bicep note (without abandoning ARM)

Bicep compiles to ARM. Authoring in Bicep often improves readability and modules, but:

- Runtime errors still surface as ARM deployments
- Existing ARM libraries remain relevant
- The nuances above still apply—they just hide behind nicer syntax

Learn ARM deeply enough to debug what Bicep emits.

## Closing

ARM mastery is less about memorizing every function and more about encoding safe defaults: pinned API versions, no secrets in Git, deterministic names, reviewed what-if, and policy backup for human error.

That is how infrastructure as code becomes a control plane—not a collection of templates that happened to deploy once.

[← Back to writing]({{ '/#writing' | relative_url }})

---

**IaC Series:** [← Advanced ARM Patterns]({{ '/articles/arm-advanced-patterns/' | relative_url }}) · Part 4 of 4
