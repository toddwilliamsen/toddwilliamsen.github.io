---
layout: article
title: App Service Deployment Slots from Scratch
topic: How-To
category: how-to
level: Intermediate
diagram: /images/howto-appservice-deployment-slots.svg
summary: Create a staging slot, deploy and warm it up, mark sticky settings, and swap into production with a rollback path.
author: Todd Williamsen
date: 2025-10-18
description: Intermediate how-to for Azure App Service deployment slots—create staging, sticky app settings, swap, smoke test, and roll back.
permalink: /articles/appservice-deployment-slots/
---

Deploying straight to production is how Friday nights go wrong. **Deployment slots** give you a live staging site on the same App Service plan: deploy, warm up, smoke test, **swap**. Connection strings and secrets that must not swap stay **slot-sticky**.

<figure>
  <img src="{{ '/images/howto-appservice-deployment-slots.svg' | relative_url }}" alt="Production and staging slots with a swap between them">
  <figcaption>Figure 1. Swap exchanges content; sticky settings stay put.</figcaption>
</figure>

## What you need before you start

- App Service plan on **Standard** or higher (slots are not on Free/Shared)
- An existing Web App
- About **30 minutes**
- A second set of settings for staging (URLs, Entra redirect URIs)

## Step 1 — Create a staging slot

### Portal

1. Web App → **Deployment slots** → **Add slot**.
2. Name: `staging`.
3. Clone settings from: production (optional starting point).
4. **Add**.

URL becomes `https://<app>-staging.azurewebsites.net`.

### CLI

```bash
az webapp deployment slot create \
  --resource-group rg-appservice-lab \
  --name <your-app> \
  --slot staging \
  --configuration-source <your-app>
```

## Step 2 — Mark sticky settings

Settings that must remain environment-specific:

- Entra client secrets / Key Vault URIs unique per slot
- Callback base URLs
- Feature flags for “this is staging”

Portal: **Configuration** → toggle **Deployment slot setting** on each sticky entry → **Save**.

Also register **both** redirect URIs on the Entra app (prod and staging hostnames).

## Step 3 — Deploy to staging only

```bash
az webapp deploy \
  --resource-group rg-appservice-lab \
  --name <your-app> \
  --slot staging \
  --src-path ./site.zip \
  --type zip
```

Point CI at the staging slot by default. Production receives code only via swap.

## Step 4 — Warm up and smoke test

1. Browse the staging URL; hit critical routes.
2. Confirm Easy Auth / MSAL against staging redirect URIs.
3. Check **Application Insights** / logs for startup errors.
4. Optional: configure **Auto swap** only if you fully trust warmup (many teams swap manually).

## Step 5 — Swap

### Portal

**Deployment slots** → **Swap** → source `staging` → target `production` → review sticky settings → **Swap**.

### CLI

```bash
az webapp deployment slot swap \
  --resource-group rg-appservice-lab \
  --name <your-app> \
  --slot staging \
  --target-slot production
```

If production misbehaves, **swap again** to roll back (same command, slots reversed in effect by swapping once more).

## Entra-specific notes

| Item | Action |
| --- | --- |
| Redirect URIs | Include `https://<app>-staging.azurewebsites.net/...` |
| App ID URI / CORS | Allow staging origin if SPA |
| Easy Auth | Configure auth per slot if settings differ |

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| Slots menu missing | Plan SKU too low |
| Prod secret leaked to staging | Setting not marked sticky |
| AADSTS50011 after swap | Redirect URI missing for one hostname |
| Cold start after swap | Insufficient warmup |
| Slot URL works, custom domain does not | Custom domain bound only to production |

## What you should remember

1. **Deploy to staging; swap to prod.**
2. **Sticky settings** protect per-environment secrets and URLs.
3. **Register every redirect hostname** in Entra.
4. **Warm up** before you swap.
5. **Swap again** to roll back quickly.

Next: deploy with GitHub Actions using OIDC federation—no client secret in GitHub.
