---
title: "Deconstructing Google PairIP ft. FRIDA"
date: 2026-10-1 00:00:00
categories: [research, reverse engineering]
tags: [android, mobile security, reverse engineering]
image: "/assets/images/pairip-drm/android-frida-2.png"
---
``DOMAIN:`` Pentesting | Vulnerability Mgmt<br>
``CONTEXT:`` The skills, tools, and methodology covered here could be applied in security assessments for dev teams and/or corporate environments to identify any security oversight in the Software Development Lifecycle (SDLC) process, specifically for the Android Platform, and address them with appropriate remedies or mitigations.

___

## Executive Summary
This project documents a deep dive into Google's PairIP (Integrity Protection) technology; a modern, industry-standard security mechanism for Android apps that aims to provide integrity protection for mobile applications as well as enforcing Digital Rights Management (DRM).

The goal is to demonstrate why this security mechanism cannot be relied on solely as a silver bullet for application security. Implemented on its own or even poorly, PairIP can be insufficient for protecting mobile applications especially when put up against powerful current-day reverse engineering methods and debugging tools.

The following assessment along with its reported findings should clarify why a holistic approach to Android AppSec should never be replaced with any single solution. Google's PairIP and other similar DRM implementations cannot compensate for the absence of crucial processes such as Secure Software Development Lifecycle (SSDLC) that encompasses tasks like "Secure by Design" app architecture, code reviews, vulnerability assessments, and so on.   


### ♦️ Scope & Ethical Considerations
- The goal is to demonstrate how a single DRM solution (PairIP) can be insufficient for protecting app integrity.
- The focus is to identify gaps in security mechanisms on Android, not encourage malicious actions.
- Obfuscation is applied where necessary to protect sensitive data.
- This project _**DOES NOT**_ distribute APKs or binaries that could encourage piracy.
- Any mention or demonstration of weaponization potential is done under an educational lens.


### ♦️ Technical Summary
Google's PairIP is primarily implemented into standard Android apps in two ways:
 - At the Java/Kotlin layer.
 - Via a native library, `libpairip.so`.

A "native implementation", while more robust, can incur penalties such as increased storage footprint (app must carry extra encrypted code) and present distribution challenges (difficulty in offering app on other legitimate storefronts).

Neither implementation should be relied on as a "one-size-fits-all" solution when it comes to security as they can both be broken and bypassed.

Identifying the specific implementation of PairIP can be done by looking for either references in the AndroidManifest file or for the existence of a native library called `libpairip.so` in the /lib directory. 

This project explores the Java/Kotlin layer implementation of PairIP which is common in apps that require maximum performance for their core routines and where the authors wish to maximize compatibility across a wide range of devices.

While it is possible to apply a "rough bypass" by way of manual code patching, this was skipped in favor of a "live demonstration" utilizing tools like Frida to **show where and how** the PairIP implementation falls short with supporting evidence from terminal output and logs. 


### ♦️ Security Impact - Why This Matters
#### Exposure of Core/Business Logic
- Attacker that bypasses the PairIP verification gate would be free to read, analyze, and rip the internal code as well as business logic without affecting app functionality.
- In the case of a non-native implementation of PairIP and absence of encryption, an attacker could steal proprietary code/scripts.

#### Malicious Code Injection
- With integrity protection neutralized, threat actors are free to inject malware into apps and repackage them.
- This could become part of an impersonation campaign.
- Threat actor would set up a fake "official website" to target unsuspecting users and spread the malicious APK, potentially leading to reputation and revenue loss.

#### API Exploitation
- Attacker could take advantage of apps that communicate with sensitive corporate back-ends once integrity protection has been bypassed.
- E.g. Siphon crypto keys, session tokens, enumerate API endpoints.
  
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

![setup-scrnshot](/assets/images/pairip-drm/1-setup.avif)
_Waydroid running on the left. Kali Linux on the right._

___

## Assessment
### ♦️ The Initial Setup & Recon
As with the previous reverse engineering project (x86 Windows binary) I wanted to work on a "real-world" application rather than purpose-built "crackmes" or easy-to-replicate tutorials. I chose a rather new release on the Google Play store that was made available mere months before this write-up. This meant that the application did not utilize outdated frameworks that could be easily exploited.

We will call this app, `Balloons 3D V1.0`. Any reference to its real name or compromising images of its components will be adequately obfuscated to protect proprietary information.

To reiterate, the goal of this project is to demonstrate how PairIP alone can be insufficient to protect an application's integrity. There is no intent to harm, defame, or discredit any party. Findings are supported with recommended remediation/mitigation steps at the end.

#### Obtaining the Package
I managed to obtain an XAPK of the Balloons 3D app. An XAPK file can be thought of as a "zipped up" folder of multiple APKs and configuration files. These may be libraries specific to the platform (e.g. config.arm64_v8a.apk) as well as crucial expansion files containing large media and assets called OBB files (Opaque Binary Blob).

Some apps, typically ones that utilize large assets and libraries, are packaged this way to get around the strict file size limits on the Google PlayStore.

We can view the contents of an XAPK by simply extracting the archive or renaming it to a ".zip" first if needed.

![xapk-content](/assets/images/pairip-drm/2-apk-contents.avif)

#### Installation
A Waydroid emulator with Lineage OS was used for testing. All tools and source files were placed in a separate Kali Linux VM to keep the environment clean. Unfortunately, Waydroid does not have a built-in installer for XAPK files. So we would have to either:
	- Use a dedicated XAPK installer.
	- Install each file manually.
	- Merge the split APK files into one before installation.

I opted to merge the separate files with a tool called AntiSplit-M. The resulting APK file was then installed on the Waydroid emulator via Android Debug Bridge (ADB).

![install](/assets/images/pairip-drm/3-streamed-install.avif)

**Note:** The Waydroid emulator is x86 based. It does not come with the necessary libraries to run a native ARM64 application like `Balloons 3D`. In order to do so, we would require a translation layer to convert the system calls performed by the app meant for an ARM processor (as those found in modern phones) to ones that are compatible with a desktop-class x86 processor. This can be done with community-made scripts such as the `waydroid_script`.

#### First Launch
With installation complete, we boot up `Balloons 3D`, only to be met with a short loading animation, followed by the dreaded "Google Play" error.

![setup-scrnshot](/assets/images/pairip-drm/4-gplay-error.avif)

This error is the direct output of the PairIP routine which detected that this specific copy of `Balloons 3D` was not obtained from the official Google Play app store. Our goal is to bypass this error message which would require bypassing the PairIP mechanism entirely.


### ♦️ Static Analysis & Control Flow Mapping
**Error Dialog Reference**
```
Something went wrong.
Check that Google Play is enabled on your device and that you're using
an up-to-date version before opening the app. If the problem persists 
try reinstalling the app
```

We start by analyzing the AndroidManifest file with a tool called JADX GUI.
This is specifically the GUI version of JADX which breaks open an APK file to display its classes, methods, and more in a very organized view.

We were able to obtain the exact package name which will be important when specifying flag values for tools like Frida. We will refer to this package name as `com.devstudio.balloons`.

We were also able to locate references to PairIP, listed as activities in the Manifest as well as the primary, "LAUNCHER" activity => `com.unity3d.player.UnityPlayerActivity`

This is the activity that is responsible for launching the Balloons application. However, it won't be executed until the PairIP verification process completes successfully. This is because PairIP intercepts the Manifest to inject its own activity and replace `com.unity3d.player.UnityPlayerActivity` so that it could run a "license check". It is only when this check completes successfully does PairIP create an Intent to hand over the execution chain back to the main application, from which `Balloons 3D` would start.

We could "patch out" the PairIP activity in the Manifest but that would not demonstrate where exactly in the execution chain the implementation falls short. To do so, we will need to delve into dynamic analysis with some logging.

#### String Search
For now, it is worth locating parts of, if not the exact string from the error dialog to see if we could "trace back" from the class that is responsible for generating error messages to the one that is responsible for the "comparison check" that determines our copy of `Balloons 3D` as legitimate or otherwise.

With a quick string search via the JADX GUI "Text Search" tool, we find an exact match for the "Google Play" error that traces back to a package called `com.pairip.licensecheck.LicenseActivity'

**Error Dialog Constructor Method**
```java
// SNIPPET
    public /* synthetic */ void lambda$showErrorDialog$0() {
        try {
            new AlertDialog.Builder(this).setTitle("Something went wrong").setMessage
            ("Check that Google Play is enabled on your device and that you're using an up-to-date version 
            before opening the app. If the problem persists try reinstalling the app.")
            .setPositiveButton("Close", new DialogInterface.OnClickListener() { // from class: com.pairip.licensecheck.LicenseActivity$$ExternalSyntheticLambda2
                @Override // android.content.DialogInterface.OnClickListener
                public final void onClick(DialogInterface dialogInterface, int i) {
                    this.f$0.lambda$showErrorDialog$1(dialogInterface, i);
                }
            }).setCancelable(false).show();
        } catch (RuntimeException e) {
            Log.d(TAG, "Couldn't show the error dialog. " + Log.getStackTraceString(e));
        }
    }
// SNIPPET
```
We can ascertain from the code snippet above that this method is responsible for constructing the error message shown to us in the event of a PairIP verification failure. It is however, not responsible for PairIP's "logic check". To find that, we need to determine which method or class initiated the call to construct the error message.

We could do this by tracing back further from this point or approaching it from the "other end" by observing where the main activity, `com.pairip.application.Application` leads.

**com.pairip.application.Application**
```java
// SNIPPET
public class Application extends android.app.Application {
    @Override // android.content.ContextWrapper
    protected void attachBaseContext(Context context) {
        LicenseClient.checkLicense(context);
        super.attachBaseContext(context);
// SNIPPET
```
We can observe that this code initiates the first "routine" for PairIP which is LicenseClient, located under `com.pairip.licensecheck.LicenseClient`. This is our next target of analysis. But before we go hunting for the "logic check" we must prove that the app execution chain flows through this path to confirm assumptions. This requires the use of a debugger and/or dynamic instrumentation tool which will be explored in the next section.

### ♦️ Generating Evidence
In order to gather visible evidence of the application's execution route, we need to log its actions and processes.
This could be the file paths accessed, classes called, operations done, etc. by the app as it boots up and runs through its routines.

We can "hook into" the system calls performed by the app for the aforementioned functions by injecting scripts with a tool called Frida. Community-made and tested scripts are available on platforms like `GitHub` and `CodeShare`.

For this project, a "file access logger" script by vycioha was utilized.<br>
<mark>LINK:</mark> https://github.com/vycioha/FridaScripts

Two additional scripts for tracking file path access and class loader activity were also built to provide greater clarity of the app's behavior. Below are snippets of the core mechanism of these scripts:

**FILE PATH LOGGER**
```java
// SNIPPET
    /* --- FILE SPY FUNCTION ---
     * We hook file constructor to see every path accessed by app
     */
    var File = Java.use("java.io.File");
    File.$init.overload('java.lang.String').implementation = function (path) {
        console.log("[FILE PATH] App is accessing: " + path);
        return this.$init(path);
    };
// SNIPPET
```

**CLASS LOADER LOGGER**
```java
// SNIPPET
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
// SNIPPET
```
The most impactful of these three scripts is the "Class Loader Logger". This script prints the exact name of classes loaded by the app as it executes. We simply need to observe if the app calls a class in `com.pairip.licensecheck.LicenseClient` or at least `com.pairip.licensecheck.LicenseActivity` to confirm the route.

![class-cmd](/assets/images/pairip-drm/5-classloader-cmd.avif)

![class-output](/assets/images/pairip-drm/5-classloader-output.avif)

We see `LicenseActivity` is called shortly before the app displays the "Google Play" error message. However, we do not spot any instance of `LicenseClient` in the terminal output. This might be due to an asynchronous process of some sort where Frida does not have visibility on processes that run outside the target, `com.devstudio.balloons` in this case.

Let's try hooking directly into the `LicenseClient` class with Frida:
```java
// SNIPPET 
  var LicenseClient = Java.use("com.pairip.licensecheck.LicenseClient");
	
	// Attempt to neutralize checkLicense
	LicenseClient.checkLicense.implementation = function (context) {
		console.log("[*] Hooked LicenseClient!");
		console.log("[*] Neutralizing license check...");
		return false;
	};
// SNIPPET

```
![license-client-hook](/assets/images/pairip-drm/6-license-client.avif)

With this script, we can observe that the `LicenseClient` class is indeed called by the app, where intercepting it and returning either an incorrect or unexpected value (true or false), causes the app to enter a loading screen animation indefinitely. We could presume that it is "waiting" for an appropriate response or result from `LicenseClient`, where we've provided a "false" in this case.

We will now analyze the `LicenseClient` class in depth in the next section.

### ♦️ Surgical Exploitation Vector
The `LicenseClient` class contains numerous methods like `startPaywallActivity` and `handleTrialEnd` that are designed to address various license verification scenarios.

We need only identify the specific method or routine that is tasked with evaluating application integrity. At the very top of `com.pairip.licensecheck.LicenseClient`, we spot an internal enumeration class called `LicenseCheckState`:

![license-client-top](/assets/images/pairip-drm/7-license-client-top.avif)

```java
public enum LicenseCheckState {
    CHECK_REQUIRED,         // Ordinal 0
    FULL_CHECK_OK,          // Ordinal 1
    LOCAL_CHECK_OK,         // Ordinal 2
    LOCAL_CHECK_REPORTED,   // Ordinal 3
    REPEATED_CHECK_REQUIRED // Ordinal 4
}
```
This internal enumeration class is likely part of a "state machine".
Under the context of Java programming, a state machine is a model for managing an app and its components' states using values called "Ordinals". The `ordinal()` method is a built-in Java class that is used to query the position of an enumeration constant.

The application relies heavily on checking the Ordinal values of this state machine inside `initializeLicenseCheck()`. When the app boots normally, it evaluates to `CHECK_REQUIRED` (Ordinal 0) and triggers the asynchronous "Google Play connection" loop to validate structural parameters.

If the signature check fails, the application calls an embedded error routine, triggers `LicenseActivity` to construct a "Google Play" error dialog (the one seen previously), and calls a final hardcoded termination hook => `LicenseClient.exitAction` running `System.exit(0)`.

```java
// SNIPPET
    protected static Runnable exitAction = new Runnable() { // from class: com.pairip.licensecheck.LicenseClient.1
        @Override // java.lang.Runnable
        public void run() {
            System.exit(0);
        }
    };
// SNIPPET
```

If we could intercept the state machine and modify the ordinal values as they are requested, we could bypass the "license check" routine and thus, overcome the PairIP mechanism without any destructive modifications.

To build an effective Frida script, we need to take into account specific bottlenecks:
**The State Machine Neutralizer**
```java
// SNIPPET
    try {
        var LicenseClient = Java.use("com.pairip.licensecheck.LicenseClient");
        var LicenseCheckState = Java.use("com.pairip.licensecheck.LicenseClient$LicenseCheckState");
        var Runnable = Java.use("java.lang.Runnable");

        // Rewrite the exit action so it can never kill the app process
        var DummyRunnable = Java.registerClass({
            name: 'com.pairip.bypass.DummyRunnable',
            implements: [Runnable],
            methods: {
                run: function () {
                    console.log("[+] System attempted to call exitAction(). Blocked safely!");
                }
            }
        });
        
        LicenseClient.exitAction.value = DummyRunnable.$new();
        console.log("[+] exitAction successfully defused.");


        // Safely target the ordinal method ONLY for the target Enum class
        LicenseCheckState.ordinal.implementation = function () {
            var realOrdinal = this.ordinal();
            var stateStr = this.toString();
            
            // Validate that we are only intercepting the specific PairIP state machine
            if (stateStr === "CHECK_REQUIRED" && realOrdinal === 0) {
		console.log("STATE: " + stateStr);
                console.log("[========= BYPASS TRIGGERED =========]");
                console.log("[+] Selectively swapping CHECK_REQUIRED -> LOCAL_CHECK_OK (2)");
                return 2; 
            }
// SNIPPET
```

**Neutralizing the Exit Kill-Switch**
Rather than battling the UI dialog lifecycle, the `exitAction` variable—which holds a standard Java `Runnable` class—was systematically defused. A custom `DummyRunnable` class was generated via Frida and dynamically assigned to `LicenseClient.exitAction.value`. If any secondary validation checks or asynchronous failures triggered an exit call, the execution hit a dead-end method, keeping the host application process completely alive.


**Surgical Enum Ordinal Tampering**
Standard global enum interception is highly unstable, as hooking `java.lang.Enum.ordinal()` impacts unrelated system enums (such as font rendering, localization, and graphic attributes), causing immediate stability crashes. To circumvent this, the hook was bound directly to the inner class `LicenseClient$LicenseCheckState` and restricted with strict validation logic:

```javascript
if (stateStr === "CHECK_REQUIRED" && realOrdinal === 0) { ... }
```

By ensuring that the interceptor **only targeted the exact state machine** when its state was actively `CHECK_REQUIRED` with a default original Ordinal value of 0, all other application enums were left unharmed. 

"Fuzzing" the replacement ordinal values (0-5) yielded the following results:
```
• Returning `1` [FULL_CHECK_OK]
Forced the app to try to read cryptographic payload structures
that were not present, triggering a native null pointer crash.

• Returning `2` [LOCAL_CHECK_OK]
Successfully convinced the application engine that the security
requirements had been met locally. The verification routine was bypassed,
the fatal error dialog was suppressed, and control was safely handed down
to the underlying Unity Engine runtime engine [com.unity3d.player.UnityPlayerActivity],
establishing a complete DRM bypass.
```
![ordinal-result](/assets/images/pairip-drm/8-ordinal-result.avif)

We can see that the `LicenseActivity` class is completely absent from the terminal output now. Accordingly, the "Google Play" error no longer triggers, proving the success of the PairIP bypass via "state machine manipulation".

In its place, we see several new calls to classes like `sun.security.provider.certpath` that correlate with an "application update" process. This may either be triggered by an internal "auto-update" logic or the app believing itself to be "incomplete" as a result of us having merged the APK prior to installation, possibly misaligning the internal file paths/pointers.

These elements however are outside the scope of this project and need not be addressed.

---

## Mitigations
### ♦️ Utilize R8/ProGuard for Robust Obfuscation
We were able to identify the critical internal state machine due to explicit enum class strings like `CHECK_REQUIRED` and `LOCAL_CHECK_OK`. An attacker could easily locate these strings via decompilers like JADX and manipulate the internal mechanism.

Utilize Google R8/ProGuard flatten control flow structures, randomize representation values for variables, and aggressively obfuscate internal state definitions.

- `PROS`: Significantly increase difficulty of reverse engineering and static analysis with minimal overhead on runtime performance.
- `CONS`: Does not prevent dynamic analysis, memory tracing or hooking via tools like Frida.


### ♦️ Upgrading PairIP to Native Layer
A hardened, native C/C++ implementation of the PairIP mechanism presents a far greater hurdle for reverse engineers to overcome. They would need to evade internal checks in order for debugging attachments and Frida server processes to run unimpeded.

- `PROS`: Dramatically increases complexity for automated tools.
- `CONS`: Higher development complexity that can complicate cross-platform compatibility. Can also lead to performance penalties if the internal encryption/decryption routines are aggressive or set to execute continuously during app operation. This may be undesirable for demanding applications.


### ♦️ Server-Side Attestation
Client-side DRM solutions are ultimately doomed especially when they trust the app's internal reporting and decision-making explicitly, as was the case with the state machine contained within `com.pairip.licensecheck.LicenseClient`.

Organizations should enforce a server-side attestation model where the mobile app would request a unique cryptographic token directly from the Google Play Integrity API. This raw token is passed to the organization's back-end infrastructure for server-to-server verification.

Crucial game resources, scripts, or asset configuration packs would remain heavily encrypted inside the application storage package, and the decryption keys are *only* delivered to the application instance after the back-end verifies that the token signature is legitimate, official, and unmodified.

- `PROS`: Bypassing the local client logic achieves nothing because the app will lack the cryptographic keys required to function.
- `CONS`: Requires a permanent, active internet connection for initialization. Offline functionality becomes impossible. Additionally introduces server maintenance costs.
