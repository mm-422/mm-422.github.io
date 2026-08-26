---
title: "CVE-2026-6726 Investigation ft. IBM TSS"
date: 2026-08-20 00:00:00 +/-TTTT
categories: [research, cryptography]
tags: [amd, tpm, crypto]     # TAG names should always be lowercase
image: "/assets/images/ryzen-tpm/ryzen tpm.png"
mermaid: true
---
``DOMAIN:`` Sysadmin | Vulnerability Mgmt<br>
``CONTEXT:`` The skills, tools, and methodology covered could be seen in system hardening routines, where an in-charge IT personnel would investigate recent security incidents, compile their findings, and then examine dev/prod systems for weaknesses based on the research, before following up with appropriate mitigations e.g. rolling out official patches.  

___

## Vulnerability Overview  
This project explores the recently reported (and patched) vulnerability concerning AMD's implementation of the TPM2 specification (fTPM) in its currently supported processors.  

To summarize, CVE-2026-6726 revolves around a flaw in TPM logic where a "Use-After-Free" scenario is forced (via complex auth sessions, type conversions, etc.). An attacker with local privileged access could then trick the TPM into certifying a false or weakly-generated key with a valid Attestation Key (AK). This would allow them to obtain genuine certificates for their forged identity from TPM-aware Certificate Authorities due to them trusting data rooted in valid AKs.

To demonstrate what the "exploit chain" might look like, a TPM Simulator was set up alongside a compatible software stack (TPM tools & libraries) for testing in a controlled environment (Debian VM). This is complemented with a write-up in a later section, showcasing where the vulnerabilities might be rooted and what mitigations are currently available and/or recommended.  

___

## Key Concepts  
### ♦️ Trusted Platform Module
- Specialized hardware chip (or integrated processor) on a motherboard.
- Acts as a secure digital vault for encryption keys, passwords, digital certs.
- Keep security tasks isolated from OS and vulnerable software.
- Two Primary Functions:
  - **Secure Key Generation**
    - TPM is commonly used to generate cryptographic keys securely.
    - Key creation is such that the private half of the key never leaves TPM.
    - TPM can also encrypt, sign, and certify keys.
    - Different types of keys are generated under different hierarchies.
    - Hierarchy is a logical collection of data.
  
  - **System Attestation**
    - TPM can be used to capture host system state.
    - This is done by recording the values stored in the TPMs registers.
    - These registers are called Platform Configuration Registers (PCR).
    - Part of the process of "vouching" for a system's trustworthiness involves reporting PCR values.


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

___

## Tools
```
• Debian 13.6 via VirtualBox v7.2.12
• IBM TPM2 Simulator and TSS
```
![debian](/assets/images/ryzen-tpm/debian-vm.avif)
_Debian 13.6 running on VirtualBox._  

___

## Demo & Analysis
### ♦️ IBM TPM2 Setup
**Prerequisites**
```
• OpenSSL 3.1 x or newer incl. dev package.
• CMake or similar build tool.
• IBM's Software TPM2.0 (via git).
  LINK: "git clone https://git.code.sf.net/p/ibmswtpm2/tpm2 ibmswtpm2-tpm2"
```
**Installation**  
```
• Build for Linux will create an executable, "tpm_server".
• Terminal:
  $ cd ~/Downloads/ibmswtpm2-tpm2/src
  $ make
```

![terminal-tpm](/assets/images/ryzen-tpm/TPM.avif)
_Root of the IBM TPM2 Source._

### ♦️ IBM TSS Setup
**Prerequisites**
```
• OpenSSL 3.1 x or newer incl. dev package.
• CMake or similar build tool.
• IBM's TPM2.0 TSS (via git).
  LINK: "git clone https://git.code.sf.net/p/ibmtpm20tss/tss ibmtpm20tss-tss"
```
**Installation**  
```
• Build will create binaries used to communicate with TPM simulator.
• After building, run the regression test with reg.sh
• Terminal:
  $ LD_LIBRARY_PATH=/usr/local
  $ cd ~/Downloads/ibmtpm20tss-tss/utils
  $ autoreconf -i
  $ ./configure --prefix=${HOME}/local --disable-tpm-1.2 --disable-hwtpm
  $ make clean
  $ make
  $ make install
• Regression test:
  $ cd ~/Downloads/ibmtpm20tss-tss/utils
  $ ./reg/sh -a
```

![terminal-tss](/assets/images/ryzen-tpm/TSS.avif)
_These binaries can be run straight from the terminal._

### ♦️ Attestation Workflow
**Foreword**<br>
To provide a clearer picture of where the TPM vulnerability lies, a demonstration of remote attestation was outlined to highlight all the "moving parts" of the workflow at every step.

We assume a scenario where a client machine is attempting to connect to a restricted VPN server. It needs to prove its identity and trustworthiness to a Certificate Authority (CA). 

Remember that the Endorsement Key (EK), derived from the Endorsement Primary Seed (EPS), is burned in at the factory. Any keys generated within this hierarchy is implicitly trusted because the EPS cannot be forged.

Since a system can only have one TPM chip active, we can use child keys generated from the Endorsement Hierarchy as a form of "ID" of the system.

However, the CA will not accept any key generated by the TPM without security guarantees. This is where the process of attestation comes in. Essentially, the TPM generates a set of data to go along with a key that "vouches" for the system's state to prove its trustworthiness.

This is done via the TPM2_Certify command, which takes a child key and wraps it (encrypt) with what is known as an Attestation Key (AK), in order to output a cryptographic structure. The client will then send this structure and the public component of the wrapped key to the CA in order to be verified.

**Setup**<br>
We initiate the TPM simulator by executing the "tpm_server" binary and allow it to run in the background. The status can be observed via the terminal directly.

In order to interact with this "software TPM" and send commands, we launch the dedicated binaries that were compiled as part of the IBM TPM Software Stack installation step. We begin by initializing the simulator and generating a parent key in the Endorsement Hierarchy:

```
# Initialization Step
$ startup -v
```

![tpm-running](/assets/images/ryzen-tpm/tpm-server.avif)
_TPM Simulator is successfully running in the background._  

```
# Generating a Key
$ createprimary -hi -e -st
```
_Note: I've added the /utils folder to the Debian system path. This allows me to run the binaries as if they were commands directly from the terminal._  

We can now create new child keys from this hierarchy.  

Next, we require an Attestation Key along with a sample key that may be used for purposes of authenticating to a VPN server e.g. RSA signing key.
```
# Generate AK and load it into memory
$ create -hp 80000000 -opu AK_pub.data -opr AK_priv.data -sir
$ load -hp 80000001 -ipu AK_pub.data -ipr AK_priv.data

# Clear memory slot for next workflow
$ flushcontext -ha 80000000

# Create new hierarchy for RSA sample key
$ createprimary -hi o -st
$ create -hp 80000000 -des -si -opu pub.bin -opr priv.bin
$ load -hp 80000002 -ipu -pub.bin -ipr priv.bin
```
_Note: The IBM TPM Simulator can only hold a maximum of 3 items in memory at any one time._

We now have both the AK and a sample key loaded into the TPM's memory and can run the Certify command to create the bundle of data that would be sent to the CA.
```
# Input the handles for the sample key and AK respectively
$ certify -ho 80000002 -hk 80000001 -os sign.data -oa attest.data
```

![items-list](/assets/images/ryzen-tpm/ls-la.avif)
_A list of all the data generated and saved locally._

### ♦️ The Vulnerability
In an attack scenario, the workflow is broken at around the certification step. Essentially, the attacker's end goal is to get their forged key certified via TPM2_Certify in order to trick the CA into believing that the key is internally generated by the TPM. Remember that the CA trusts the AK and all data wrapped by it.

The attacker achieves this by manipulating objects in the TPM's memory slots in order to force a "Use-After-Free" scenario. In other words, a memory slot is improperly flushed or overwritten with new data that inherits some attributes of the old data that it replaced.  

When the attacker runs the TPM2_Certify command with their forged key as the input, the TPM simply reads the memory slots and wraps the compromised key with a valid AK. Because the signature on the resulting cryptographic structure comes from an untampered AK, the TPM-aware CA trusts it completely.  

The attacker can now export the resulting data and use it to impersonate a secure endpoint against any machine in the world, bypassing hardware isolation entirely.

___

## Mitigation & Remediation
As the flaw lies in the TPM2 spec by the Trusted Computing Group (TCG) and AMD's proprietary implementation of it, a BIOS firmware upgrade is required to fully resolve the issue. These have been made available via official channels since May 2026, rendering workarounds or temporary fixes unnecessary.  

Download the latest AGESA microcode from the BIOS download/update pages of your specific motherboard vendor. These are the patched versions for each Zen architecture:

```
• ComboAM4PI 1.0.0.11 on May 18 for Ryzen 3000.
• ComboAM4v2PI 1.2.0.12 on May 27 for Ryzen 4000 & 5000.
• ComboAM5PI 1.3.0.1b and 1.2.0.3k for Ryzen 7000, 8000, 9000.
```
