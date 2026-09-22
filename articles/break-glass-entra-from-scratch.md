---
layout: article
title: Entra Break-Glass Accounts from Scratch
topic: How-To
category: how-to
level: Beginner
diagram: /images/howto-break-glass-entra-from-scratch.svg
summary: Create and protect two cloud-only Entra emergency accounts so Conditional Access mistakes do not lock every Global Administrator out of the tenant.
author: Todd Williamsen
date: 2025-09-24
description: Beginner how-to for Entra ID break-glass accounts—cloud-only Global Administrators, long passwords, excluded from CA, monitored sign-ins, and safe storage.
permalink: /articles/break-glass-entra-from-scratch/
---

Conditional Access that requires MFA from a broken IdP, or a bad “block all” policy, can lock every day-to-day admin out of the tenant. Break-glass (emergency access) accounts are the fire axe: rarely used, heavily monitored, excluded from the policies that can strand you.

This is identity hygiene, not a daily login path. If these accounts become someone’s normal admin, you failed the exercise.

<figure>
  <img src="{{ '/images/howto-break-glass-entra-from-scratch.svg' | relative_url }}" alt="Two break-glass accounts excluded from Conditional Access with monitoring">
  <figcaption>Figure 1. Two cloud-only emergency admins, excluded from CA, watched by alerts.</figcaption>
</figure>

## What you need before you start

- Existing **Global Administrator** access (or ability to create users and assign roles)
- A secure place for secrets (offline password manager / sealed process—not Slack)
- About **30 minutes**
- Agreement that at least **two** people or sealed processes can retrieve the creds

Microsoft recommends at least two emergency accounts. Do not share one password across a team chat.

## Step 1 — Create two cloud-only users

Use a clear naming pattern and **onmicrosoft.com** UPNs so they do not depend on federated DNS.

Examples:

- `breakglass01@yourtenant.onmicrosoft.com`
- `breakglass02@yourtenant.onmicrosoft.com`

### Portal

1. Entra admin center → **Users** → **All users** → **Create new user**.
2. Create user: identity in the tenant domain (`*.onmicrosoft.com`).
3. Autogenerate a **very long** password; copy it into your sealed store immediately.
4. Do **not** enable “account is federated” or make this a synced AD account.
5. Repeat for the second account.

These should be **cloud-only**. Synced accounts die when on-prem sync or federation is the outage.

## Step 2 — Assign Global Administrator

### Portal

1. Entra → **Roles and administrators** → **Global Administrator**.
2. **Add assignments** → add `breakglass01` and `breakglass02`.
3. Prefer permanent assignment for true emergency accounts (PIM-eligible-only emergency accounts can fail when PIM or Entra is partially degraded—follow your org’s current Microsoft guidance and risk appetite).

Document why these two are permanent.

## Step 3 — Exclude them from risky Conditional Access

Every CA policy that could block interactive admin access must exclude these accounts (or an emergency group that contains only them).

### Portal

1. Entra → **Protection** → **Conditional Access** → each policy that targets admins or all users.
2. **Users** → **Exclude** → select both break-glass accounts (or the `Break Glass` security group).
3. Prefer a dedicated **security group** `sg-break-glass` with only these two users, and exclude the group—so you do not edit every policy by name later.
4. Save.

Also exclude them from policies that require a compliant device you might not have during an outage, or location conditions that assume corp egress.

## Step 4 — MFA stance for emergency accounts

Organizations differ. Common hardened patterns:

- Phishing-resistant methods registered in advance and stored with the sealed process, **or**
- Carefully controlled exceptions with compensating monitoring

Whatever you choose, **test** that you can still sign in when CA is broken. Do not discover the gap during an incident.

## Step 5 — Monitor like a smoke alarm

Treat any sign-in as an incident until proven otherwise.

### Minimum

1. Entra → **Sign-in logs** → filter by the two UPNs; pin a workbook or query.
2. Create an alert (Sentinel / Log Analytics / Entra ID Protection workflows) when either account signs in.
3. Quarterly: verify passwords/secrets still retrieveable; rotate on a schedule.

### Example Kusto sketch (Log Analytics with Entra sign-in import)

```kusto
SigninLogs
| where UserPrincipalName in ("breakglass01@yourtenant.onmicrosoft.com", "breakglass02@yourtenant.onmicrosoft.com")
| project TimeGenerated, UserPrincipalName, AppDisplayName, IPAddress, ResultType, Location
```

## Step 6 — Operational rules (write them down)

1. No daily use. No browser bookmark as default admin.
2. Two-person retrieval if your org requires it.
3. After any use: incident ticket, rotate secret, review CA changes that caused the need.
4. Keep a paper or offline procedure for “tenant locked” that a non-Azure specialist can follow.

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| Locked out anyway | Break-glass still included in a CA policy |
| Account useless in hybrid outage | Synced / federated UPN instead of cloud-only |
| Silent compromise | No sign-in alerting on the accounts |
| “Everyone knows the password” | Shared informal storage; not an emergency control |

## What you should remember

1. **Two cloud-only** Global Administrators on `*.onmicrosoft.com`.
2. **Exclude** them (via a group) from CA that can strand admins.
3. **Monitor every sign-in** as high severity.
4. **Never** use them for daily admin work.
5. **Test retrieval** before you need them.

Next: Conditional Access for Azure portals and CLI with these exclusions already in place, then PIM so standing Global Admin is rare for humans.
