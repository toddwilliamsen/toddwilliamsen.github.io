---
layout: article
title: Your First Bicep Deploy End-to-End
topic: How-To
category: how-to
level: Intermediate
diagram: /images/howto-bicep-first-deploy.svg
summary: Author a small Bicep file for a resource group and storage account, compile or deploy with Azure CLI, and understand parameters, what-if, and cleanup.
author: Todd Williamsen
date: 2025-10-01
description: Intermediate how-to for a first Bicep deploy end-to-end—install Bicep, write a template, az deployment, what-if, parameters, and teardown.
permalink: /articles/bicep-first-deploy/
---

Clickops does not scale and does not review well in PRs. Bicep is Azure’s concise DSL that compiles to ARM. This walkthrough deploys a resource group-scoped storage account from a `.bicep` file with Azure CLI—enough to prove the loop before modules and pipelines.

<figure>
  <img src="{{ '/images/howto-bicep-first-deploy.svg' | relative_url }}" alt="Bicep file to az deployment to Azure resources">
  <figcaption>Figure 1. Edit Bicep. What-if. Deploy. Same CLI path CI will use later.</figcaption>
</figure>

## What you need before you start

- Azure CLI with Bicep (`az bicep version` or `az bicep install`)
- Rights to create a resource group and storage account
- A terminal and a text editor
- About **30 minutes**

## Step 1 — Install / verify Bicep

```bash
az bicep install
az bicep version
az login
az account set --subscription "YOUR-SUB-NAME"
```

## Step 2 — Write a minimal template

Create `main.bicep`:

```bicep
@description('Storage account name (lowercase alphanumeric)')
param storageName string

@description('Azure region')
param location string = resourceGroup().location

resource stg 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageName
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    allowBlobPublicAccess: false
    minimumTlsVersion: 'TLS1_2'
    supportsHttpsTrafficOnly: true
  }
}

output storageId string = stg.id
```

Create `main.bicepparam` (optional but clean):

```bicep
using 'main.bicep'

param storageName = 'stlabbicep001'
```

Pick a globally unique `storageName`.

## Step 3 — Create the resource group

```bash
az group create --name rg-bicep-lab --location eastus
```

## Step 4 — Run what-if before deploy

```bash
az deployment group what-if \
  --resource-group rg-bicep-lab \
  --template-file main.bicep \
  --parameters storageName=stlabbicep001
```

Read the Create/Modify/NoChange summary. What-if is your pre-flight review.

## Step 5 — Deploy

```bash
az deployment group create \
  --resource-group rg-bicep-lab \
  --template-file main.bicep \
  --parameters storageName=stlabbicep001 \
  --name deploy-storage-1
```

Or with a parameter file:

```bash
az deployment group create \
  --resource-group rg-bicep-lab \
  --template-file main.bicep \
  --parameters main.bicepparam
```

Confirm the storage account exists and public blob access is disabled.

## Step 6 — Change and redeploy

Flip redundancy in the template (still LRS for lab cost) or add a tag:

```bicep
  tags: {
    Environment: 'lab'
    Owner: 'platform'
  }
```

Redeploy the same command. Bicep/ARM converges toward the desired state for resources it owns in that template.

## Step 7 — Tear down

```bash
az group delete --name rg-bicep-lab --yes --no-wait
```

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| Storage name invalid | Uppercase, dashes, or not globally unique |
| Authorization failed | Wrong subscription; missing Contributor |
| What-if scary Delete | Resource removed from template but still in RG—expected if you deleted from file |
| `az bicep` not found | CLI extension not installed |

## What you should remember

1. **Bicep is the source**; ARM JSON is the compile target.
2. **`what-if` before `create`** on shared environments.
3. **Parameters** keep names and SKUs out of hardcoding.
4. **Same CLI** is what GitHub Actions will run later with OIDC.
5. **Delete the lab RG** when done.

Next: hub-and-spoke with Bastion, then OIDC from GitHub so humans are not long-lived deploy secrets.
