---
layout: article
title: "Windows Hybrid: Lessons from Devices Caught Between Worlds"
topic: Field Notes
category: field-notes
diagram: /images/lessons-windows-hybrid.svg
summary: Hybrid join, Intune, MECM/Tenant Attach, and why Windows device identity still decides whether cloud security policies mean anything.
author: Todd Williamsen
date: 2026-08-06
description: Lessons learned on hybrid Windows management with Entra Connect, hybrid join, Intune, Autopilot, and MECM Tenant Attach.
permalink: /articles/lessons-windows-hybrid/
---

Cloud security slides love “modern endpoints.” Reality in many enterprises is still hybrid: Active Directory on-prem, Entra Connect syncing identities, MECM co-managing with Intune, and Conditional Access expecting a device trust story that may or may not exist.

<figure>
  <img src="{{ '/images/lessons-windows-hybrid.svg' | relative_url }}" alt="Path from on-prem AD through Entra Connect hybrid join to Intune and Conditional Access">
  <figcaption>Figure 1. Fix device identity before rewriting cloud policy.</figcaption>
</figure>

## Lesson 1: Sync health is a security control

Broken Entra Connect cycles, UPN mismatches, stale federation metadata, and multi-forest weirdness show up as “random” auth failures—and as gaps in CA and PIM assumptions.

**What I do now:** treat connector health and export errors as Sev-2 security-adjacent issues, not just identity chores.

## Lesson 2: Hybrid join is binary until it isn’t

Devices that *look* joined in one console and not another create policy roulette. Autopilot and Tenant Attach help, but only if the join path is designed end to end.

**What I do now:** validate with `dsregcmd` / Graph device objects in troubleshooting before blaming MFA prompts.

## Lesson 3: GPO to Intune is a migration, not a checkbox

Moving baselines from Group Policy to Intune Security Baselines (NIST/CIS-aligned) fails when conflicting GPOs still win or co-management workloads are unclear.

**What I do now:** decide workload authority explicitly (Endpoint Protection, Device Configuration, etc.) and document it where humans can find it.

## Lesson 4: Scale without inventory is theater

Large estates (tens or hundreds of thousands of endpoints) need compliance dashboards and drift reporting. “We deployed Intune” without patch/config visibility is a press release.

## Closing

Basic concept: manage Windows from the cloud. Lesson learned: hybrid identity and co-management have to be healthy, or M365 and Azure security controls evaluate fiction.

**Related scripts:** `troubleshoot/Test-HybridJoin.ps1`, `troubleshoot/Get-IntuneComplianceSummary.ps1` in [cloud-powershell](https://github.com/toddwilliamsen/cloud-powershell).

