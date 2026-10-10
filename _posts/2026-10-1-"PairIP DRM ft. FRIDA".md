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
In order to gather visible evidence of the application's execution route, we need to log its actions. This could be the file paths accessed, classes called, operations done, etc.

We can launch the app and attach scripts to it to perform such functions using Frida.
Here are three scripts used for this project that each perform a specific function.

FILE PATH LOGGER
```java
// Script to spy on File Paths & Shell Commands

Java.perform(function () {
    console.log("[+] Spy Script Loaded...");

    /* --- FILE SPY FUNCTION ---
     * We hook File constructor to see every path the app "looks" at
     */
    var File = Java.use("java.io.File");
    File.$init.overload('java.lang.String').implementation = function (path) {
        console.log("[FILE PATH] App is accessing: " + path);
        return this.$init(path);
    };

    /* --- COMMAND SPY FUNCTION ---
     * We hook Runtime.exec to see any system commands the app tries
     */
    var Runtime = Java.use("java.lang.Runtime");
    Runtime.exec.overload('java.lang.String').implementation = function (cmd) {
        console.log("[SHELL COMMAND] App is executing: " + cmd);
        return this.exec(cmd);
    };

	// Also hook the String[] version of exec (common in many apps)
    Runtime.exec.overload('[Ljava.lang.String;').implementation = function (cmds) {
    var cmdString = cmds.join(" ");
    console.log("[SHELL COMMAND] App is executing: " + cmdString);
    return this.exec(cmds);
    };
});
```

FILE ACCESS LOGGER
```java
Java.perform(function () {
    var File = Java.use("java.io.File");

    function printDivider() {
        console.log("-------------------------------------------------------------");
        
    }

    function getColorText(text, colorCode) {
        return "\x1b[" + colorCode + "m" + text + "\x1b[0m";
    }
    
    console.log("File access logger successfully attached.");

    // Hook file read operations
    var FileInputStream = Java.use('java.io.FileInputStream');
    FileInputStream.$init.overload('java.io.File').implementation = function (file) {
        try {
            this.$init(file);
            console.log(getColorText("SUCCESS     - READ        - [*] file: " + file.getAbsolutePath(), "34")); // Blue color
        } catch (e) {
            console.log(getColorText("FAIL        - READ        - [*] file: " + file.getAbsolutePath(), "31")); // Red color
        }
    };

    // Hook file write operations
    var FileOutputStream = Java.use('java.io.FileOutputStream');
    FileOutputStream.$init.overload('java.io.File').implementation = function (file) {
        try {
            this.$init(file);
            console.log(getColorText("SUCCESS     - WRITE       - [*] file: " + file.getAbsolutePath(), "32")); // Green color
        } catch (e) {
            console.log(getColorText("FAIL        - WRITE       - [*] file: " + file.getAbsolutePath(), "31")); // Red color
        }
    };

    // Hook file delete operations
    File.delete.implementation = function () {
        var result = this.delete.call(this);
        if (result) {
            console.log(getColorText("SUCCESS     - DELETE      - [*] file: " + this.getAbsolutePath(), "32")); // Green color
        } else {
            console.log(getColorText("FAIL        - DELETE      - [*] file: " + this.getAbsolutePath(), "31")); // Red color
        }
        return result;
    };
});
```

CLASS LOADER LOGGER
```
Java.perform(function () {
	
	/* We build a wrapper for java.lang.ClassLoader
	 * This is a class found in Android apps, responsible for loading classes
	 * We then implement subclasses to draw information
	 */
	
	var ClassLoader = Java.use("java.lang.ClassLoader");
	console.log("Script Loaded. Les go!");
	
	// Hooking most common overload
	ClassLoader.loadClass.overload("java.lang.String").implementation = function (className) {
		console.log("[*] ClassLoader.loadClass called for: " + className);
		
		/* We need to call the original method again to prevent app from breaking
		 * We return the "call" at the end
		 * This ensures app continues execution
		 */
		var result = this.loadClass(className);
	
		/* We can set a target class to monitor
		 * Check to see if it loads
		 * Modify the variable
		 */
		
		if (className === "com.pairip.licensecheck.LicenseActivity") {
			console.log("[!] TARGET LOADED. TIME TO HOOK!");
		};
		
		return result;
	};
	
	// Hook ALT overload: loadClass(String name, boolean resolve)
	ClassLoader.loadClass.overload("java.lang.String" , "boolean").implementation = function(className, resolve) {
		console.log("[*] ClassLoader.loadClass (resolve) called for: " + className);
		
		// Call original class to continue execution
		return this.loadClass(className, resolve);
	};
});
```

The most impactful of these three scripts it the Class Loader Logger. This script prints the exact name of classes loaded by the app as it executes. We simply need to confirm if the app calls `com.pairip.licensecheck.LicenseClient` or at least `com.pairip.licensecheck.LicenseActivity` to confirm the route.

[COMMAND-SCRNSHOT]
[COMMAND-OUTPUT]

We see LicenseActivity is called shortly before the app displays the "Google Play" error message. However, we do not spot any instance of LicenseClient in the terminal output. This might be due to an asynchronous process of some sort.

Let's try hooking directly into the LicenseClient class with the following Frida script:
```java
Java.perform(function () {
	console.log("[*] Script loaded");
	
	// Use class loader tracing if Java.use not available
	try {
		var LicenseClient = Java.use("com.pairip.licensecheck.LicenseClient");
	
		// Attempt to neutralize checkLicense
		LicenseClient.checkLicense.implementation = function (context) {
			console.log("[*] Hooked LicenseClient!");
			console.log("[*] Neutralizing license check...");
			return false;
		};
	} catch (err) {
		console.log("[!] Class not available, switching to dynamic class loader hook...");
		
		// Alt Method: Catch class the second the classloader registers it
		// Java.use("block_if_needed_but_use_class_factory");
		var ClassLoader = Java.use("java.lang.ClassLoader");
		ClassLoader.loadClass.overload('java.lang.String').implementation = function (className) {
			var result = this.loadClass(className);
			if (className == "com.pairip.licensecheck.LicenseClient") {
				console.log("[+] Caught LicenseClient loading!");
				var TargetClass = Java.use("com.pairip.licensecheck.LicenseClient");
				TargetClass.checkLicense.implementation = function (ctx) {
					console.log("[+] Neutralized checkLicense via dynamic hook");
					return;
				};
			}
			return result;
		};
	}
}); 

```
[COMMAND OUTPUT]

With this script, we can observe that the LicenseClient class is indeed called by the app, where intercepting it and returning either an incorrect or unexpected value, causes the app to stay on a loading screen animation indefinitely. We could ascertain that it is "waiting" for an appropriate response or result from LicenseClient.

We will now analyze the LicenseClient class in depth in the next section.

### ♦️ Surgical Exploitation Vector
With the core initialization workflow mapped out, the focus shifted to identifying the precise evaluation gate determining application integrity. Static analysis of `LicenseClient.java` revealed a central state machine dependent on an internal enumeration class: `com.pairip.licensecheck.LicenseClient$LicenseCheckState`. 

```java
public enum LicenseCheckState {
    CHECK_REQUIRED,         // Ordinal 0
    FULL_CHECK_OK,          // Ordinal 1
    LOCAL_CHECK_OK,         // Ordinal 2
    LOCAL_CHECK_REPORTED,   // Ordinal 3
    REPEATED_CHECK_REQUIRED // Ordinal 4
}
```

The application relies heavily on checking the `.ordinal()` value of this state machine inside `initializeLicenseCheck()`. When the app boots normally, it evaluates to `CHECK_REQUIRED` (0) and triggers the asynchronous Google Play connection loop to validate structural parameters. If the signature check fails, the application calls an embedded error routine, triggers `LicenseActivity` to paint a "Google Play Store Error" UI dialog, and calls a hardcoded termination hook: `LicenseClient.exitAction` running `System.exit(0)`.

To cleanly bypass this gate without breaking the runtime context, a multi-layered Frida hook was deployed to target three specific bottlenecks:

1. **Neutralizing the Exit Kill-Switch:** Rather than battling the UI dialog lifecycle, the `exitAction` variable—which holds a standard Java `Runnable` class—was systematically defused. A custom `DummyRunnable` class was generated via Frida and dynamically assigned to `LicenseClient.exitAction.value`. If any secondary validation checks or asynchronous failures triggered an exit call, the execution hit a dead-end method, keeping the host application process completely alive.
2. **Forcing Natural Initialization:** The logic requires the application to branch properly through its core initialization pipeline. The helper method `isIsolatedProcess()` was pinned to explicitly return `false`. This forced the runtime away from dead-end sandbox code paths and allowed standard execution context mapping.
3. **Surgical Enum Ordinal Tampering:** Standard global enum interception is highly unstable, as hooking `java.lang.Enum.ordinal()` impacts unrelated system enums (such as font rendering, localization, and graphic attributes), causing immediate stability crashes. To circumvent this, the hook was bound directly to the inner class `LicenseClient$LicenseCheckState` and restricted with strict validation logic:

```javascript
if (stateStr === "CHECK_REQUIRED" && realOrdinal === 0) { ... }
```

By ensuring that the interceptor *only* targeted the exact state machine when its state was actively `CHECK_REQUIRED`, all other application enums were left unharmed. 

Fuzzing the replacement ordinal values (0-5) yielded explicit results. Returning `1` (`FULL_CHECK_OK`) forced the app to try to read cryptographic payload structures that were not present, triggering a native null pointer crash. However, selectively returning **`2` (`LOCAL_CHECK_OK`)** successfully convinced the application engine that the security requirements had been met locally. The verification routine was bypassed, the fatal error dialog was suppressed, and control was safely handed down to the underlying Unity Engine runtime engine (`com.unity3d.player.UnityPlayerActivity`), establishing a complete DRM bypass.

---

## Mitigation
To safeguard mobile applications against runtime dynamic instrumentation and signature-spoofing attacks, organizations should adopt a defense-in-depth approach rather than relying on standard client-side validation logic.

### 1. Robust Code Obfuscation & R8/ProGuard Control Flow Flattening
* **Mechanism:** Simple name obfuscation leaves package structures and method logic intact. Threat actors can easily discover state machines if enums preserve strings like `CHECK_REQUIRED` or `LOCAL_CHECK_OK`. Advanced optimization configurations must be injected into the ProGuard/R8 compilation pipeline to flatten control flow structures, randomize integer representation values for variables, and aggressively obfuscate or inline internal state definitions.
* **Pros:** Marginally zero overhead on runtime performance; significantly increases the cognitive load for reverse engineers attempting static code mapping.
* **Cons:** Does not explicitly prevent dynamic memory tracing or hooking via frameworks like Frida if an attacker successfully identifies entry-point wrappers.

### 2. Upgrading Anti-Tamper Suites to Native Layer Enforcement
* **Mechanism:** The application should transition its integrity protection suites away from pure Java bytecode implementations into hardened native C/C++ compiled binaries (`.so` libraries). Native enforcement engines can execute low-level operating system hooks to monitor runtime integrity, detect debugging attachments via `ptrace(PTRACE_TRACEME)`, check for signature modifications natively, and actively scan process memory maps (`/proc/self/maps`) to isolate and sever Frida server interaction ports.
* **Pros:** Dramatically increases complexity for automated modification tools; strips away standard Java-level hooking vectors.
* **Cons:** Native protection layers present higher development complexity, complicate cross-platform compilation architecture (ARM vs. x86), and can lead to structural overhead if executed continuously during execution.

### 3. Server-Side Attestation & Asset Encryption Decoupling
* **Mechanism:** The fundamental flaw in client-side DRM is trusting local application decisions. Organizations should enforce a server-side attestation model. The mobile app must request a unique cryptographic attestation token directly from the Google Play Integrity API. This raw token is passed to the organization's backend infrastructure for server-to-server verification. Crucial game resources, scripts, or asset configuration packs remain heavily encrypted inside the application storage package, and the decryption keys are *only* delivered to the application instance after the backend verifies that the token signature is legitimate, official, and unmodified.
* **Pros:** The definitive industry standard for preventing application duplication, fraud, and asset piracy. Bypassing the local client logic achieves nothing because the app will lack the cryptographic keys required to function.
* **Cons:** Requires a permanent, active internet connection for initialization; offline functionality becomes impossible. Additionally introduces server maintenance costs and scales infrastructure dependency requirements.
