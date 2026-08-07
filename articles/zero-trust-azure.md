---
layout: article
title: Zero Trust in Azure
topic: Cloud Security
category: cloud-security
diagram: /images/zero-trust-azure.svg
summary: Identity-driven segmentation, continuous verification, and Conditional Access enforcement—and why identity telemetry needed a better anomaly engine.
author: Todd Williamsen
date: 2026-06-08
description: Zero Trust in Azure through Entra ID, continuous verification, and network segmentation, leading into high-dimensional identity anomaly challenges.
permalink: /articles/zero-trust-azure/
---

Zero Trust is not a product—it’s an operational discipline. After building SOAR automation, I focused on strengthening identity boundaries and enforcing continuous verification across Azure workloads.

<figure>
  <img src="{{ '/images/zero-trust-azure.svg' | relative_url }}" alt="Zero Trust control loop diagram with identity, device, network, and signals surrounding a resource access grant">
  <figcaption>Figure 1. Authenticate, authorize, observe, and adapt—continuously.</figcaption>
</figure>

## Identity as the control plane

Everything begins with Microsoft Entra ID:

- Conditional Access
- Device compliance
- Risk-based access
- Privileged Identity Management

Identity signals became the most valuable telemetry source in the environment—not because they replaced network or endpoint data, but because they answered who was acting, under what conditions, and with what privilege.

## Continuous verification

Zero Trust requires evaluating identity posture in real time:

- Login patterns
- Device health
- Network location
- Privilege elevation
- Behavioral anomalies

That reinforced the need for a detection engine capable of evaluating identity anomalies quickly enough to matter during an active session—not only in a next-day report.

## Segmentation

Workloads were segmented using:

- Application Security Groups
- Network Security Groups
- Private Endpoints
- Hub-and-spoke routing

Identity and network signals formed a unified behavioral model: access decisions at the control plane, containment at the data plane.

## The limitation

Identity telemetry is high-dimensional and extremely noisy. Traditional ML struggled to classify anomalies without false positives.

I needed:

- Dimensionality reduction
- Kernel-based classification
- Deterministic anomaly scoring

That led directly into Landing Zone telemetry analysis—where identity, network, and governance signals land together at scale.

