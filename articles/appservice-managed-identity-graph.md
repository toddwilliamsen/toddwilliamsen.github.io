---
layout: article
title: App Service Managed Identity Instead of Client Secrets
topic: How-To
category: how-to
level: Intermediate
diagram: /images/howto-appservice-managed-identity-graph.svg
summary: Enable App Service managed identity, grant least-privilege Microsoft Graph (or API) app roles, and acquire tokens with DefaultAzureCredential—no client secret.
author: Todd Williamsen
date: 2025-10-14
description: Intermediate guide to replace Entra client secrets with App Service managed identity for Microsoft Graph and Azure resource access.
permalink: /articles/appservice-managed-identity-graph/
---

Client secrets for “the web app calling Graph” expire, leak into logs, and tempt people to paste them into pipelines. **Managed identity** gives App Service a service principal Entra can mint tokens for—**no secret to rotate in your config**.

<figure>
  <img src="{{ '/images/howto-appservice-managed-identity-graph.svg' | relative_url }}" alt="App Service managed identity gets tokens from Entra to call Graph or other resources">
  <figcaption>Figure 1. The platform identity is the credential.</figcaption>
</figure>

## What you need before you start

- App Service Web App
- Entra rights to grant Graph **application** permissions (admin consent)
- About **30 minutes**
- Clear on **app-only** vs **delegated**: MI uses app-only tokens

## Step 1 — Enable managed identity

```bash
az webapp identity assign \
  --resource-group rg-appservice-lab \
  --name <your-app>
```

Prefer **system-assigned** for a single app. Use **user-assigned** when multiple apps share one identity intentionally.

## Step 2 — Grant Graph application permissions to the identity

Managed identities do not use the App registrations “API permissions” UI the same way multi-tenant apps do. Assign **app roles** on Microsoft Graph to the MI’s service principal.

Example with Microsoft Graph PowerShell or Azure CLI + REST is common. Portal path many teams use:

1. Entra ID → **Enterprise applications** → search your Web App name (system-assigned MI appears after enable).
2. Or use Graph to add app role assignments for permissions like `User.Read.All` **only if required**.

Minimal lab: if you only need Key Vault or Azure RBAC resources, **skip Graph** and assign Azure RBAC instead—still MI, still no secret.

For Graph app-only:

```bash
# Conceptual: assign a Graph app role to the MI principal
# Prefer documented scripts from Microsoft for your permission set
# Always admin-consent and document why each permission exists
```

Practical checklist:

1. Identify the MI **object id**.
2. Assign the **smallest** Graph application permission that works.
3. Admin consent.
4. Wait a few minutes for replication.

## Step 3 — Acquire tokens in code

.NET:

```csharp
var credential = new DefaultAzureCredential();
var token = await credential.GetTokenAsync(
  new TokenRequestContext(new[] { "https://graph.microsoft.com/.default" }));
```

Python / Node equivalents use Azure Identity libraries with the same `.default` scope for app-only.

On App Service, `DefaultAzureCredential` finds the managed identity automatically. Locally, use Visual Studio / Azure CLI / env vars—**never** commit secrets for local overrides.

## Step 4 — Call Graph (or prefer resource-specific APIs)

```http
GET https://graph.microsoft.com/v1.0/users?$select=id,displayName
Authorization: Bearer <token>
```

If you only need Azure resources (Storage, SQL with MI, Key Vault), prefer those RBAC assignments over broad Graph permissions.

## Step 5 — When you still need an app registration

| Scenario | Approach |
| --- | --- |
| User sign-in | App registration + redirect URIs |
| App-only as MI | MI + app role assignment |
| CI from GitHub | Federated credential on an app reg |
| Call custom API app-only | API app roles + MI assignment |

Do not create a client secret “because the sample had one.”

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| 401 from Graph | App role not assigned / not consented |
| Works locally, fails in Azure | Local used your user token; Azure needs app roles |
| Overpowered MI | `Directory.ReadWrite.All` for a read UI |
| Confused with Easy Auth | Easy Auth is user auth; MI is app identity |

## What you should remember

1. **MI replaces client secrets** for Azure-hosted app-only calls.
2. **Least privilege app roles**—especially on Graph.
3. **`.default` scope** for app-only token requests.
4. **DefaultAzureCredential** in Azure; separate local auth story.
5. **User sign-in still needs an app registration**—different problem.

Next: compare client secrets, certificates, and federated credentials explicitly.
