---
layout: article
title: Zero Trust in Azure
topic: Zero Trust
summary: Zero Trust in Azure is a control loop across identity, devices, networks, and continuous verification—explained so both architects and CISOs can pressure-test it.
author: Todd Williamsen
date: 2026-07-28
description: A practical Zero Trust guide for Azure covering Conditional Access, device trust, segmentation, and continuous verification with CISO decision criteria.
permalink: /articles/zero-trust-azure/
---

Zero Trust becomes useless when it turns into a slogan or a product checklist. In Azure, it is valuable only when it shows up as enforceable decisions:

**Who can sign in, from which device, to which resource, under which conditions—and what changes when risk signals change?**

That is a control loop, not a one-time project. Architects implement the controls. CISOs should be able to follow the story and ask whether the loop is closed.

<figure>
  <img src="{{ '/images/zero-trust-azure.svg' | relative_url }}" alt="Zero Trust control loop diagram with identity, device, network, and signals surrounding a resource access grant">
  <figcaption>Figure 1. Zero Trust in Azure is authenticate, authorize, observe, and adapt—continuously.</figcaption>
</figure>

## What Zero Trust is not

- Not “we turned on MFA.”
- Not “we have a firewall.”
- Not “we bought a Zero Trust platform.”
- Not “VPN is gone, so we are done.”

MFA, segmentation, and modern access tools matter. They are insufficient alone. Zero Trust fails when each control is strong in isolation and weak in combination—for example, strong identity on a non-compliant device reaching a flat network with weak session monitoring.

## The four decisions that matter

### 1. Identity: verify explicitly

Conditional Access is the primary enforcement point in Microsoft Entra ID.

**Technical baseline**

- MFA for all users; phishing-resistant methods for privileged roles
- Risk-based controls for unfamiliar sign-ins and elevated risk
- Least-privilege RBAC, with PIM for standing admin elimination
- Separate control paths for workforce, guests, and workload identities

**CISO translation**

If privileged access can still be obtained with password + SMS from an unmanaged laptop, identity verification is incomplete—no matter how many Zero Trust slides exist.

### 2. Device and session trust: raise the bar for valuable data

A verified identity on a weak device is still a weak session.

**Technical baseline**

- Device compliance requirements for access to sensitive apps
- Application protection policies for mobile / BYOD scenarios
- Session controls for browser and unmanaged endpoints when business access is still required
- Stronger assurance for admin portals and production management planes

**CISO translation**

Ask which crown-jewel applications can be reached from devices your organization does not control. If the answer is “most of them,” Zero Trust is aspirational.

### 3. Network: contain, do not replace identity

Network controls reduce the value of a stolen session. They do not replace authentication and authorization.

**Technical baseline**

- Segmentation aligned to application trust boundaries
- Private endpoints and controlled egress for platform services
- No broad flat networks where one compromised workload can reach everything
- Inspection and logging on paths that matter for forensics

**CISO translation**

Network Zero Trust is working when lateral movement is difficult and explainable—not when every subnet merely has a different name.

### 4. Signals: continuous verification

Tokens age. Risk scores move. Devices fall out of compliance. Access that was right at 9:00 a.m. may be wrong at 9:40 a.m.

**Technical baseline**

- Entra ID Protection / risk signals feeding Conditional Access
- Defender for Cloud and endpoint signals informing response
- Microsoft Sentinel correlation across identity, device, and cloud resource activity
- Playbooks that can revoke sessions, disable tokens, or force re-authentication

**CISO translation**

Continuous verification means the organization can change access based on new evidence without waiting for the next quarterly access review.

## A reference access story

Use one sensitive workload—finance system, patient data platform, source-code environment, or identity admin portal—and complete this story end to end:

1. **Identity path:** How is the user authenticated? Which CA policies apply?
2. **Device bar:** What device state is required? What happens on BYOD?
3. **Network path:** Is the service private? How does egress work?
4. **Authorization:** Is access least-privilege and time-bounded for admins?
5. **Signal path:** Which alert fires if risk spikes mid-session?
6. **Response path:** Who can revoke access, and how fast?

If any chapter is missing, Zero Trust for that workload is incomplete. This exercise is more useful than a generic maturity score because it exposes real operational gaps.

## Implementation sequence that avoids theater

1. **Protect privileged access first.** Admin accounts, PIM, phishing-resistant MFA, device requirements for admin portals.
2. **Protect crown-jewel apps next.** Conditional Access + device/session controls on the systems that would materially harm the business.
3. **Remove standing privilege and broad network trust.** Shrink what a stolen session can do.
4. **Close the loop with detection and response.** Identity risk, impossible patterns, token theft indicators, and rapid revocation.
5. **Expand outward.** Only after the high-value paths are enforced and monitored.

This order matters. Broad user MFA without privileged-access hardening leaves the highest-value targets exposed. Network redesign without identity enforcement creates expensive complexity with limited risk reduction.

## Metrics a CISO can trust

| Question | Useful metric |
| --- | --- |
| Are admins still over-privileged? | Standing vs. eligible privileged roles |
| Are weak sessions reaching sensitive apps? | Sensitive-app sign-ins from non-compliant devices |
| Is lateral movement constrained? | Critical resources reachable without private path / without CA |
| Can we adapt to risk? | Median time to revoke session after high-risk signal |
| Is this becoming culture or exception? | Aging Conditional Access exclusions |

Avoid vanity metrics such as “number of Zero Trust tools deployed.” Prefer metrics that show enforcement and response.

## Board-ready takeaway

Zero Trust in Azure is credible when access to valuable systems requires strong identity, sufficient device assurance, limited network blast radius, and continuous signal-driven adjustment. If leadership cannot explain that story for the organization’s most sensitive workloads, the program is still in presentation mode.
