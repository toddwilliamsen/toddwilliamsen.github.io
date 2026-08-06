---
layout: article
title: "Least Privilege: Lessons from Standing Admin"
topic: Lessons · Cloud Basics
summary: RBAC and PIM fundamentals learned after standing Owner rights and shared admin accounts turned into predictable risk.
author: Todd Williamsen
date: 2026-08-11
description: Lessons learned on Azure RBAC, Entra PIM, managed identities, and removing standing administrative access.
permalink: /articles/lessons-rbac-least-privilege/
---

Least privilege is the most basic cloud security concept that still loses to “just give them Owner so we can ship.” The lesson is operational: standing admin is a future incident with better calendar invites.

<figure>
  <img src="{{ '/images/lessons-rbac.svg' | relative_url }}" alt="Avoid standing Owner versus prefer PIM eligible roles and verify who remains standing">
  <figcaption>Figure 1. Prefer eligible privilege—then verify who is still standing.</figcaption>
</figure>

## Lesson 1: Owner is not a personality type

Subscription Owner and User Access Administrator show up because they unblock everything. They also unblock lateral movement and quiet persistence.

**What I do now:** default to narrower roles; elevate with PIM; review standing assignments on a schedule that actually happens.

## Lesson 2: PIM without culture is theater

Eligible roles nobody activates correctly, or approvals that always auto-approve, recreate standing access with extra clicks.

**What I do now:** require justification, short windows, and spot-check activation logs.

## Lesson 3: Scripts need identities too

Automation run as a human’s cloud shell or a shared SP with Contributor everywhere will eventually leak.

**What I do now:** managed identity per workload where possible; Graph/Azure permissions scoped to the task; secrets out of repos (see ARM nuances).

## Lesson 4: M365 admin roles drift too

Exchange Admin, SharePoint Admin, and Helpdesk roles accumulate. Azure RBAC hygiene with ignored M365 roles is half a control plane.

**What I do now:** include Entra directory roles in the same access-review conversation as Azure RBAC.

## Closing

Basic concept: least privilege. Lesson learned: measure standing privilege, make elevation temporary, and include both Azure and M365 admin planes.

**Related scripts:** `secure/Get-StandingAzureRoleAssignments.ps1`, `secure/Get-PimEligibleRoles.ps1` in [cloud-powershell](https://github.com/toddwilliamsen/cloud-powershell).

---

**Lessons:** [← Logging]({{ '/articles/lessons-logging-diagnostics/' | relative_url }}) · Part 5 · [Projects: Rocky lab →]({{ '/projects/rocky-lab/' | relative_url }})
