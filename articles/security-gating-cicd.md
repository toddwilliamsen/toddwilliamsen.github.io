---
layout: article
title: Security Gating CI/CD Pipelines
topic: How-To
category: how-to
level: Intermediate
diagram: /images/howto-security-gating-cicd.svg
summary: I was asked about security gating CI/CD pipelines—here is the fail-closed pattern I would put in front of Azure deploys across pre-merge, build, release, and runtime.
author: Todd Williamsen
date: 2026-11-11
description: Practical security gating for Azure CI/CD—branch policies, secret scanning, SAST, SCA, MSDO/Trivy/IaC scans, artifact signing, Azure Policy and Defender environment checks, AKS admission, plus GitHub Actions and Azure DevOps workflow samples.
permalink: /articles/security-gating-cicd/
---

I was asked today about security gating CI/CD pipelines, and I had to think what would I do?

Not a tool checklist for its own sake. Checkpoints that **fail closed**—vulnerable code, misconfigured infrastructure, and unapproved artifacts never reach production just because a deploy job was green on the last green day.

Depending on whether you use **Azure DevOps** or **GitHub Actions** against Azure, I would enforce gates at four stages.

<figure>
  <img src="{{ '/images/howto-security-gating-cicd.svg' | relative_url }}?v=3" alt="Four CI/CD stages with security gates from pre-merge through runtime admission">
  <figcaption>Figure 1. Pre-merge, build, release, and runtime—each stage can stop the train.</figcaption>
</figure>

## What you need before you start

- A repo on **GitHub** or **Azure DevOps** with branch policies / protection rules you can change
- Rights to create **Environments** (Approvals and Checks) and wire an Azure service connection or OIDC app
- For Azure DevOps: the free **Microsoft Security DevOps** extension from the Marketplace
- About **60–90 minutes** to wire a lab pipeline; longer to harden production thresholds
- Sample workflows in [`samples/workflows/`]({{ '/samples/workflows/' | relative_url }})

## 1 — Pre-merge and source gates (shift left)

Before code enters `main`, block insecure changes with branch policy and automated scans.

- **Branch policies:** Require pull requests, a minimum reviewer count, and required status checks before merge.
- **Secret scanning:** Enable GitHub Advanced Security / Secret Protection (or Advanced Security for Azure DevOps) with **push protection** so commits containing API keys, passwords, or Azure connection strings never land.
- **SAST:** Run CodeQL, SonarQube, or Semgrep on the PR build to catch injection patterns, XSS, and insecure defaults before review rubber-stamps the change.

If the PR is the only human review, the PR build is also the only honest security review. Treat required checks as policy, not decoration.

## 2 — Build and artifact gates (CI)

During the build, verify dependencies and images meet a baseline **before** you publish anything downstream.

- **SCA:** Dependency-Check, Snyk, or GitHub dependency review / Advanced Security dependency scanning—**fail** on Critical/High CVEs introduced by the change.
- **Container and IaC:** Use **Microsoft Security DevOps (MSDO)** in Azure Pipelines (or equivalent Actions) to run unified scanners such as **Trivy** (containers) and **Checkov / Terrascan** (Terraform, Bicep, ARM).
- **Artifact signing:** Sign verified images with Notation or Cosign and push to **Azure Container Registry** so later environments only pull trusted digests.

<figure>
  <img src="{{ '/images/howto-security-gating-fail-closed.svg' | relative_url }}?v=3" alt="Scan results evaluated at a severity gate that either blocks deploy or allows OIDC deploy">
  <figcaption>Figure 2. Warn-only is research. Exit 1 on Critical/High is a gate.</figcaption>
</figure>

## 3 — Release and deployment gates (CD)

In **Azure DevOps Environments** or **GitHub Environments**, pause deploy until security conditions pass.

- **Azure Policy compliance:** In Azure DevOps, use **Check Azure Policy compliance** against the target subscription or resource group. Unencrypted storage, public exposure, or other deny/audit failures stop the release.
- **Azure Monitor / Defender alerts:** A **Query Azure Monitor alerts** check can halt deploy when Microsoft Defender for Cloud has active high-severity findings in the target scope.
- **Custom REST / Azure Function:** Call Wiz, Prisma Cloud, ServiceNow change tickets, or your own Function—only a pass (for example HTTP 200 with an explicit allow body) continues.
- **Manual security approvals:** Require sign-off from a Security Champions group for production. Automation scales; humans still own blast radius.

GitHub Environment protection rules (required reviewers, wait timer, deployment branches) are the parallel control when you are not on Azure DevOps.

## 4 — Runtime and cluster admission (post-CD)

Pipelines are not the only path into Azure. Keep a final gate on the platform.

- **AKS admission (OPA Gatekeeper / Azure Policy for AKS):** Block privileged pods, unapproved registries, and unsigned images even if someone kubectl-applies out of band.
- **Microsoft Defender for DevOps:** Connect Azure DevOps and GitHub orgs to Defender for Cloud for code-to-cloud posture, centralized visibility, and PR annotations on critical findings.

## Quick setup — Azure DevOps

1. Install **Microsoft Security DevOps** from the Visual Studio Marketplace.
2. Add `MicrosoftSecurityDevOps@1` in the build stage to scan code, IaC, and containers.
3. Follow with a script or SARIF parser that **exits 1** on Critical/High findings—do not publish artifacts on a soft fail.
4. Open **Pipelines → Environments → [production] → Approvals and checks** and add Azure Policy and Azure Monitor gates before the deployment job.

Full sample: [`samples/workflows/azure-devops-security-gates.yml`]({{ '/samples/workflows/azure-devops-security-gates.yml' | relative_url }}).

```yaml
- task: MicrosoftSecurityDevOps@1
  displayName: Microsoft Security DevOps
  inputs:
    categories: "code,artifacts,IaC,containers"

# Then parse SARIF and exit 1 on Critical/High (see sample workflow)
```

## Quick setup — GitHub Actions

1. Protect `main`: required PR, required checks, dismiss stale reviews.
2. Turn on secret scanning + push protection (and dependency review on PRs).
3. Run CodeQL (or your SAST) and a filesystem/IaC scanner such as Trivy with `exit-code: 1` on Critical/High.
4. Put the deploy job behind `needs: […]`, `environment: production`, and **OIDC** login—no `AZURE_CLIENT_SECRET`. See [GitHub Actions OIDC Deploy to Azure]({{ '/articles/github-actions-oidc-azure/' | relative_url }}).

Full sample: [`samples/workflows/github-security-gates.yml`]({{ '/samples/workflows/github-security-gates.yml' | relative_url }}).

```yaml
jobs:
  sca-and-iac:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aquasecurity/trivy-action@0.28.0
        with:
          scan-type: fs
          severity: CRITICAL,HIGH
          exit-code: "1"

  deploy-gated:
    needs: [secret-and-sast, sca-and-iac]
    environment: production
    permissions:
      id-token: write
      contents: read
```

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| Scans “run” but nothing blocks merge | Checks not required on the branch / environment |
| SARIF uploaded, pipeline still green | No fail-closed step on Critical/High |
| Policy gate always passes | Scope wrong (RG vs subscription) or policies still in Audit |
| Deploy works from laptop, fails in CI | Missing OIDC `id-token: write` or federated subject mismatch |
| AKS still accepts bad images | Admission policy not assigned / not in Deny |

## What you should remember

1. **Four stages:** pre-merge, build, release, runtime—skip one and attackers (or tired humans) will use that path.
2. **Fail closed** on Critical/High; warn-only is how you inventory risk, not how you gate production.
3. **Environment checks** (Policy, Defender alerts, manual security approval) belong in front of deploy, not after the postmortem.
4. **Short-lived deploy identity** (OIDC / federation) keeps the gated pipeline from becoming a standing secret store.
5. **Admission control** catches what never went through CI.

Next: wire OIDC for the deploy job that only runs after these gates—[GitHub Actions OIDC to Azure]({{ '/articles/github-actions-oidc-azure/' | relative_url }}) and [Deploy App Service with GitHub Actions and OIDC]({{ '/articles/github-actions-appservice-oidc/' | relative_url }}).
