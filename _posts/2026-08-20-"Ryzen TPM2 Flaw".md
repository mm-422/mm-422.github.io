---
title: "Investigating CVE-2026-6726 ft. IBM TSS [WIP]"
date: 2026-08-20 00:00:00 +/-TTTT
categories: [research, cryptography]
tags: [amd, tpm, crypto]     # TAG names should always be lowercase
image: "/assets/images/ryzen-tpm/ryzen tpm.png"
---
``DOMAIN:`` Sysadmin | Vulnerability Mgmt<br>
``CONTEXT:`` The skills, tools, and methodology covered could be seen in a system hardening routine, where an in-charge IT personnel would investigate recent reports of breaches, compile their findings, and then examine dev/prod systems for weaknesses based on the research. They might also follow up with application of appropriate mitigations e.g. rolling out official patches.  

___

## Vulnerability Overview  
This project explores the recently reported (and patched) vulnerability concerning AMD's implementation of the TPM2 specification (fTPM) in its currently supported processors.  

To summarize, CVE-2026-6726 revolves around a flaw in TPM logic where a "Use-After-Free" scenario is forced (via complex auth sessions, type conversions, etc.). An attacker with local privileged access could trick the TPM into certifying a false or weakly-generated key with a valid Attestation Key (AK), which would allow them to obtain genuine certificates for their forged identity from TPM-aware Certificate Authorities (due to them trusting data rooted in a valid AK).  

To demonstrate what the "exploit chain" might look like, a TPM Simulator was set up alongside a compatible software stack (TPM tools & libraries) for testing in a controlled environment (Debian VM). This is complemented with a write-up in a later section, showcasing where the vulnerabilities might be rooted and what mitigations are currently available and/or recommended.  

## Key Concepts  
### ♦️ Trusted Platform Module
- Specialized hardware chip (or integrated processor) on a motherboard.
- Acts as a secure digital vault for encryption keys, passwords, digital certs.
- Keep security tasks isolated from OS and vulnerable software.
- Two Primary Functions:
```
1. Secure Key Generation
• TPM is commonly used to generate cryptographic keys securely.
• Key creation is such that the private half of the key never leaves TPM.
• TPM can also encrypt, sign, and certify keys.
• Different types of keys are generated under different hierarchies.
• Hierarchy is a logical collection of data.

2. System Attestation
• TPM can be used to capture host system state.
• This is done by recording the values stored in the TPMs registers.
• These registers are called Platform Configuration Registers (PCR).
• Part of the process of "vouching" for a system's trustworthiness involves reporting PCR values.
```

### ♦️ Objects, Handles, Slots
- TPMs possess a small amount of persistent memory called NVDATA.
- They also possess spaces in memory for storing data temporarily.
- These "working memory" spaces are called **slots.**
- The stored data items themselves are referenced as **objects.**
- Each object stored in a slot has an address a.k.a. **handle.**  

### ♦️ Hierarchies & Key Creation
- Key generation & signing within TPM is tied to a system of hierarchies.
- Each hierarchy begins with a seed, followed by primary/parent key.
- The primary key is then used to create child keys.
- Four main hierarchies:
  - **Owner/Storage Hierarchy**
    - The user/owner is responsible for secure key creation.
    - Seed can be reset or cleared when re-installing OS.
  - **Platform Hierarchy**
    - Reserved for objects created and certified by OEM (ASUS, MSI, Dell, etc.).
    - Seed for this hierarchy is generated at manufacturing time.
    - Can be reset by manufacturer.
  - **Endorsement Hierarchy**
    - Reserved for objects created and certified by TPM manufacturer.
    - Seed is burned in at factory. 
  - **Null Hierarchy**
    - Reserved for ephemeral keys.
    - Seed is regenerated each time host reboots.  

### ♦️ Remote Attestation  
- This is one of numerous workflows that can be performed by TPM.
- Involves verifying a system state in order to vouch for its trustworthiness.
- This enables it to prove its identity to external servers.
- Begins with creation of Attestation Key, derived from Endorsement Key.
- The AK is then used to certify (wrap) a key that may be used for encryption/signing.
- The process of certification generates a cryptographic structure.
- The client sends this structure and the public component of the wrapped key to the server to be verified.

## Tools
```
• Debian 13.6 via VirtualBox v7.2.12
• IBM TPM2 Simulator and TSS
```

## Demo & Analysis
We will begin with a short guide on setting up the TPM simulator and software stack required to communicate with it.<br> Note: Different TPM simulators have different dependencies and compatibilities. For this research, the IBM implementation for both the simulator and software stack was used.
### ♦️ IBM TPM2 Setup
### ♦️ IBM TSS Setup
### ♦️ Legitimate Remote Attestation Workflow
### ♦️ Where CVE-2026-6726 Intervenes


## Mitigation & Remediation
AGESA Mitigations rolled out in May.
```
• ComboAM4PI 1.0.0.11 on May 18 for Ryzen 3000.
• ComboAM4v2PI 1.2.0.12 on May 27 for Ryzen 4000 & 5000.
• ComboAM5PI 1.3.0.1b and 1.2.0.3k for Ryzen 7000, 8000, 9000.
```
