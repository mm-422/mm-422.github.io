---
title: "Ryzen TPM [WIP]"
date: 2026-01-05 00:00:00 +/-TTTT
categories: [research, reverse engineering]
tags: [windows, reverse engineering, ghidra]     # TAG names should always be lowercase
image: "/assets/images/ryzen-tpm/ryzen tpm.png"
---
``DOMAIN:``  | <br>
``REAL-WORLD CONTEXT:`` .

___

## Vulnerability Overview
- High level summary of issue.
- CVE ID, affected versions, privilege requirements etc.

## Key Concepts
- Explain TPM transient objects.
- Volatile RAM slots.
- Key wrapping.
- Context save/load mechanics.

## Tools & Setup
- Debian VM
- IBM TPM Simulator and TPM SS install
- Verification

## Demo & Analysis
- Step by step walkthrough.
- Command logs showing normal context mgmt vs sequence used to trigger context/slot reuse.
- Observation of anomaly.
- Terminal side by side or log output showing corrupted slot state or auth bypass

## Mitigation & Remediation
- How vendors patch this upstream in reference stack
- e.g. strict sequence counter validation, zeroizing slot mem on context save/flush

