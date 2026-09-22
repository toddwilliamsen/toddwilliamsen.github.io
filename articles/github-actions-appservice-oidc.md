---
layout: article
title: Deploy App Service with GitHub Actions and OIDC
topic: How-To
category: how-to
level: Intermediate
diagram: /images/howto-github-actions-appservice-oidc.svg
summary: Create an Entra app with a GitHub federated credential, grant least-privilege Azure RBAC, and deploy to App Service without storing AZURE_CLIENT_SECRET.
author: Todd Williamsen
date: 2025-10-19
description: Intermediate guide to GitHub Actions OIDC federation with Microsoft Entra for Azure App Service deployment—federated credential, RBAC, and workflow.
permalink: /articles/github-actions-appservice-oidc/
---

Storing `AZURE_CLIENT_SECRET` in GitHub Secrets works until it leaks or expires at 2 a.m. **OIDC federation** lets GitHub mint a short-lived token Entra trusts. No long-lived Entra secret in the repo.

<figure>
  <img src="{{ '/images/howto-github-actions-appservice-oidc.svg' | relative_url }}" alt="GitHub Actions OIDC to Entra federated credential to App Service deploy">
  <figcaption>Figure 1. Trust the workflow identity; do not paste client secrets into GitHub.</figcaption>
</figure>

## What you need before you start

- GitHub repo with Actions enabled
- Rights to create an Entra app registration and Azure RBAC assignments
- Target Web App
- About **40 minutes**

## Step 1 — Create an Entra app for deployment

1. App registration: `sp-github-deploy-lab` (single tenant).
2. No redirect URI required.
3. Note **Application (client) ID** and **Directory (tenant) ID**.
4. Create a service principal if needed:

```bash
az ad sp create --id <client-id>
```

## Step 2 — Add a federated credential

Portal: app → **Certificates & secrets** → **Federated credentials** → **GitHub Actions deploying Azure resources**.

| Field | Value |
| --- | --- |
| Organization | your GitHub org/user |
| Repository | repo name |
| Entity | Environment, Branch, or PR |
| Example subject | `repo:contoso/web:environment:production` |

Prefer **Environments** with protection rules for production.

```bash
az ad app federated-credential create \
  --id <client-id> \
  --parameters '{
    "name": "github-prod",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:contoso/web:environment:production",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

## Step 3 — Grant least-privilege Azure RBAC

Scope to the Web App or resource group—not the subscription—when possible.

```bash
az role assignment create \
  --assignee <client-id> \
  --role "Website Contributor" \
  --scope /subscriptions/<sub>/resourceGroups/rg-appservice-lab/providers/Microsoft.Web/sites/<app>
```

Add **Reader** on the RG if the action must list resources. Avoid Owner.

## Step 4 — GitHub secrets (no client secret)

Repository or environment secrets:

| Name | Value |
| --- | --- |
| `AZURE_CLIENT_ID` | app client ID |
| `AZURE_TENANT_ID` | tenant ID |
| `AZURE_SUBSCRIPTION_ID` | subscription ID |

Do **not** create `AZURE_CLIENT_SECRET`.

## Step 5 — Workflow

```yaml
name: Deploy App Service
on:
  push:
    branches: [main]
permissions:
  id-token: write
  contents: read
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - name: Deploy
        run: |
          az webapp deploy \
            --resource-group rg-appservice-lab \
            --name <your-app> \
            --src-path ./site.zip \
            --type zip
```

`permissions.id-token: write` is mandatory for OIDC.

## Step 6 — Deploy to a slot (recommended)

Point the workflow at `--slot staging`, run smoke tests, then swap in a controlled job or manual approval environment.

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| Federated token error | Subject string mismatch (branch vs environment) |
| Login failed | Missing `id-token: write` |
| Authorization failed | RBAC scope too narrow/wrong role |
| Someone re-added client secret | Old template muscle memory—remove it |

## What you should remember

1. **Federated credential > client secret** for GitHub → Azure.
2. **Subject must match** entity type exactly.
3. **Least privilege RBAC** on the Web App/RG.
4. **`id-token: write`** in the workflow.
5. **Prefer slot deploy + swap** for production.

Next (advanced): SPA + API with PKCE.
