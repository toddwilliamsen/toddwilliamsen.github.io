---
layout: article
title: Quantum Helix
topic: Series · Part 4
summary: A quantum-kernel anomaly engine for identity, network, and governance telemetry—built to test whether quantum techniques can offer practical value without quantum hardware.
author: Todd Williamsen
date: 2026-06-22
description: Quantum Helix combines PCA, PennyLane quantum kernels, and classical baselines into a hybrid anomaly engine for multi-cloud security telemetry.
permalink: /articles/quantum-helix/
---

Quantum Helix is the culmination of the work across SOAR automation, Zero Trust identity enforcement, and Landing Zone telemetry architecture.

<figure>
  <img src="{{ '/images/quantum-helix.svg' | relative_url }}" alt="Quantum Helix pipeline from AWS and Azure ingest through CIM normalization, PCA vectors, hybrid ensemble scoring, and SIEM output">
  <figcaption>Figure 1. Ingest → normalize → reduce → hybrid score → ASFF / CEF alerting</figcaption>
</figure>

## Why Quantum Helix exists

After building:

- Automated triage
- Enrichment pipelines
- Identity-driven segmentation
- Landing Zone telemetry models

I needed a detection engine capable of:

- Reducing high-dimensional telemetry
- Classifying anomalies deterministically
- Evaluating identity signals quickly enough to matter
- Scaling across cloud workloads

Traditional ML struggled on the subtle, noisy cases—especially identity anomalies that do not look suspicious at first glance.

## Quantum kernel approach

Lately, I’ve been exploring whether quantum techniques can offer anything practical for cloud threat detection—even without access to real quantum hardware. With some hype around “quantum cybersecurity,” I wanted to see if quantum kernels running on classical simulators behave differently on subtle cloud anomalies.

To test that, I built a pipeline that:

- Pulls in AWS CloudTrail and Azure Activity / NSG logs
- Normalizes them into a Common Information Model
- Runs them through StandardScaler and PCA
- Scores with a hybrid ensemble: Isolation Forest, RBF SVM, and a PennyLane QSVM quantum kernel

Classical models handle the straightforward anomalies. More complex edge cases get evaluated by the quantum kernel. An ensemble layer combines the scores so I can see where the detectors disagree.

## Unified detection

Quantum Helix integrates:

- SOAR enrichment context
- Zero Trust identity posture
- Landing Zone telemetry
- Governance signals

On the engineering side, a lightweight SOC-style console—Flask with SSE streaming, React with Vite, and SQLite with SQLAlchemy—keeps the experiment operable: triage, cases, and live scoring in one place.

## Still early

The most interesting part has been watching where the quantum kernel diverges from the classical detectors. Still early, but it has been a useful way to explore a concrete question:

**Is there any practical quantum value in cloud security today, even without quantum hardware?**

[View Quantum Helix on GitHub →](https://github.com/toddwilliamsen/quantumhelix)

---

**Series:** [← Azure Landing Zone Security]({{ '/articles/azure-landing-zone-security/' | relative_url }}) · Part 4 of 4 · [← Back to writing]({{ '/#writing' | relative_url }})
