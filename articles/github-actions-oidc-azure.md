---
layout: article
title: GitHub Actions OIDC Deploy to Azure
topic: How-To
category: how-to
level: Advanced
diagram: /images/howto-github-actions-oidc-azure.svg
summary: Federate a GitHub Actions environment to Entra ID with OIDC, grant the app deployment rights, and ship a Bicep deploy workflow that needs no long-lived Azure client secret.
author: Todd Williamsen
date: 2025-10-07
description: Advanced how-to GitHub Actions OIDC to Azure—App registration federated credential, az login in workflow, Bicep deploy, least-privilege RBAC.
permalink: /articles/github-actions-oidc-azure/
---

Long-lived service principal secrets in GitHub are a rotation tax and a leak magnet. OIDC federation lets GitHub request a short-lived Entra token for a specific repo and branch/environment—no `AZURE_CLIENT_SECRET` required.

<figure>
  <img src="{{ '/images/howto-github-actions-oidc-azure.svg' | relative_url }}" alt="GitHub Actions OIDC token exchanged with Entra for Azure deployment">
  <figcaption>Figure 1. GitHub proves identity with OIDC. Entra issues a token. Azure deploy runs.</figcaption>
</figure>

## What you need before you start

- GitHub repo you control (Actions enabled)
- Rights to create Entra app registrations and assign Azure RBAC
- Azure CLI locally for setup
- About **45 minutes**
- A small Bicep template (from the earlier how-to) or any harmless RG deploy

## Step 1 — Create an Entra application and service principal

```bash
az ad app create --display-name "github-oidc-lab-deploy"
APP_ID=$(az ad app list --display-name "github-oidc-lab-deploy" --query [0].appId -o tsv)
az ad sp create --id "$APP_ID"
SP_OID=$(az ad sp show --id "$APP_ID" --query id -o tsv)
```

Note the **Application (client) ID** and your **Directory (tenant) ID** (`az account show --query tenantId -o tsv`).

## Step 2 — Add a federated credential for GitHub

Prefer environment-scoped subject for production; branch subject is fine for a lab.

### Portal

1. Entra → **App registrations** → your app → **Certificates & secrets** → **Federated credentials** → **Add**.
2. Scenario: **GitHub Actions deploying Azure resources**.
3. Org, repo, entity type: **Environment** (create `lab` in GitHub) or **Branch** `main`.
4. Name the credential; save.

### CLI (branch example)

```bash
# Adjust org/repo/branch
cat > cred.json <<'EOF'
{
  "name": "github-main",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:YOUR_ORG/YOUR_REPO:ref:refs/heads/main",
  "audiences": ["api://AzureADTokenExchange"],
  "description": "GitHub Actions main branch"
}
EOF

az ad app federated-credential create --id "$APP_ID" --parameters cred.json
```

Environment subject looks like `repo:ORG/REPO:environment:lab`.

## Step 3 — Grant Azure RBAC to the app

Least privilege: Contributor on one resource group, not Owner on the subscription.

```bash
az group create -n rg-gh-oidc -l eastus
RG_ID=$(az group show -n rg-gh-oidc --query id -o tsv)

az role assignment create \
  --assignee-object-id "$SP_OID" \
  --assignee-principal-type ServicePrincipal \
  --role Contributor \
  --scope "$RG_ID"
```

## Step 4 — GitHub repository secrets / variables

In the repo → **Settings** → **Secrets and variables** → **Actions**, add:

- `AZURE_CLIENT_ID` = app (client) ID
- `AZURE_TENANT_ID` = tenant ID
- `AZURE_SUBSCRIPTION_ID` = subscription ID

No client secret.

If using an Environment `lab`, create it under Settings → Environments and store secrets there; protect with required reviewers if you want.

## Step 5 — Workflow with OIDC login

Create `.github/workflows/deploy-bicep.yml`:

```yaml
name: Deploy Bicep
on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    # environment: lab   # uncomment if federated on environment
    steps:
      - uses: actions/checkout@v4

      - name: Azure login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy
        uses: azure/cli@v2
        with:
          inlineScript: |
            az deployment group create \
              --resource-group rg-gh-oidc \
              --template-file infra/main.bicep \
              --parameters storageName=stghoidc001 \
              --name gha-deploy-${{ github.run_id }}
```

Ensure `permissions.id-token: write` is present—OIDC fails silently-ish without it.

## Step 6 — Prove and harden

1. Push to `main` (or run workflow_dispatch).
2. Confirm the job logs show successful federated login and deploy.
3. Harden: narrow RBAC further; require environment approval; lock federated subject to one environment.

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| AADSTS70021 / federated token errors | Subject mismatch (branch vs environment); wrong org/repo |
| Login fails with permission error | Missing `id-token: write` |
| Deploy Forbidden | Role assignment on wrong scope; SP not created |
| Works locally SP secret, fails OIDC | Still using secret-based login in workflow |

## What you should remember

1. **Federated credential** ties GitHub identity to the Entra app.
2. **No client secret** in GitHub for this pattern.
3. **`id-token: write`** is mandatory in the workflow.
4. **RBAC scoped to one RG** beats subscription Owner for CI.
5. **Subject matching is exact**—branch and environment strings must match.

You now have a path from clean subscriptions through private networking, identity, policy, detection, and secretless deploy. Keep tearing down labs; keep encoding what survived into Bicep and MG policy.
