---
title: "Investigating CVE-2026-6726 ft. IBM TSS [WIP]"
date: 2026-08-20 00:00:00 +/-TTTT
categories: [research, cryptography]
tags: [amd, tpm, crypto]     # TAG names should always be lowercase
image: "/assets/images/ryzen-tpm/ryzen tpm.png"
---
``DOMAIN:`` Sysadmin | Vulnerability Mgmt<br>
``REAL-WORLD CONTEXT:`` Investigating reports, compiling findings, and then examining development/production systems for weaknesses based on research. Follow up with application of appropriate mitigations e.g. rolling out official patches.  

___

## Vulnerability Overview  
This project details the setup, installation, and demo of the IBM TPM2 Simulator and TSS alongside recent reports of high-severity vulnerabilities in AMD's implementation of the TPM2 spec (fTPM) in its processors.  

To summarize, CVE-2026-6726 revolves around a flaw in the TPM2.0 spec that allows a local/privileged attacker to obtain credentials for falsified TPM keys. This could then be used to falsify TPM attestations by tricking the TPM to certify false/weak keys with a valid AK, which would then slip past TPM-aware Certificate Authorities.

## Key Concepts
- Explain TPM transient objects.
- Volatile RAM slots.
- Key wrapping.
- Context save/load mechanics.

## Tools
```
• Debian 13.6 via VirtualBox v7.2.12
• IBM TPM2 Simulator and TSS
```
## Demo & Analysis
- Step by step walkthrough.
- Command logs showing normal context mgmt vs sequence used to trigger context/slot reuse.
- Observation of anomaly.
- Terminal side by side or log output showing corrupted slot state or auth bypass

## Mitigation & Remediation
AGESA Mitigations rolled out in May.
```
• ComboAM4PI 1.0.0.11 on May 18 for Ryzen 3000.
• ComboAM4v2PI 1.2.0.12 on May 27 for Ryzen 4000 & 5000.
• ComboAM5PI 1.3.0.1b and 1.2.0.3k for Ryzen 7000, 8000, 9000.
```

