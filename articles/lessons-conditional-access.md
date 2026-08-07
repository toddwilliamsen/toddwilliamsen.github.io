---
layout: article
title: "Conditional Access: Lessons from Policies That Drifted"
topic: Field Notes
category: field-notes
diagram: /images/lessons-conditional-access.svg
summary: Conditional Access is straightforward on a whiteboard and messy in a tenant—what actually holds for Azure admin portals and Microsoft 365 apps.
author: Todd Williamsen
date: 2026-08-04
description: Lessons learned implementing Conditional Access for Microsoft 365 and Azure admin access, including exclusions, device compliance, and break-glass.
permalink: /articles/lessons-conditional-access/
---

Conditional Access sounds basic: if user + app + context, then require MFA or a compliant device. The lesson is not the concept—it is keeping policies from rotting into a maze of permanent exclusions.

<figure>
  <img src="{{ '/images/lessons-conditional-access.svg' | relative_url }}" alt="Conditional Access signals flowing into policy decisions and outcomes">
  <figcaption>Figure 1. Signals in, enforcement out—exclusions need expiry dates.</figcaption>
</figure>

## Lesson 1: Protect admin paths first

Broad user MFA without hardening Azure portal / Graph / PowerShell admin paths leaves the highest-value targets soft.

**What I do now:** phishing-resistant methods and compliant (or strong) device requirements for privileged roles before polishing edge cases for every SaaS app.

## Lesson 2: “Report-only” is not a personality trait

Report-only mode is for learning impact. Leaving policies there forever is how nothing ever enforces.

**What I do now:** time-box report-only, review sign-in diagnostic logs, then enforce—or delete the policy.

## Lesson 3: Exclusions age like milk

Guest access, legacy apps, break-glass, that one vendor portal—exclusions accumulate. Without owners and review dates, CA becomes Swiss cheese with documentation.

**What I do now:** every exclusion has an owner ticket and a revisit date. No ticket, no exclusion.

## Lesson 4: Device compliance only works if devices are real

CA requiring compliant devices fails socially when Intune enrollment or hybrid join is broken. Users work around it; security looks like the villain.

**What I do now:** pair CA rollout with device health metrics, not just policy screenshots.

## Closing

Basic concept: Conditional Access enforces Zero Trust signals. Lesson learned: treat policies like code—review, expire exceptions, and verify the signals are trustworthy.

**Related scripts:** `secure/Get-ConditionalAccessExclusions.ps1`, `troubleshoot/Get-EntraSignInFailures.ps1` in [cloud-powershell](https://github.com/toddwilliamsen/cloud-powershell).

