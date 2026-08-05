---
layout: article
title: ARM Nuances in Production
topic: IaC Series · Part 4
summary: API versions, secrets, RBAC races, complete-mode risk, and drift—the gap between “deployment succeeded” and “we’re actually safe.”
author: Todd Williamsen
date: 2026-07-22
description: Production ARM nuances with a human tone—API version pinning, secrets, role assignment races, drift, and complete deployment mode.
permalink: /articles/arm-nuances/
---

Most ARM failures in the wild aren’t syntax errors. Syntax errors are merciful—they fail loudly and early. The painful ones are semantic: the template deployed, the portal looks fine, and three weeks later security finds a public endpoint that everyone assumed was impossible because “we have IaC.”

These are the nuances that separate “it worked” from “it converges safely.”

<figure>
  <img src="{{ '/images/arm-nuances.svg' | relative_url }}" alt="ARM production nuances including API versions, secrets, idempotency, RBAC, complete mode, and drift">
  <figcaption>Figure 1. Treat templates as contracts—Azure does not accept “we meant to” as a parameter.</figcaption>
</figure>

## API versions change behavior

Pinning `apiVersion` freezes more than a date string. Properties appear, disappear, and change defaults—sometimes with the energy of a silent software update that rearranges your kitchen overnight.

Practical rules:

- Pin per resource type in code review, not by accident from a portal export from 2021
- When upgrading an API version, what-if in non-prod first
- Don’t mix “whatever the sample used” across a whole library and call it consistency

Silent default changes are how public blob access and weaker TLS settings sneak back into otherwise locked-down modules. It’s not malice. It’s entropy with a changelog.

## Secrets never belong in source

Anti-patterns that somehow still ship:

- Passwords in `parameters.json` committed to Git (“temporary,” which is Latin for permanent)
- Connection strings as plain `string` parameters that pipelines cheerfully log
- Outputs that echo secrets into deployment history for future archaeologists

Prefer:

- `secureString` parameters supplied at deploy time
- Key Vault references where supported
- Managed identities instead of long-lived keys when the resource allows it

Deployment history is durable. Assume anything non-secure in parameters or outputs will be readable later—by someone helpful, or someone not.

## Idempotency is a design requirement

A second deploy of the same template should be boring. Boring is a compliment.

Watch for:

- **Guid-based names generated each run** — congratulations, you invented duplicates
- **Child resources** with implicit names that clash on update
- **Role assignments** that need a deterministic name (often a GUID derived from principal + role + scope)

If CI fails on redeploy after a perfect first run, the template isn’t finished—it’s a one-hit wonder.

## RBAC races and principal timing

Managed identities and role assignments are a classic race:

1. Identity resource creates
2. Role assignment runs before the principal is usable everywhere
3. Intermittent failure that “passes on retry,” which is Azure’s way of teaching patience

Mitigations:

- Explicit `dependsOn` to the identity
- Deterministic role assignment resource names
- Sometimes a short Deployment Script or pipeline retry when eventual consistency is feeling artistic

Also: a perfect template still fails if the deploying identity lacks `Microsoft.Authorization/roleAssignments/write`. ARM cannot grant you permissions you don’t have. It will, however, let you discover that fact in production if you skip the canary.

## Complete mode is a sharp tool

Complete mode remediates drift by deletion. Valuable for locked platform resource groups. Catastrophic for shared RGs where someone else’s jump box is now “unauthorized infrastructure.”

Rules of thumb:

- Complete mode only on scopes fully owned by one pipeline
- Always what-if first
- Document scope ownership in the repo README—not in Slack folklore

## Drift will happen—plan for it

Portal clicks, hotfix hotfixes, and “temporary” firewall rules accumulate like gym memberships.

Combine:

- **Source control as intended state**
- **What-if in PRs** against a representative subscription
- **Azure Policy** for guardrails templates shouldn’t be solely trusted to remember
- **Periodic redeploy** of platform modules so incremental updates reassert configuration

IaC without policy is documentation with a deploy button. Policy without IaC is a ticket queue with strong feelings.

## Naming and tagging are security controls

Unowned resources don’t get patched, don’t get billed correctly, and don’t get isolated cleanly during an incident. They just… exist. Menacingly.

Require tags in the template (and reinforce with policy):

- `owner`
- `costCenter`
- `dataClassification`
- `environment`

If ownership is optional, incident response invents it under pressure—and invents it wrong.

## Expression language footguns

A few that have stolen real hours of my life:

- `reference()` needs the right API version and sulks when the resource isn’t ready
- `resourceId()` mistakes (wrong subscription/RG) fail late, with confidence
- `uniqueString()` is stable for a seed—until someone “improves” the seed and renames the world
- Nested templates change evaluation scope in ways that surprise people who expected JavaScript

When an expression is hard to explain in a PR, simplify the parameters model. More cleverness is rarely the missing ingredient.

## Bicep note (without abandoning ARM)

Bicep compiles to ARM. Authoring in Bicep is often nicer. Agreed.

But:

- Runtime errors still surface as ARM deployments
- Existing ARM libraries aren’t going to apologize and disappear
- Every nuance above still applies—they just wear better syntax

Learn ARM deeply enough to debug what Bicep emits. Otherwise you’re flying a plane you can take off in but can’t read the instruments for.

## Sample template

[identity-rbac-securestring.json]({{ '/samples/arm/nuances/identity-rbac-securestring.json' | relative_url }}) shows three habits worth stealing:

1. User-assigned identity with an explicit `dependsOn` into the role assignment
2. Deterministic role assignment name via `guid(...)` so redeploys are boring
3. A `secureString` parameter you pass at deploy time—not from a committed parameters file

```json
"variables": {
  "roleAssignmentName": "[guid(resourceGroup().id, parameters('identityName'), parameters('roleDefinitionId'))]"
}
```

```bash
az deployment group create \
  -g rg-arm-samples \
  -f samples/arm/nuances/identity-rbac-securestring.json \
  -p bootstrapSecret='replace-me-at-deploy-time'
```

Full index of samples: [`/samples/arm/`]({{ '/samples/arm/' | relative_url }}).

## Closing

ARM mastery isn’t memorizing every function. It’s encoding safe defaults: pinned API versions, no secrets in Git, deterministic names, reviewed what-if, and policy as backup for the days humans are human.

That’s how infrastructure as code becomes a control plane—not a collection of templates that happened to deploy once and then entered local legend.

[← Back to writing]({{ '/#writing' | relative_url }})

---

**IaC Series:** [← Advanced ARM Patterns]({{ '/articles/arm-advanced-patterns/' | relative_url }}) · Part 4 of 4
