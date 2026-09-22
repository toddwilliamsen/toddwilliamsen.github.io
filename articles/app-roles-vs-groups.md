---
layout: article
title: App Roles vs Group Claims for Authorization
topic: How-To
category: how-to
level: Beginner
diagram: /images/howto-app-roles-vs-groups.svg
summary: Decide when to authorize with Entra app roles versus group object IDs in the token—and how to avoid oversized group claims.
author: Todd Williamsen
date: 2025-10-11
description: Beginner guide comparing Entra ID app roles and group claims for app authorization—manifest roles, assignments, token claims, and pitfalls.
permalink: /articles/app-roles-vs-groups/
---

Authentication answers “who are you?” Authorization answers “what can you do?” In Entra-backed apps you usually choose between **app roles** (defined on the registration) and **group claims** (directory groups emitted into the token).

Default recommendation for application permissions: **prefer app roles**. Use groups when the business already lives in security groups and you can keep claim size under control.

<figure>
  <img src="{{ '/images/howto-app-roles-vs-groups.svg' | relative_url }}" alt="Token carries roles claim and optional groups claim into your API for authorization">
  <figcaption>Figure 1. Roles are app-specific; groups are directory-wide and can bloat tokens.</figcaption>
</figure>

## What you need before you start

- An existing app registration (UI or API)
- Rights to edit the app manifest / App roles blade
- About **20 minutes**
- Clarity on two or three roles (e.g. `Reader`, `Contributor`, `Admin`)

## Step 1 — Define app roles

### Portal

1. Open the **API** (or app) registration → **App roles** → **Create app role**.
2. Display name: `Reader`.
3. Allowed member types: **Users/Groups** (and Applications if you need app-only).
4. Value: `Reader` (this string lands in the `roles` claim—keep it stable).
5. Description: short and accurate.
6. Enable → **Apply**.

Repeat for `Contributor` and `Admin`.

Manifest snippet shape (do not paste blindly—use the blade):

```json
{
  "appRoles": [
    {
      "allowedMemberTypes": ["User"],
      "displayName": "Reader",
      "id": "<generate-guid>",
      "isEnabled": true,
      "description": "Read data",
      "value": "Reader"
    }
  ]
}
```

## Step 2 — Assign users or groups to roles

1. Entra ID → **Enterprise applications** → your app.
2. **Users and groups** → **Add user/group**.
3. Select a user or security group → select role `Reader` → assign.

Assigning a **group** to an app role is usually better than assigning hundreds of users. The token still gets `roles: ["Reader"]` for those members when configured correctly.

## Step 3 — Emit and check the roles claim

For delegated user tokens, ensure the app requests tokens for **this** API audience. The access token should include:

```text
"roles": ["Reader"]
```

In your API, authorize on `roles` (or map to policies). Do not invent a parallel permission system that ignores the claim.

## Step 4 — When to use group claims

Enable groups only if you must:

1. App registration → **Token configuration** → **Add groups claim**.
2. Choose **Security groups** (or Groups assigned to the application—prefer this to reduce size).
3. Understand **group overage**: if a user is in too many groups, Entra may omit the list and force a Graph call.

Optional: emit group **names** only in carefully controlled tenants; object IDs are safer and stable.

## Comparison cheatsheet

| Concern | App roles | Group claims |
| --- | --- | --- |
| Defined where | App registration | Entra groups |
| Claim | `roles` | `groups` |
| Token size | Small | Can be huge |
| Portability | Clear per-app contract | Coupled to directory |
| Best for | App authorization | Org-wide access patterns |

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| No `roles` in token | Role not assigned; wrong audience; ID token vs access token mix-up |
| Token too large | Full group claim on busy users |
| Authorization works in one env only | Different enterprise app assignments |
| Using display names | Rely on role `value` / group OID, not display strings |

## What you should remember

1. **Prefer app roles** for “can this user call this feature?”
2. **Role `value` is the contract**—do not rename casually.
3. **Assign via enterprise app**—registration alone is not enough.
4. **Group claims need a size strategy**—assigned groups or Graph fallback.
5. **Authorize on the access token** meant for your API audience.

Next: keep connection strings and client secrets out of App Service plain settings with Key Vault.
