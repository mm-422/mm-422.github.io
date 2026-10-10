#---
title: "Deconstructing Google PairIP DRM ft. FRIDA"
date: 2026-10-1 00:00:00
categories: [research, reverse engineering]
tags: [android, mobile security, reverse engineering]
image: "/assets/images/pairip-drm/android-frida.png"
mermaid: true
#---
``DOMAIN:`` Pentesting | Vulnerability Mgmt<br>
``CONTEXT:`` The skills, tools, and methodology covered here could be applicable in security assessments for dev teams and/or organizations to identify any security oversight in the SDLC process (Android Platform) and address them with appropriate remedies or mitigations.
___

## Executive Summary
This project documents a deep dive into Google's PairIP (Integrity Protection) technology; a modern, industry-standard security mechanism for Android apps that aims to provide integrity protection for mobile applications as well as enforcing Digital Rights Management (DRM).

The goal is to demonstrate how this security mechanism cannot be relied on solely as a silver bullet for application security. Implemented on its own or even poorly, PairIP can be insufficient for protecting mobile applications especially when put up against powerful current-day reverse engineering methods and debugging tools.

The following assessment along with its reported findings should clarify why a holistic approach to Android AppSec should never be replaced with any single solution. Google's PairIP and other similar DRM implementations cannot compensate for the absence of crucial processes such as Secure Software Development Lifecycle (SSDLC) that encompasses tasks like "Secure by Design" app architecture, code reviews, vulnerability assessments, and so on.   


### ♦️ Scope & Ethical Considerations
- This project focuses on understanding and identifying gaps in security mechanisms, in particular Google's PairIP Integrity Protection.
- The goal is to demonstrate how a single DRM solution can be insufficient for protecting application integrity on the Android platform.
- Obfuscation is applied where necessary to protect sensitive and proprietary data.
- This project _**DOES NOT**_ distribute material that could encourage piracy.
- _**NO**_ modified binaries or APKs are distributed.
- Any mention or demonstration of weaponization potential is done under an educational lens.


### ♦️ Technical Summary
Google's PairIP is primarily implemented into standard Android apps in two ways:
 - At the Java/Kotlin layer.
 - Via a native library, `libpairip.so`.

A "native implementation", while more robust, can incur penalties such as increased storage footprint (app must carry extra encrypted code) and present distribution challenges (difficulty in offering app on other legitimate storefronts).

Neither implementation should be relied on as a "one-size-fits-all" solution when it comes to security as they can both be broken and bypassed.

Identifying the specific implementation of PairIP can be done by looking for either references in the AndroidManifest or for the existence of a native library called `libpairip.so` in the /lib directory. 

This project explores the Java/Kotlin layer implementation of PairIP which is common in apps that require maximum performance for their core routines and where the authors wish to maximize compatibility across a wide range of devices.

While it is possible to apply a "rough bypass" by way of manual code patching, this was skipped in favor of a "live demonstration" utilizing tools like Frida to show where and how the PairIP implementation falls short with supporting evidence from terminal output and logs. 


### ♦️ Security Impact - Why This Matters
#### Exposure of Core/Business Logic
- Attacker that bypasses the PairIP verification gate would be free to read, analyze, and rip the internal code as well as business logic without affecting app functionality.
- In the case of a non-native implementation of PairIP and absence of encryption, an attacker could steal proprietary code/scripts.

#### Malicious Code Injection
- Threat actors are free to modify an application and repackage it.
- With active signature enforcement neutralized, they could inject malware.
- This could become part of an impersonation campaign where the threat actor could up a fake "official website" to target unsuspecting users and spread the malicious APK.
- This could lead to reputation and revenue loss.

#### API Exploitation
- With broken integrity protection, attacker could take advantage of an app that communicates with sensitive corporate back-ends.
- Attacker can siphon crypto keys, session tokens, enumerate API endpoints.
  
___

## Tools Used
```
Host
• Fedora Workstation 44 kernel 7.1.5

VM & Hypervisor
• Kali Linux 2026.2 via VirtualBox v7.2.14
• Waydroid v1.6.3 w/ Lineage OS

Static Analysis
• JADX GUI v1.5.6
• ADB 1.0.41

Dynamic Analysis
• Frida + Frida-Server v17.19.0

Misc
• apktool 3.0.3
• uber-apk-signer v1.3.0
• AntiSplit-M v2.3.2
```

___

## Assessment
### ♦️ The Initial Setup & Recon
As with the previous reverse engineering project (x86 Windows binary) I wanted to work on a "real-world" application rather than purpose-built "crackmes" or easy-to-replicate tutorials. I chose a rather new release on the Google Play store that was made available mere months before this write-up. This meant that the application did not utilize outdated frameworks that could be easily exploited.

We will call this app, `Balloons 3D V1.0`. Any reference to its real name or compromising images of its components will be adequately obfuscated to protect proprietary information.

To reiterate, the goal of this project is to demonstrate how PairIP alone can be insufficient to protect an application's integrity. There is no intent to harm, defame, or discredit any party. Findings are supported with recommended remediation/mitigation steps at the end.

#### Obtaining the Package
I managed to obtain an XAPK of the Balloons 3D app. An XAPK file can be thought of as a "zipped up" folder of multiple APKs and configuration files. These may be libraries specific to the platform (e.g. config.arm64_v8a.apk) as well as crucial expansion files containing large media files and assets called OBB files (Opaque Binary Blob).

Some apps, typically ones that utilize large assets and libraries, are packaged this way to get around the strict file size limits on the Google PlayStore.

We can view the contents of an XAPK by simply extracting the archive or renaming it to a ".zip" first if needed.

[IMG-XAPK-CONTENTS]

#### Installation
A Waydroid emulator with Lineage OS was used for testing. All tools and source files were placed in a separate Kali Linux VM to keep the environment clean. Unfortunately, Waydroid does not have a built-in installer for XAPK files. So we would have to either:
	- use a dedicated XAPK installer
	- install each file manually
	- merge the split APK files into one before installation.

I opted to merge the separate files with a tool called AntiSplitM. The resulting APK file was then installed on the Waydroid emulator via Android Debug Bridge (ADB).

[STREAMED-INSTALL]

NOTE: The Waydroid emulator is an x86-based application. It does not come with the necessary libraries to run a native ARM64 application like Balloons 3D. In order to do so, we would require a translation layer to convert the system calls performed by the app meant for an ARM processor (as those found in modern phones) to ones that are compatible with a desktop-class x86 processor. This can be done with community-made scripts such as the `waydroid_script`.

#### First Launch
With installation complete, we boot up Balloons 3D, only to be met with a short loading animation, followed by the dreaded "Google Play" error.

[GOOGLE PLAY ERROR]

This error is the direct output of the PairIP routine which detected that this specific copy of Balloons 3D was not obtained from the official Play Store. Our goal is to bypass this error message which would require bypassing the PairIP mechanism.


### ♦️ Static Analysis & Control Flow Mapping
#### Error Dialog Reference
```
Something went wrong.
Check that Google Play is enabled on your device and that you're using
an up-to-date version before opening the app. If the problem persists 
try reinstalling the app
```

We start by analyzing the AndroidManifest file with a tool called JADX GUI.
This is specifically the GUI version of JADX which breaks open an APK file to display its contents in a very organized view.

We were able to obtain the exact package name which will be important for when specifying flag values for tools like Frida. We will refer to this package name as `com.devstudio.balloons`.

We were also able to locate references to PairIP, listed as activities in the Manifest as well as the primary, "LAUNCHER" activity => `com.unity3d.player.UnityPlayerActivity`

This is the activity that is responsible for launching the Balloons application. However, it won't be executed until the PairIP verification process completes successfully. This is because PairIP intercepts the Manifest to inject its own activity and replace `com.unity3d.player.UnityPlayerActivity` so that it could run a "license check". It is only when this check completes successfully does PairIP create an Intent to hand over the execution chain back to the main application, from which Balloons 3D would start.

We could "patch out" the PairIP activity in the Manifest but that would not demonstrate how this mechanism could be bypassed and when. To do so, we will need to delve into the Dynamic Analysis section later on.

#### String Search
For now, it is worth locating parts of if not the exact string from the error dialog to see if we could "trace back" from the class that is reponsible for generating error messages to the one that is responsible for the "comparison check" that determines our copy of Balloons 3D as legitimate or otherwise.

With a quick string search via the JADX GUI Text Search tool, we find an exact match for the "Google Play" error that traces back to a package called `com.pairip.licensecheck.LicenseActivity'

#### An Error Dialog Method in com.pairip.licensecheck.LicenseActivity
```
    /* JADX INFO: Access modifiers changed from: private */
    public /* synthetic */ void lambda$showErrorDialog$0() {
        try {
            new AlertDialog.Builder(this).setTitle("Something went wrong").setMessage("Check that Google Play is enabled on your device and that you're using an up-to-date version before opening the app. If the problem persists try reinstalling the app.").setPositiveButton("Close", new DialogInterface.OnClickListener() { // from class: com.pairip.licensecheck.LicenseActivity$$ExternalSyntheticLambda2
                @Override // android.content.DialogInterface.OnClickListener
                public final void onClick(DialogInterface dialogInterface, int i) {
                    this.f$0.lambda$showErrorDialog$1(dialogInterface, i);
                }
            }).setCancelable(false).show();
        } catch (RuntimeException e) {
            Log.d(TAG, "Couldn't show the error dialog. " + Log.getStackTraceString(e));
        }
    }
```
Based on the code, it is reasonable to conclude that this method is responsible for constructing the error message that is shown to us in the event of a PairIP verification failure. It is however, not responsible for PairIP's "logic check". To find that, we need to determine which method or class initiated the call to construct the error message.

We could do this by tracing back further from this point or approaching it from the other way by observing where the main activity, `com.pairip.application.Application` leads.

#### com.pairip.application.Application
```
package com.pairip.application;

import android.content.Context;
import com.pairip.licensecheck.LicenseClient;

/* JADX INFO: loaded from: classes2.dex */
public class Application extends android.app.Application {
    @Override // android.content.ContextWrapper
    protected void attachBaseContext(Context context) {
        LicenseClient.checkLicense(context);
        super.attachBaseContext(context);
    }
}
```
We can clearly observe that this code initiates the first "routine" for PairIP which is LicenseClient located under `com.pairip.licensecheck.LicenseClient`. This is our next target of analysis. But before we go hunting for the "logic check" we must prove that the app execution chain flows through this path to confirm assumptions. This requires the use of a debugger and/or dynamic instrumentation tool which will be explored in the next section.

### ♦️ Generating Evidence
• While it may be trivial to patch the pairip implementation at the manifest level, the goal of this project is to demonstrate how it operates.
• We build a Frida script that prints out file path, file operations, and classes as it executes.
• Explain Scripts one by one.
	- File path
	- File command
	- Class Loader Scan Script
• Partial log output provided for sake of obfuscation.
• Correlate the data.

### ♦️ Surgical Exploitation Vector
• Analyzing methods in LicenseClient.
• We see plenty of references to an "Ordinal system".
• App has routines for evaluating different scenarios (paywall, trial, etc.)
• We find LicenseCheckState enumerator
	- CHECK_REQUIRED vs LOCAL_CHECK_OK
• Validating findings with custom Frida Script
	- Build a script to intercept the call to LicenseClient
	- Pass a certain value to ensure app doesn't crash
	- Verify that pairip verification has been successfully bypassed with script output comparison.

___

## Mitigation
• Apply R8/Proguard Flattening
	- Obfuscate predictable state enumerators (CHECK_REQUIRED, FULL_CHECK_OK)
• Upgrade PairIP implementation if possible
	- May not be desirable in certain scenarios.
• Server-side Attestation
	- Client must pass an encrypted Play Integrity token to company server to decrypt asset keys.
	- Offline play may not be possible.
