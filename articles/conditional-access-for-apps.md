---
layout: article
title: Conditional Access Policies for Your App
topic: How-To
category: how-to
level: Advanced
diagram: /images/howto-conditional-access-for-apps.svg
summary: Target Conditional Access at your app’s enterprise application—start in report-only, require MFA or compliant devices, and avoid locking out break-glass accounts.
author: Todd Williamsen
date: 2025-10-24
description: Advanced how-to for Microsoft Entra Conditional Access on custom apps—cloud app targeting, grant controls, report-only mode, and exclusions.
permalink: /articles/conditional-access-for-apps/
---

App registration gets users a token path. **Conditional Access (CA)** decides whether that path is allowed based on user, device, location, and risk. For your custom app, assign policies to the **enterprise application** (service principal), not “all cloud apps,” unless you intend global blast radius.

<figure>
  <img src="{{ '/images/howto-conditional-access-for-apps.svg' | relative_url }}" alt="User sign-in gated by Conditional Access before tokens are issued to the app">
  <figcaption>Figure 1. CA runs at Entra before your app sees a successful login.</figcaption>
</figure>

## What you need before you start

- Entra ID P1/P2 (CA licensing)
- Rights to create CA policies
- Your app’s enterprise application object
- Break-glass accounts excluded from CA
- About **30–40 minutes** plus observation time in report-only

## Step 1 — Find the cloud app target

1. Entra ID → **Enterprise applications** → your app.
2. Confirm users can be assigned; note the display name.
3. Prefer policies that list **this app** under Cloud apps / target resources.

## Step 2 — Create a policy in report-only

1. **Protection** → **Conditional Access** → **Create new policy**.
2. Users: a pilot group (not all users on day one).
3. Target resources: select your enterprise app.
4. Grant: **Require multifactor authentication** (baseline).
5. Enable policy: **Report-only**.
6. **Create**.

Watch **Sign-in logs** → Conditional Access tab for Would block / success.

## Step 3 — Strengthen grants thoughtfully

| Grant | When |
| --- | --- |
| MFA | Default for corporate apps |
| Compliant device / HAADJ | Higher assurance admin or finance apps |
| Require auth strength | Phishing-resistant MFA for privileged |
| Block | Rare; prefer grant controls |

Session controls (sign-in frequency, persistent browser) can reduce token lifetime for sensitive apps—test with MSAL silent acquire behavior.

## Step 4 — Exclusions that must exist

| Exclude | Why |
| --- | --- |
| Break-glass accounts | Avoid total lockout |
| Specific automation (carefully) | Some ROPC/legacy flows break—prefer modernize |
| Emergency access group | Documented dual accounts |

Never exclude broad “all guest users” without a compensating policy.

## Step 5 — Move to On

1. Pilot group → policy **On**.
2. Expand users gradually.
3. Keep a second policy or dual control for privileged roles.
4. For multi-tenant SaaS, remember **customer tenants apply their own CA** to your SP in their directory; document requirements for customers.

## App design interactions

| App pattern | CA note |
| --- | --- |
| Easy Auth | Users hit CA during platform login |
| MSAL SPA | Interactive acquire may need extra prompts on step-up |
| OBO | Downstream may fail if interactive control required |
| Daemon / MI | Not user CA; protect with RBAC and network |

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| All apps affected | Targeted “All cloud apps” by accident |
| Locked out admins | No break-glass exclusion |
| OBO failures | Policy requires interactive MFA mid-chain |
| Report-only ignored | Never reviewed logs before enabling On |

## What you should remember

1. **Target your enterprise app**, not everything, for app-specific policy.
2. **Report-only first**, then On for a pilot.
3. **Break-glass exclusions** are mandatory.
4. **MFA baseline**; device compliance for higher trust.
5. **OBO and daemons** need explicit design under CA.

Next: put App Service hardening into one secure baseline.
