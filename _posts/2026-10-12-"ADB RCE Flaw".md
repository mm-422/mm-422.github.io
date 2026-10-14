---
title: "CVE-2026-0073 Investigation ft. ADB"
date: 2026-10-12 00:00:00 +/-TTTT
categories: [research, cryptography]
tags: [android, crypto, authentication]     # TAG names should always be lowercase
image: "/assets/images/adb-rce/adb-rce-thumbnail-3.png"
---
``DOMAIN:`` Sysadmin | Vulnerability Mgmt<br>
``CONTEXT:`` The findings and methodology covered here could be applied to patch management and system hardening routines. An IT personnel would investigate a relevant critical vulnerability, compile their findings, examine existing systems for weaknesses and follow up with appropriate solutions or mitigations e.g. rolling out official security patches.

___

## Executive Summary
CVE-2026-0073 details a critical logic flaw residing within Android's wireless debugging feature, specifically the TLS certificate verification routine.

A threat actor within network reach is able to initiate a remote connection to a target device with wireless debugging enabled via Android Debug Bridge. They then exploit a mishandling of comparison return values to bypass authentication. This enables the attacker to obtain a shell session on the device and execute arbitrary commands. No interaction from the user is required.

- Vulnerability: Cryptographic Logic Flaw resulting in Auth Bypass (RCE)
- Affected Component: Android Debug Bridge Daemon (adbd).
- Impact: Zero click RCE as shell user on vulnerable device within network reach.
- Severity: High (CVSS ~8.8)
- Remediation: Addressed in May 2026 Android Security Bulletin & Google Play System (Mainline) updates.
- <mark>LINK:</mark> https://nvd.nist.gov/vuln/detail/cve-2026-0073

This project explores the mechanisms involved leading to RCE with working demonstrations utilizing an Android Virtual Device (AVD) within Android Studio.

___

## Technical Summary
Android's `adbd` daemon (a background process responsible for executing commands) relies on a "mTLS handshake" process in order to establish secure communication channels for remote debugging work.

During the handshake, the client or attacker presents an X.509 certificate (contains a cryptographic key) to the target device's daemon. The daemon sifts through the list of authorized public keys stored locally under `/data/misc/adb/adb_keys` and invokes OpenSSL's `EVP_PKEY_cmp()` function to look for a match. A value is returned at the end of the routine to signify the result as follows:

- `1: Keys match.`
- `0: Keys do not match.`
- `-1: Error or unsupported key type.`

This return value is then received by a function within the daemon (found under daemon/auth.cpp) called `adbd_tls_verify_cert` which will determine either a successful authentication or otherwise based on that value.

In the case of the mTLS handshake process, if there is a mismatch between key types during the comparison phase (e.g. ECDSA or ED25519 key supplied but RSA key stored locally) then the `EVP_PKEY_cmp()` function will return a value of -1.

Because negative numbers evaluate to true in C/C++, the daemon (and subsequently the function `adbd_tls_verify_cert`) misinterprets the value of `-1` as "true", allowing for successful authentication.

This essentially means that the attacker need only supply a mismatched key type in order to trick the daemon into letting them through, bypassing cryptographic validation and obtaining shell level access via the wireless debugging feature.

___

## Tools
```
Host
• Fedora Workstation 44 kernel 7.1.5

VMs & Emulators
• Kali Linux 2026.2 via VirtualBox v7.2.14
• AVD via Android Studio 2026.1.4
• Pixel 9 virtual device running AOSP API Level 35 (Android 15)

Misc
• Python3.14 with cryptography library
• Android Debug Bridge (ADB) v1.0.41
```
![setup](/assets/images/adb-rce/1-setup.avif)
_AVD running on the left. Kali Linux VM on the right._

___

## Assessment
### ♦️ Setup
We begin by setting up the necessary environments, tools, and devices:
- A Kali Linux VM was used to host the tools and scripts.
- The target device is a Pixel 9 virtual device (Android Studio).<br>
_**Note:** If using a physical device, please ensure you've obtained the necessary permissions._
- A socat relay was set up on the host (Fedora Workstation) in order to bridge communications from the VM to the Pixel 9 virtual device.
```bash
# Sample socat relay
# Receives inbound data on 192.168.160.70
# Forwards to 127.0.0.1 or "localhost" on Fedora
socat TCP-LISTEN:40000,bind=192.168.160.70,fork TCP:127.0.0.1:40000
```
It is recommended to start with a fresh device if using an emulator via Android Studio to minimize instability and bugs.

If starting with a clean device with no prior connection history, the `adb_keys` file under the `/data/misc/adb/` directory will either be non-existent or empty. This is where public keys (RSA) are kept for the purposes of mTLS authentication.

We require the `adb_keys` file to be generated with a valid key in order to enable the "comparison check" phase of the mTLS handshake process. To do this, we simply need to connect to the target device via regular USB debugging i.e using ADB.

![empty-adb](/assets/images/adb-rce/2-empty-adb.avif)

We see here that the `adb_keys` file exists now but is empty. We need to trigger the target device to explicitly request for saving keys by turning on Wireless Debugging and initiating a successful connection.

![wireless-dbg](/assets/images/adb-rce/4-wireless-dbg.avif)

The IP and port details listed under the Wireless Debugging menu were noted down in order to configure the socat relay in the background for successful connection.

![remember](/assets/images/adb-rce/4-remember.avif)

Once the connection has been made and the client device is set to "remembered" in the target device, the `adb_keys` file should now be populated with a valid RSA key.

![populated-adb](/assets/images/adb-rce/5-adb-key-filled.avif)

![rsa-key](/assets/images/adb-rce/6-rsa-key.avif)


### ♦️ Execution
We utilize a pre-made script for automating the mTLS authentication process. Below is a link to a Proof-of-Concept built by SecTestAnnaQuinn.<br>

<mark>LINK:</mark> https://github.com/SecTestAnnaQuinn/CVE-2026-0073-Android-adbd-authentication-bypass-POC/tree/main<br>
_**Note:** Always test untrusted material in a sandbox or controlled environment such as a VM._

Ensure that the client device (Kali Linux VM) has been completely disconnected from the target device (Pixel 9) to prevent interference and connection errors.

Execute the script and specify the appropriate IP and port numbers. If using a relay or ADB's port forwarding function, ensure you specify the correct port number.

![success](/assets/images/adb-rce/7-success.avif)

The script acts immediately to initiate the mTLS handshake, bypass authentication by submitting a mismatched key for the certificate, and finally spawn a shell session.

___

## Security Impact - Why This Matters
### ♦️ BYOD Fleet Vulnerability
As this vulnerability is relatively recent following the date of publish for this project, a substantial portion of corporate mobile device fleets likely operate on Android version older than 17.

Because OS upgrades lag behind rapidly shifting threat landscapes, unpatched or delayed security baseline states leave fleets exposed to network compromise.

### ♦️ "Power Users" & Developer Risk Profile
The wireless debugging feature is often enabled by developers, QA testers, and even tech-savvy users/employees for reasons such as convenience, automation, or device modding.

When these users connect their personal or work-profile devices to unsecured corporate guest Wi-Fi, home networks, or public hotspots, they inadvertently create an open attack surface.

### ♦️ Silent Lateral Movement
Because the exploit bypasses mTLS authentication without any user interaction (Zero-Click), an attacker on the same local network segment can silently establish an interactive shell in seconds.

This would then allow them to pivot from one compromised mobile device to local network services, credential data stores, or even stage further internal network reconnaissance.

___

## Mitigations & Remediation
### ♦️ Apply Latest Security Updates
As outlined previously, this vulnerability has been addressed in the May 2026 Android Security Patch. Core system components (adbd) are also patched independently via Google Play system updates. Verify your system update timestamp under device details and ensure patch level is current.

### ♦️ Network Hygiene
Disable Wireless Debugging feature when not actively developing. This prevents exposing unused yet active ports on untrusted spaces like public Wi-Fi networks.

### ♦️ Strict Conditional Patching
For code reviewers and maintainers, ensure cryptographic comparison return values are explicitly evaluated against expected success states rather than relying on loose truthy evaluations. Example:
```c
// SNIPPET
if (EVP_PKEY_cmp(...) == 1) // Enforces exact match for validation
// SNIPPET
```
