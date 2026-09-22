---
layout: article
title: Deploy a Simple Azure App Service Web App
topic: How-To
category: how-to
level: Beginner
diagram: /images/howto-appservice-webapp-from-scratch.svg
summary: Create a resource group, App Service plan, and Linux Web App, then deploy a hello-world site over HTTPS—portal and CLI included.
author: Todd Williamsen
date: 2025-10-09
description: Beginner guide to deploy an Azure App Service web app from scratch—plan SKU, runtime stack, ZIP deploy, HTTPS URL, and cleanup.
permalink: /articles/appservice-webapp-from-scratch/
---

Azure **App Service** hosts HTTP apps without you managing VMs. You pick a **plan** (compute) and a **Web App** (site). This lab stands up a Linux site you can hit in a browser, then tear down.

<figure>
  <img src="{{ '/images/howto-appservice-webapp-from-scratch.svg' | relative_url }}" alt="Code deploys into App Service plan and Web App, users browse the azurewebsites.net URL">
  <figcaption>Figure 1. Plan holds compute; the Web App is the site and URL.</figcaption>
</figure>

## What you need before you start

- An Azure subscription with rights to create App Service resources
- Azure CLI installed (optional but useful)
- About **20–30 minutes**
- A tiny sample (Node or static HTML is enough)

## Step 1 — Resource group

### Portal

1. **Resource groups** → **Create**.
2. Name: `rg-appservice-lab`.
3. Region: e.g. **East US**.
4. **Create**.

### CLI

```bash
az group create --name rg-appservice-lab --location eastus
```

## Step 2 — App Service plan

The plan is the VM-equivalent SKU. For a first lab, **Free (F1)** or **Basic (B1)** is fine. Free has limitations (no slots, cold starts). B1 is more realistic.

### Portal

1. Search **App Service plans** → **Create**.
2. Resource group: `rg-appservice-lab`.
3. Name: `asp-lab`.
4. Operating System: **Linux**.
5. Region: same as the group.
6. Pricing plan: **Free F1** or **Basic B1**.
7. **Create**.

### CLI

```bash
az appservice plan create \
  --name asp-lab \
  --resource-group rg-appservice-lab \
  --is-linux \
  --sku B1
```

## Step 3 — Create the Web App

### Portal

1. Search **Web App** → **Create**.
2. Resource group: `rg-appservice-lab`.
3. Name: globally unique, e.g. `app-contoso-lab-<unique>`.
4. Publish: **Code**.
5. Runtime stack: **Node 20 LTS** (or .NET 8 if you prefer).
6. Operating System: **Linux**.
7. Region: same region.
8. Plan: **asp-lab**.
9. **Review + create** → **Create**.

Note the default URL: `https://<name>.azurewebsites.net`.

### CLI

```bash
az webapp create \
  --resource-group rg-appservice-lab \
  --plan asp-lab \
  --name app-contoso-lab-UNIQUE \
  --runtime "NODE:20-lts"
```

## Step 4 — Deploy something

### Quick ZIP for Node

Create a folder with `package.json` and `server.js` (or use `az webapp up`). Example deploy:

```bash
# From your app folder after zip
az webapp deploy \
  --resource-group rg-appservice-lab \
  --name app-contoso-lab-UNIQUE \
  --src-path ./site.zip \
  --type zip
```

Or from VS Code: Azure extension → Deploy to Web App.

### Portal alternative

**Deployment Center** → connect GitHub (later article covers OIDC) or use **Advanced Tools (Kudu)** → ZIP deploy.

## Step 5 — Confirm HTTPS

1. Browse `https://<name>.azurewebsites.net`.
2. In the portal: **TLS/SSL settings** → **HTTPS Only** = On.
3. Minimum TLS version: **1.2**.

You now have a public site. Do not put secrets in Application settings as plain text if you can avoid it—use Key Vault references next.

## Step 6 — Useful first settings

| Setting | Recommendation |
| --- | --- |
| HTTPS Only | On |
| Min TLS | 1.2 |
| Always On | On when not on Free (avoids idle unload) |
| Application Insights | Enable for labs that will grow |

```bash
az webapp update \
  --resource-group rg-appservice-lab \
  --name app-contoso-lab-UNIQUE \
  --https-only true

az webapp config set \
  --resource-group rg-appservice-lab \
  --name app-contoso-lab-UNIQUE \
  --min-tls-version 1.2
```

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| Name not available | App name must be globally unique |
| 503 / blank after deploy | Wrong startup command or runtime |
| FTP credentials in git | Never commit publish profiles with secrets |
| Free tier surprises | No deployment slots; limited outbound |
| Wrong region | Plan and app must align with your plan region |

## Tear down

```bash
az group delete --name rg-appservice-lab --yes --no-wait
```

## What you should remember

1. **Plan = compute; Web App = site + URL.**
2. **HTTPS only** from day one.
3. **Unique name** for `*.azurewebsites.net`.
4. **Deploy via ZIP/CI**, not hand-edited files on the server.
5. **Delete the lab RG** so Free/Basic meters stop.

Next: register an Entra app and sign users into this site with a matching redirect URI.
