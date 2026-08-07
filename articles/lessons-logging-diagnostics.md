---
layout: article
title: "Logging Before the Fire: Lessons from Blind Investigations"
topic: Field Notes
category: field-notes
diagram: /images/lessons-logging.svg
summary: Diagnostics, Entra sign-in logs, and M365 unified audit—basic telemetry concepts learned after needing answers that were never collected.
author: Todd Williamsen
date: 2025-08-08
description: Lessons learned on Azure diagnostic settings, Entra sign-in logs, and Microsoft 365 unified audit logging for investigations.
permalink: /articles/lessons-logging-diagnostics/
---

Everyone agrees logging matters. The lesson arrives later: if diagnostic settings and audit pipelines were optional when the resource or workload was created, you will invent history during the incident—badly.

<figure>
  <img src="{{ '/images/lessons-logging.svg' | relative_url }}" alt="Azure and M365 telemetry flowing into collection then Log Analytics Sentinel and Purview">
  <figcaption>Figure 1. If it was never collected, it is not investigable.</figcaption>
</figure>

## Lesson 1: Azure diagnostics are a create-time decision

Activity Logs help. Resource logs and metrics to Log Analytics help more. “We’ll enable diagnostics after go-live” means go-live without an investigation path.

**What I do now:** DeployIfNotExists policies and templates that require a workspace ID—see also the ARM series and `secure/Enable-ResourceDiagnostics.ps1`.

## Lesson 2: Entra sign-in and audit logs are M365’s black box recorder

Failed CA, risky sign-ins, and legacy auth attempts live here. Retention and export to Sentinel (or at least a durable store) need a decision, not a default shrug.

## Lesson 3: Unified Audit Log is not optional for M365

Mailbox access, SharePoint sharing, Teams oddities—Unified Audit Logging (UAL) is how you answer “who did what?” Weeks of “audit wasn’t on” is an expensive way to learn a basic concept.

## Lesson 4: Collection without ownership is noise

Logs that land in a workspace nobody queries are comfort blankets. Tie workspace access, retention, and SOC content packs to an owner.

## Closing

Basic concept: enable logging. Lesson learned: make logging mandatory at provision time for Azure and M365, and prove someone can query it before you need it.

**Related scripts:** `secure/Enable-ResourceDiagnostics.ps1`, `troubleshoot/Get-M365UnifiedAudit.ps1`, `troubleshoot/Test-AzureDiagnostics.ps1` in [cloud-powershell](https://github.com/toddwilliamsen/cloud-powershell).

