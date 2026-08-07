---
layout: article
title: "Identity First: Lessons from Azure and M365"
topic: Field Notes
category: field-notes
diagram: /images/lessons-identity.svg
summary: Basic identity concepts, learned the hard way—Entra ID as the control plane for Azure, Microsoft 365, and hybrid Windows.
author: Todd Williamsen
date: 2025-08-01
description: Lessons learned on Microsoft Entra ID as the shared control plane for Azure, M365, and hybrid Windows environments.
permalink: /articles/lessons-identity-entra-m365/
---

You can spend months “securing Azure” and still leave the front door open if identity is an afterthought. The basic concept everyone knows—centralize authentication—only sticks when Entra ID is treated as the control plane for **Azure, Microsoft 365, and hybrid Windows**, not a directory bolted on later.

<figure>
  <img src="{{ '/images/lessons-identity.svg' | relative_url }}" alt="Entra ID as control plane feeding Azure, M365, and hybrid Windows">
  <figcaption>Figure 1. One identity plane—or three separate fire drills.</figcaption>
</figure>

## Lesson 1: Same directory, different blast radii

Azure RBAC, Exchange Online, Teams, and hybrid-joined Windows devices often share the same Entra tenant. That is convenient. It is also how a weak admin path in one workload becomes a problem in all of them.

**What I do now:** map privileged roles across Azure *and* M365 before changing anything. Global Administrator is not “just an M365 thing.”

## Lesson 2: Workload identity is still identity

Managed identities and service principals need least privilege the same way humans do. Long-lived secrets in scripts are how labs become production incidents.

**What I do now:** prefer managed identity; if a secret must exist, it has an owner, an expiry, and a rotation path—not a comment that says “temporary.”

## Lesson 3: Hybrid join failures look like Conditional Access failures

When Windows devices fail hybrid join or Entra Connect sync drifts, users blame MFA and Conditional Access. Half the time the device never had a trustworthy identity to evaluate.

**What I do now:** verify device identity and sync health before rewriting CA policies. PowerShell helpers for that live in [cloud-powershell](https://github.com/toddwilliamsen/cloud-powershell).

## Lesson 4: Break-glass is a monitored exception, not folklore

Emergency accounts excluded from CA will exist. If nobody alerts on their use, you do not have break-glass—you have an unsupervised skeleton key.

## Closing

Basic concept: centralize identity. Lesson learned: centralize it *on purpose*, with privileged access, workload identities, and hybrid device health treated as one story.

**Related scripts:** `troubleshoot/Get-EntraSignInFailures.ps1`, `troubleshoot/Test-HybridJoin.ps1` in [cloud-powershell](https://github.com/toddwilliamsen/cloud-powershell).

