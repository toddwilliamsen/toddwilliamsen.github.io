---
layout: article
title: "Defender for Cloud: Plans and Secure Score Triage"
topic: How-To
category: how-to
level: Advanced
diagram: /images/howto-defender-for-cloud-baseline.svg
summary: Enable the right Defender for Cloud plans for a lab subscription, read Secure Score recommendations, and triage a handful of high-impact fixes without boiling the ocean.
author: Todd Williamsen
date: 2025-10-06
description: Advanced how-to Defender for Cloud baseline—enable plans, Secure Score, recommendation triage, exemptions, and continuous export notes.
permalink: /articles/defender-for-cloud-baseline/
---

Defender for Cloud is posture + (optional) workload protection plans. The failure mode is enabling everything, ignoring Secure Score, and drowning in recommendations. This walkthrough enables a sensible plan set for a lab, reads Secure Score, and remediates a few high-value items deliberately.

<figure>
  <img src="{{ '/images/howto-defender-for-cloud-baseline.svg' | relative_url }}" alt="Defender plans feeding Secure Score recommendations and triage">
  <figcaption>Figure 1. Plans generate signals. Secure Score ranks work. Triage beats bulk-click.</figcaption>
</figure>

## What you need before you start

- Subscription Owner or Security Admin patterns that can enable Defender plans
- About **40 minutes**
- Cost awareness: Defender plans are paid—use lab subscriptions carefully

## Step 1 — Open Environment settings and pick plans

### Portal

1. Search **Microsoft Defender for Cloud**.
2. **Environment settings** → select your subscription.
3. Review plans. For a first infrastructure lab, common enables include:
   - **Foundational CSPM** / posture (naming evolves—enable the free/foundational posture features)
   - **Servers** if you have VMs and want agent/agentless insights (paid)
   - **Storage** if you want storage threat protection (paid)
4. Start with **CSPM/posture** recommendations first; add paid workload plans with intent.
5. Save.

Exact plan names change in the portal UI—read the price column before enabling.

## Step 2 — Read Secure Score without panic

1. Defender for Cloud → **Secure Score** (or Overview tile).
2. Note the percentage and top recommendations by **potential score impact**.
3. Filter to your subscription / resource groups for the lab.

Secure Score is a prioritization aid, not a compliance certificate.

## Step 3 — Triage method (use this every week)

For each top recommendation:

1. **Understand** the risk in one sentence.
2. **Scope**: which resources are affected?
3. **Decide**: Remediate now / accept with exemption + expiry / defer with owner.
4. **Fix** with portal quick-fix only when you trust it; otherwise use your IaC pipeline.
5. **Re-check** score and unhealthy resource count.

Example high-value infrastructure themes:

- Storage accounts allowing public blob access
- Missing Owner tags / diagnostics
- Management ports exposed to Internet
- Disks without encryption settings your org requires

## Step 4 — Remediate one recommendation end-to-end

Pick something you already know from earlier how-tos (storage public access):

1. Open the recommendation.
2. View unhealthy resources.
3. Fix in the storage networking/public access settings (or redeploy Bicep).
4. Wait for Defender to refresh (not always instant).
5. Confirm the resource moves to healthy.

## Step 5 — Exemptions with adult supervision

If a recommendation cannot apply:

1. Create an **exemption** with justification and expiration.
2. Ticket reference in the justification.
3. Review exemptions monthly—permanent silent exemptions are how baselines rot.

## Step 6 — Optional continuous export

Export recommendations/secure score to Log Analytics for trend reporting and Sentinel correlation. Configure under Environment settings / continuous export once the basic triage loop works.

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| Score never moves | Recommendations not actually remediated; or refresh lag |
| Bill surprise | Enabled Servers/Containers/SQL plans on many resources |
| “Quick fix” broke an app | Portal remediation without change control |
| Noise forever | No owner for recommendations; no exemption expiry |

## What you should remember

1. **Enable plans deliberately**—posture first, paid plans with purpose.
2. **Secure Score prioritizes**; it does not replace risk acceptance.
3. **Triage weekly**: fix, exempt with expiry, or assign.
4. **Prefer IaC fixes** for anything that must stick.
5. **Watch cost** alongside score.

Next: GitHub Actions OIDC to Azure so remediation and deploys do not rely on long-lived secrets.
