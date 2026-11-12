---
layout: article
title: "cloud-powershell: Scripts That Earn Their Keep"
topic: Field Notes
category: field-notes
diagram: /images/cloud-powershell-scripts.svg
summary: A catalog of the PowerShell helpers I keep next to the lessons—troubleshoot, build, and secure Azure, Entra ID, Microsoft 365, and hybrid Windows.
author: Todd Williamsen
date: 2026-11-12
description: Overview of the cloud-powershell GitHub repo—troubleshoot, build, and secure scripts for Azure, Entra ID, M365, and Windows hybrid, with links to each script.
permalink: /articles/cloud-powershell-scripts/
---

Portal clicks do not scale when you are chasing CA exclusions, standing Owner assignments, or a hybrid join that looks fine until PRT is missing. The [cloud-powershell](https://github.com/toddwilliamsen/cloud-powershell) repo is the companion kit to the field-notes series: short, educational scripts you run with eyes open—not unattended production runbooks.

<figure>
  <img src="{{ '/images/cloud-powershell-scripts.svg' | relative_url }}" alt="Three folders: troubleshoot, build, and secure PowerShell helpers">
  <figcaption>Figure 1. Same three lanes as the repo—diagnose, provision, then verify posture.</figcaption>
</figure>

## How to use the repo

- Clone or browse [github.com/toddwilliamsen/cloud-powershell](https://github.com/toddwilliamsen/cloud-powershell)
- Prefer **PowerShell 7+**; each script declares its modules (`Az.*`, `Microsoft.Graph.*`, `ExchangeOnlineManagement`)
- Connect with least privilege for the task—do not default to Global Admin
- Teaching / operator aids only; never commit secrets

The field notes that motivated most of these scripts: [Conditional Access]({{ '/articles/lessons-conditional-access/' | relative_url }}), [RBAC / PIM]({{ '/articles/lessons-rbac-least-privilege/' | relative_url }}), [logging]({{ '/articles/lessons-logging-diagnostics/' | relative_url }}), [identity]({{ '/articles/lessons-identity-entra-m365/' | relative_url }}), and [Windows hybrid]({{ '/articles/lessons-windows-hybrid/' | relative_url }}).

## troubleshoot/

When something is already broken—or you need evidence before rewriting policy.

| Script | What it does |
|--------|----------------|
| [Test-HybridJoin.ps1](https://github.com/toddwilliamsen/cloud-powershell/blob/main/troubleshoot/Test-HybridJoin.ps1) | Local `dsregcmd` summary for Azure AD join, domain join, and PRT—run on the Windows endpoint |
| [Get-EntraSignInFailures.ps1](https://github.com/toddwilliamsen/cloud-powershell/blob/main/troubleshoot/Get-EntraSignInFailures.ps1) | Recent failed Entra sign-ins with error code, failure reason, and CA status |
| [Test-AzureDiagnostics.ps1](https://github.com/toddwilliamsen/cloud-powershell/blob/main/troubleshoot/Test-AzureDiagnostics.ps1) | Lists resources in a resource group that appear to lack diagnostic settings |
| [Get-M365UnifiedAudit.ps1](https://github.com/toddwilliamsen/cloud-powershell/blob/main/troubleshoot/Get-M365UnifiedAudit.ps1) | Searches the Microsoft 365 Unified Audit Log for a time window (optional operations filter) |
| [Get-IntuneComplianceSummary.ps1](https://github.com/toddwilliamsen/cloud-powershell/blob/main/troubleshoot/Get-IntuneComplianceSummary.ps1) | Groups managed devices by Intune compliance state |

Folder: [troubleshoot/](https://github.com/toddwilliamsen/cloud-powershell/tree/main/troubleshoot)

## build/

Lab and learning scaffolding—stand something up cleanly before you harden it.

| Script | What it does |
|--------|----------------|
| [New-LabResourceGroup.ps1](https://github.com/toddwilliamsen/cloud-powershell/blob/main/build/New-LabResourceGroup.ps1) | Creates a tagged resource group (`owner`, `environment`, `purpose=lab`) |
| [Deploy-ArmTemplate.ps1](https://github.com/toddwilliamsen/cloud-powershell/blob/main/build/Deploy-ArmTemplate.ps1) | Validates an ARM template, runs what-if, then incremental deploy |
| [New-EntraAppRegistration.ps1](https://github.com/toddwilliamsen/cloud-powershell/blob/main/build/New-EntraAppRegistration.ps1) | Creates a basic single-tenant app registration and service principal—credentials stay out of the script output |

Folder: [build/](https://github.com/toddwilliamsen/cloud-powershell/tree/main/build)

## secure/

Posture checks and small hardening steps that match how the lessons say to operate.

| Script | What it does |
|--------|----------------|
| [Enable-ResourceDiagnostics.ps1](https://github.com/toddwilliamsen/cloud-powershell/blob/main/secure/Enable-ResourceDiagnostics.ps1) | Enables diagnostic settings on a resource to a Log Analytics workspace |
| [Set-StorageAccountHardening.ps1](https://github.com/toddwilliamsen/cloud-powershell/blob/main/secure/Set-StorageAccountHardening.ps1) | Applies TLS 1.2, HTTPS-only, and disables anonymous blob public access |
| [Get-StandingAzureRoleAssignments.ps1](https://github.com/toddwilliamsen/cloud-powershell/blob/main/secure/Get-StandingAzureRoleAssignments.ps1) | Lists user/group/SP role assignments at subscription (or custom) scope—hunt standing privilege |
| [Get-PimEligibleRoles.ps1](https://github.com/toddwilliamsen/cloud-powershell/blob/main/secure/Get-PimEligibleRoles.ps1) | Lists Entra PIM eligible directory role assignment instances |
| [Get-ConditionalAccessExclusions.ps1](https://github.com/toddwilliamsen/cloud-powershell/blob/main/secure/Get-ConditionalAccessExclusions.ps1) | Surfaces Conditional Access policies with excluded users and groups |

Folder: [secure/](https://github.com/toddwilliamsen/cloud-powershell/tree/main/secure)

## Closing

These scripts exist so the lessons have teeth: when the note says “inventory CA exclusions” or “verify who is still standing,” there is a concrete path on GitHub instead of another portal scavenger hunt. Start at the [repo root](https://github.com/toddwilliamsen/cloud-powershell), pick a folder that matches the problem, and read the synopsis before you connect.
