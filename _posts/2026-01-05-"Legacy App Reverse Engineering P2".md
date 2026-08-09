---
title: "Puzzleball P2: Activation"
date: 2026-01-05 00:00:00 +/-TTTT
categories: [research, reverse engineering]
tags: [windows, reverse engineering, ghidra]     # TAG names should always be lowercase
image: "/assets/images/reverse_p2.png"
---
## Activation Mechanism
> GOAL: Decode the activation mechanism.

I began the analysis by first making simple observations of the application's features. While the initial goal was to decode the activation mechanism, I still had to establish whether that was actually feasible with Puzzleball 3D. Below are the findings:

## Initial Observations
### ♦️ First Impressions
Launching Puzzleball 3D from either double-clicking the main .EXE or quick start icon presents a stylized "launcher" menu. The revision number is interestingly shown here at the bottom<br>
➠ Rev. 3744 Build 177.<br>

This was likely used for internal troubleshooting or to even help with customer support and does not appear to be relevant to the goal, considering the fact that this is a closed-source application that likely did not have a public repository with other versions made available.<br>

![main-menu](/assets/images/re/part2/1.avif)<br>

The highlighted text, ``...no additional download required``, indicates that the game install is complete in where all the required assets, data, and features are already present on the system and that the trial mode applies a digital/software lock. This is promising because it means reverse-engineering is feasible.

### ♦️ Exploring The Menus
![menu-options](/assets/images/re/part2/2.avif)<br>

Clicking on the ``Other Games`` button leads to an insecure and likely expired URL. Since the publisher's site is no longer functional (non-existent backend), exploits like SSRF and IDOR are not possible. They would require explicit permission from the server owners anyway and is something outside the scope of this project. But it is interesting to imagine how "fishing" for unlock codes stored in a database could've been a possibility.<br>

![page-not-found](/assets/images/re/part2/3.avif)<br>

Clicking on ``Already Paid`` brings forth a new window into view with 4 separate tabs.
The text here indicates that an internet connection is required to activate the game or unlock the full version. We could intercept the application's attempt at "calling back to base" and host our own server to spoof the validation process but there is far too little information (protocol, request format, etc.) at this point to go down this route.<br>

![purchase-method](/assets/images/re/part2/4.avif)<br>

Clicking on the hyperlink, ``I'm not connected to the internet``, renders a new screen displaying a URL and interestingly, a product code. This code doesn't seem to change with multiple restarts of the application so it is likely tied to some system property or attribute.
We'll note this down for future reference.<br>

![activate-code](/assets/images/re/part2/5.avif)<br>

Going back to the dead URL, we see that the product code is also exposed as a value for the parameter, ``pid``, which likely stands for product ID. It is difficult to ascertain what ``did`` is, but we'll note down this URL for reference as well.

The other tabs are simply different payment methods for what used to be the way to obtain legitimate unlock/activation codes for Puzzleball 3D. They all require a response from the payment processor's server which is well outside the scope of this project.

## Static Analysis
After having established that reverse-engineering may be possible with Puzzleball 3D, I performed some basic static analysis using PE-bear to sift through the binary for string references and Ghidra to provide a graphical view of the functions.

### ♦️ PE-bear Findings
Often times, the protection mechanism for an application is found in a separate binary or executable like a DLL file. This file runs in tandem with the main application and acts as a "security guard".

This is why "cracks" back then came with 2 files; a **modified .EXE** executable and a **modified DLL** file, both of which you would have to copy over to the root directory of the relevant application in order to overwrite and neuter whatever protection mechanism was in place.

In our case, a proprietary DLL file does exist in the root directory of Puzzleball 3D, which we'll refer to as ``RA.dll`` for both the sake of simplicity and obfuscation. The following are strings I found that stuck out the most from each binary:

**Puzzleball 3D.exe**  
• ``radll_HasTheProductBeenPurchased`` @ offset 2aaa0  
• ``unittest_GetBrandedApplicationID`` @ offset 2ab7c  
• ``The DRM dll has been altered since it was generated and is not deemed to be secure`` @ offset 2ac70  
• ``radll_GetUnlockCode`` @ offset 2a514  

**RA.dll**  
• ``Decryption Key Data=A/ZTZDGDQ7YGUERNPMP3VVVT4XFWSBBL`` @ offset b8818  
• ``GlobalUnlock`` @ offset c6fde  
• ``radll_GetUnlockCode`` @ offset c7280  
• ``unittest_DecryptUnlockCode`` @ offset c73df  
• ➟ ``unittest_ValidateUnlockCode`` @ offset c7553  
• ``Action Fail Wrong Unlock Code`` @ offset c8660  
• ``Form Edit Control Containing Unlock Code`` @ offset c86f4  
• ``UnlockCode`` @ offset c8728  

Aside from the intriguing "Decryption Key Data" string in the DLL file, there were no hardcoded unlock codes found in either of the examined binaries. More importantly, the 3rd string listed under the main .EXE (@ offset 2ac70) essentially confirms that the copy protection or DRM (Digital Rights Management) is tied to the DLL file.

While all of these strings looked enticing to follow, for the purposes of this project, ``unittest_ValidateUnlockCode`` seemed like the one closest to our goal of decoding the activation mechanism. Looking this up with Ghidra's search tool led to a function with that exact name.<br>

### ♦️ Ghidra Findings
![ghidra-findings](/assets/images/re/part2/6.avif)<br>

The above is a "Ghidra representation" of the function,  ``unittest_ValidateUnlockCode``, translated from machine code into human-readable form. There are 3 variables being declared:
- cVar1, a single character data type.
- local_c and local_8, both integer data types.<br>

The routine seems to begin with some processing applied to the supplied arguments (*param_1 and *param_2) along with the aforementioned variables.

A new value for cVar1 is derived from the processing involving function ``FUN_100180d1`` before being compared against the value ``'\0'`` at the end which is likely a null byte.

To understand this routine better, I switched to the "Function Graph" view for ``unittest_ValidateUnlockCode`` in Ghidra to examine the "assembly logic"<br>
![unittest-validateunlock](/assets/images/re/part2/7.avif)<br>

Beginning from the top, we see that the ECX register's value is pushed twice onto the stack in place of where the ESP register's value would typically be subtracted to "make space" for local variables. This could just be a form of compiler optimization.

We also see that the function, ``FUN_1007ccbd``, is applied, if you will, in a consistent manner to both ``param_1`` and ``param_2``. The preceding and following instructions to the function call are very similar. Another function towards the end, ``FUN_1007cdca``, also seems to be applied in this way. It could be reasoned that the former is some sort of "preparation step" and the latter a "finishing step".

These two functions are likely not where the core logic or math for the activation mechanism is located but I still thought it important to delve deeper into ``FUN_1007ccbd`` in an attempt to fully deconstruct the routine.<br>

![function-1007ccbd](/assets/images/re/part2/8.avif)<br>

Here we see a nested set of blocks that divide the assembly into several sections. Attempting to decode this without any dynamic analysis or runtime context took an immense amount of time and effort.

Suffice to say, this function essentially allocates system memory to load the variables and arguments in an "expected state" before being processed by the other routines found in ``unittest_ValidateUnlockCode``, namely ``FUN_10018172`` and ``FUN_100180d1``.

The presence of standard C library functions like ``_strlen``, ``_strncpy``, as well as ``operator_new`` (found within sub-function ``FUN_1007ebe0`` in ``FUN_1007ccbd``) are the biggest giveaways.

![function-10018172](/assets/images/re/part2/9.avif)<br>

Backing out of ``FUN_1007ccbd`` and moving onto ``FUN_10018172``, we see a routine that is even more convoluted. Attempting to decode this with only information and context available through static analysis alone was near impossible, largely due to unknown items like ``DAT_100dc868`` and ``DAT_100dc86c`` as well as the numerous amount of nested functions.

All that I could ascertain from this summarized view of ``FUN_10018172`` is that if the argument ``param_2`` contains a character 'F', then further processing is applied. Otherwise, the app's workflow would skip to the end and proceed to the next routine in the parent function which would be ``FUN_100180d1``.

___

## Making A Decision
Reverse-engineering is a process that can get quickly overwhelming if the right tools and methodology aren't applied. But as we'll see later, even that may not be enough to fully deconstruct something like Puzzleball 3D which, while not seriously obfuscated, is a relatively ancient application with peculiar design philosophies that don't necessarily agree with modern programming standards.

After having spent countless hours trying to uncover the purpose of ``FUN_100180d1`` but ultimately ending with nothing substantial, I deemed this approach a "failure". This however, did not stop me from stepping back and taking an entirely different angle, as will be detailed in Part 2.
