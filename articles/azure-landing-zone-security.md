---
layout: article
title: Azure Landing Zone Security
topic: Architecture
summary: How identity, network segmentation, governance, and security operations form one control plane inside an Azure Landing Zone.
author: Todd Williamsen
date: 2026-07-28
description: A technical guide to securing Azure Landing Zones across identity, network, governance, and SOC operations.
permalink: /articles/azure-landing-zone-security/
---

An Azure Landing Zone is not a subscription catalog. It is the set of decisions that determine whether the next workload inherits security by default—or inherits drift.

That means identity, network, policy, and telemetry designed as one system. The practical test is simple: **can a new workload land safely, be observed immediately, and fail closed when it drifts?**

If the answer is no, you have documentation. If the answer is yes, you have architecture.

<figure>
  <img src="{{ '/images/landing-zone-structure.svg' | relative_url }}" alt="Diagram of an Azure Landing Zone control plane with Entra ID above management groups, identity, network, and governance feeding security operations">
  <figcaption>Figure 1. Landing zone security is a control plane, not a checklist of products.</figcaption>
</figure>

## Why landing zones fail in production

Most landing-zone programs fail the same way: each domain is designed by a different owner, then “integrated” later.

- Identity is strong, but subscriptions can still be created with standing Owner rights.
- Networking is segmented on paper, but private endpoints and DNS are optional.
- Policy exists, but exemptions are informal and permanent.
- Sentinel is licensed, but diagnostic settings are incomplete.

None of those failures show up as a single red alert on day one. They show up as uneven blast radius, uneven detection, and slow incident response six months later.

## 1. Identity as the real control plane

Treat Microsoft Entra ID as the front door to the platform, not a directory bolted onto subscriptions after the fact.

### What good looks like

- **Privileged access is temporary.** Platform and subscription admins elevate through Privileged Identity Management (PIM), with approval, justification, and access reviews.
- **Workload identities are first-class.** Managed identities and service principals follow least privilege the same way humans do. Long-lived secrets are an exception with an expiry, not a pattern.
- **Conditional Access applies to admin paths.** MFA, device compliance, and risk-based controls are mandatory for privileged roles—not only for end users.
- **Break-glass accounts are monitored and rare.** Emergency access exists, is excluded carefully, and generates high-priority alerts when used.

<figure>
  <img src="{{ '/images/landing-zone-identity.svg' | relative_url }}" alt="Identity architecture showing humans, workloads, and partners flowing through Entra ID Conditional Access, PIM, and RBAC into platform and landing zone subscriptions">
  <figcaption>Figure 2. Every actor—human, workload, or partner—should pass through the same identity control plane.</figcaption>
</figure>

### Design checks

1. Who can create subscriptions, and is that path JIT or standing?
2. How many accounts hold permanent Owner or User Access Administrator?
3. Are workload identities reviewed with the same rigor as human admins?
4. What alert fires when a break-glass account signs in?

## 2. Segmentation that matches blast radius

Hub-and-spoke or Virtual WAN patterns only help if trust boundaries match business risk. Pretty topology diagrams are not the same as containment.

### Design rules that hold under pressure

- **Default deny east-west.** Application flows are allowed explicitly; peering is not a shortcut around inspection.
- **Private connectivity is the default** for platform services: Key Vault, Storage, SQL, and similar dependencies.
- **DNS, routing, and private endpoints are designed together.** Broken private DNS is how teams “temporarily” reopen public endpoints.
- **Internet-facing and corporate workloads stay separated.** Online spokes should not inherit corp trust.
- **Egress exits through a controlled path.** Central firewall or NVA inspection gives you both policy and forensics.

<figure>
  <img src="{{ '/images/landing-zone-network.svg' | relative_url }}" alt="Hub and spoke network diagram with corp, online, and sandbox spokes and traffic rules for default deny, private DNS, and hub egress">
  <figcaption>Figure 3. Network design should make lateral movement expensive and egress observable.</figcaption>
</figure>

### What “done” means operationally

If a workload is compromised, responders should be able to answer quickly:

- What else can it reach without a new credential?
- Where did its egress go?
- Which identity authorized the path?

If those answers require a week of discovery, segmentation was never finished.

## 3. Governance as executable policy

Architecture decisions that only live in a wiki will not survive delivery pressure. Azure Policy, management groups, and subscription vending should encode the decisions the organization has already made.

<figure>
  <img src="{{ '/images/landing-zone-governance.svg' | relative_url }}" alt="Management group hierarchy with policy effects, required baseline controls, and operating signals such as compliance dashboards and exemption workflows">
  <figcaption>Figure 4. Governance converts decisions into deny, deploy, and audit controls—with a real exemption process.</figcaption>
</figure>

### Baseline controls worth enforcing

| Control intent | Typical enforcement |
| --- | --- |
| Limit geographic risk | Allowed locations only |
| Prevent accidental exposure | Deny public storage / open management ports |
| Guarantee telemetry | DeployIfNotExists diagnostic settings |
| Guarantee posture management | Defender plans required on landing-zone subscriptions |
| Preserve ownership | Required tags: owner, cost center, data classification |
| Reduce standing privilege | Restrict Owner assignment patterns |

Exemptions will happen. Make them time-bound, ticketed, and visible. A permanent silent exemption is just unmanaged risk with better formatting.

## 4. Security operations wired into the platform

Detection cannot be a follow-on project. If telemetry is optional at subscription creation, coverage will be uneven by design.

<figure>
  <img src="{{ '/images/landing-zone-security-ops.svg' | relative_url }}" alt="Flow from subscriptions, workloads, and identity into diagnostic collection, Log Analytics, then Sentinel, SOAR, and ticketing">
  <figcaption>Figure 5. SOC capability depends on what the landing zone collects by default.</figcaption>
</figure>

### Minimum operating wiring

At subscription vending time, require:

- Diagnostic settings to a central Log Analytics workspace
- Defender for Cloud plans aligned to workload risk
- Activity logs and Entra sign-in/audit signals available to Microsoft Sentinel
- Ownership and support-group tags that routing/automation can trust
- A mapped escalation path for platform vs. workload incidents

SOAR playbooks only work if inventory, identity context, and ownership data are already reliable. Enrichment cannot invent clean CMDB data during an incident.

## 5. Operating model and evidence of progress

Measure outcomes, not slideware.

**Leading indicators**

- Percentage of subscriptions created through the approved vending path
- Percentage with complete diagnostic coverage
- Count of standing privileged roles vs. eligible PIM roles
- Policy compliance by management group, with aging exemptions

**Lagging indicators**

- Time to contain an identity or network incident
- Number of publicly exposed resources found by Defender / attack surface tools
- Repeat audit findings on the same control family

### A practical readiness test

Onboard a fictional application through the standard path:

1. Can the team get a subscription without standing Owner rights?
2. Does it inherit network, policy, and logging automatically?
3. Does an intentional misconfiguration (public storage, open NSG) get denied or alerted?
4. Does the SOC receive usable telemetry on day one?

Four yes answers means the landing zone is becoming an operating system for secure delivery. Anything less means security is still a negotiation.

## Closing

Azure Landing Zone security succeeds when identity, segmentation, governance, and SOC wiring are inherited—not requested. The secure path should be the default path, and drift should be visible before it becomes an incident.
