---
title: "Legacy App Reverse Engineering P1 - Intro & Mechanisms"
date: 2026-01-05 00:00:00 +/-TTTT
categories: [research, reverse engineering]
tags: [windows, reverse engineering, ghidra]     # TAG names should always be lowercase
image: "/assets/images/reverse_p1.png"
---

# Executive Summary
> This repository details a reverse engineering project featuring a legacy software application from the early 2000s.

## Overview 
This project documents a thorough analysis of an early-2000s era application and its components by way of Reverse Engineering. Aspects such as the authentication mechanism, file integrity checks, UI construction and more are covered.

The goal is to demonstrate the importance of understanding an application's intended design and behavior **before** any application of technical steps and methodologies.

The legacy application in question is a video game that was primarily distributed on Windows circa early to mid 2000s. This game implemented custom routines and file validation mechanisms in order to fight off tampering.

While the original servers and publisher are no longer operational, for the sake of ethics, the exact name of this application and images of its components will be obfuscated as necessary.
Hence, the video game app will be referred to as "Puzzleball 3D" throughout the case study.<br>

## Project Goals
- Analyze the structure and behavior of legacy authentication mechanisms.
- Reverse engineer authentication logic with no source code or documentation.
- Demonstrate the importance of thorough research to provide context prior to forming hypotheses and  testing.<br>

## Overall Plan
- Start with basic reconnaissance to gather preliminary info for later analysis.
- Apply static analysis methods to identify functions and algorithm patterns.
- Move to dynamic analysis to observe application behavior.
- Determine potential vulnerabities and attack vectors.
- Document.

## Scope & Ethical Considerations
- This project focuses on understanding mechanisms and applying methodology.
- This project _**DOES NOT**_ distribute material that would encourage piracy.
- No KeyGen or hacktool creation is demonstrated.
- Any example of weaponization potential is done under an educational lens.
- While the application for this project can be found on sites like the Internet Archive, no form of source code or vendor documentation is available.<br>

## Technical Summary
- Observed application behavior with regard to user input.
- Performed static & dynamic analysis on both the main executable and an auxiliary DLL.
- Investigated file integrity checks within application's binary.
- Identified validation logic and performed bypass.
- Evaluated security weaknesses and weaponization potential.<br>

## Tools Used
- _**Disassemblers**_
	- ``Ghidra``
	- ``x64dbg``
- _**Primary Debugger**_
	- ``WinDbg``
- _**Process Inspection**_
	- ``Procmon``
	- ``Spy++``
- _**Auxiliary Tools**_
    - ``HxD`` for editing hex.
    - ``PE-bear`` for quick string searches.
    - ``Detect-It-Easy`` for app property inspection.
    - ``GIMP``, ``Notepad``, and ``LibreOffice`` for mindmaps and notes.<br>
- All testing was performed in a controlled ``Windows 10 22H2`` VM.<br>

## Takeaways
- Focus should be on understanding the intended design behind an application.
- Most if not all client-side validation is "doomed" with modern analysis and debugging tools.
- Clear documentation helps tremendously with prolonged debugging sessions.
- Design oversights and security mistakes seen in a 20-year old app can still be seen today.<br>

# The Environment
A testing environment including all the necessary tools and appropriate settings is paramount to reverse engineering, especially when analyzing (potentially) malicious software.

I prefer using a virtual machine for these tasks as they are easy to spool up with the exact configurations required. The "running state" can also be saved with a click of a button; **invaluable for long debugging sessions.**

Below are the environment details along with the specific settings applied.

## The Virtual Machine
- Set up Windows 10 22H2 64-bit through VirtualBox version 7.2.4.r170995.
- Puzzleball 3D was likely designed for Windows XP 32-bit.
- However, replicating the exact OS env is not necessary for the goals of this project.
<img width="1280" height="720" alt="VM DESKTOP" src="https://github.com/user-attachments/assets/78035289-022b-4aff-8589-c7b2672860eb" />

## Resources
- Allocated 4 logical cores (Zen 3 CPU) and 8GB of memory to Virtual Machine.
- This was sufficient for up to 3 instances of Ghidra + 1 instance of WinDbg simultaneously.

## Files & File Paths
- Created a backup of Puzzleball 3D's files on external media.
- Installed Puzzleball 3D in default directory under "Program Files (x86)".
- Duplicated Puzzleball 3D's root directory and files to Desktop.
- All testing and modifications to the app's binaries was performed on the files here.

## OS-level Settings
- Disabled network connection.
- Added the "Desktop" directory to Windows Defender's exception list.
- Left ASLR and default memory integrity settings like Core Isolation alone.

# The Tools
## Summary
Initially, I relied on just x64dbg and PE-bear to do the heavy lifting. But this was only sufficient for some static analysis. Eventually, I moved on to Ghidra + WinDbg as my potent duo, while also discovering plenty of incredible tools like Spy++ and Detect-It-Easy along the way.

I would also be remiss to not mention the incredible use I've found in leveraging LLMs like ChatGPT and Gemini in order to perform tasks such as research for application design philosophies relevant to the early 2000s era as well as sifting through hundreds of lines of assembly code.

Care should be taken when utilizing these technologies, especially when sensitive info is concerned. Considering the fact that Puzzleball 3D is openly available on sites like the Internet Archive with no relevant owners to contact and that hosting a local LLM would present its own set of challenges (cost, speed, reliability), the use of online hosted LLMs is justified in my opinion.

## Static Analysis
- PE-bear <sub>v0.7.1</sub>
- Ghidra <sub>v11.4.2</sub>

## Dynamic Analysis
- WinDbg <sub>v1.2511.21001.0</sub>
- x64dbg <sub>v0.0.2.5</sub>
- Procmon <sub>v4.0.1</sub>

## Auxiliary
- Detect-It-Easy <sub>v3.11</sub>
- HxD <sub>v2.5.0.0</sub>
- Spy++ <sub>v18.00.11101</sub>

# PROLOGUE
> A preface detailing the reasoning and approach.

## Motivation
I wanted to explore reverse engineering ― an often challenging area of cybersecurity ― through a real-world application rather than a purpose-built "crackme" or tutorial-style test program.

Initially, the goal was to deconstruct a target application in order to understand its components like the activation mechanism, validation routines etc.

While the early attempts were met with failure, I was able to refine my ability to reason about the intended design behind the target app and strengthen my proficiency with the relevant tools and methodology with each iteration.

## Historical Context
Puzzleball 3D is a Windows-based video game released in the early 2000s, with a free trial that relied on CD-keys for activation; a typical mechanism for the time.

This period of software distribution was also rife with "cracks" and "keygens", which were tools designed to circumvent and/or bypass legacy copy protection and enable piracy.

The publisher for Puzzleball 3D ― who also held rights over numerous other titles ― implemented a custom launcher tied to a proprietary DLL file in order to unify all their offerings under a single banner. The DLL file also contained validation and integrity checks in order to resist tampering. This became the target for crackers.

However, in 2009, an update to the DLL rendered existing keygens and bypass methods non-functional. Not long after, the publisher also ceased operations, leaving video games like Puzzleball 3D locked in a permanent "free trial" state.

## Initial Assumptions
I originally chose Puzzleball 3D for this project with the assumption that its "archaic" activation mechanism would be relatively simple to analyze and even overcome with a "bit flip" or conditional check modification.

I initially approached the problem with techniques and methodology that would've been more suited to modern applications, which while a lot more secure, are also standardized.

It was only after repeated failure that I realized the need to reassess my approach. Aside from documentation and symbols being absent, the application reflected design patterns that were specific to its era.

<img width="1280" height="720" alt="HDD vs RAM" src="https://github.com/user-attachments/assets/4b190b31-f82b-4115-a81a-3be3707d1fd9" />

For example, simple assets for the launcher like text preceding an input field were copied multiple times into memory instead of being "pulled straight from the disk". This may seem inefficient by today's standards but to preserve good user experience, the original developers likely made this tradeoff as hard drives back then were relatively slow.

This complicated analysis methods like tracing user input and decoding integrity checks, as will be demonstrated in a later part.

## Analysis Structure
As outlined in the Executive Summary, the overall plan (initially at least) was to move from:<br>
  _Initial Recon_ ➟ _Static Analysis_ ➟ _Dynamic Analysis_ ➟ _Summarize & Document_

The sections in this analysis phase will be divided into several stages:
- Parts 1 and 2 outline the intial approaches and why they failed.
- Part 3 details the path that ultimately succeeded.

This project is presented strictly for educational purposes. No cracks, activation keys, or hack tools intended on enabling piracy are provided.  

# Part 1 - Activation Mechanism
> GOAL: Decode the activation mechanism.

I began the analysis by first making simple observations of the application's features. While the initial goal was to decode the activation mechanism, I still had to establish whether that was actually feasible with Puzzleball 3D. Below are the findings:

## Initial Observations
### First Impressions
Launching Puzzleball 3D from either double-clicking the main .EXE or quick start icon presents a stylized "launcher" menu. The revision number is interestingly shown here at the bottom<br>
➠ Rev. 3744 Build 177.<br>

This was likely used for internal troubleshooting or to even help with customer support and does not appear to be relevant to the goal, considering the fact that this is a closed-source application that likely did not have a public repository with other versions made available.<br>

<img width="1280" height="720" alt="Launcher Window fixed" src="https://github.com/user-attachments/assets/878a21c3-5db1-42b9-a150-2aea369bdadf" /><br>

The highlighted text, ``...no additional download required``, indicates that the game install is complete in where all the required assets, data, and features are already present on the system and that the trial mode applies a digital/software lock. This is promising because it means reverse-engineering is feasible.

### Exploring The Menus
<img width="1280" height="720" alt="buttons" src="https://github.com/user-attachments/assets/6d8004b6-c7a4-4f98-b98b-1fa482416b92" />
<br>

Clicking on the ``Other Games`` button leads to an insecure and likely expired URL. Since the publisher's site is no longer functional (non-existent backend), exploits like SSRF and IDOR are not possible. They would require explicit permission from the server owners anyway and is something outside the scope of this project. But it is interesting to imagine how "fishing" for unlock codes stored in a database could've been a possibility.<br>

<img width="1280" height="720" alt="insecure site fixed" src="https://github.com/user-attachments/assets/03bcbe02-e329-4e69-875a-2164aeb54daa" /><br>

Clicking on ``Already Paid`` brings forth a new window into view with 4 separate tabs.
The text here indicates that an internet connection is required to activate the game or unlock the full version. We could intercept the application's attempt at "calling back to base" and host our own server to spoof the validation process but there is far too little information (protocol, request format, etc.) at this point to go down this route.<br>

<img width="1280" height="720" alt="already paid" src="https://github.com/user-attachments/assets/38ad7676-7886-4643-830f-8528bbc20c33" /><br>

Clicking on the hyperlink, ``I'm not connected to the internet``, renders a new screen displaying a URL and interestingly, a product code. This code doesn't seem to change with multiple restarts of the application so it is likely tied to some system property or attribute.
We'll note this down for future reference.<br>

<img width="1280" height="720" alt="not connected" src="https://github.com/user-attachments/assets/09fa4d07-54a2-424c-a656-d6f12651513d" /><br>

Going back to the dead URL, we see that the product code is also exposed as a value for the parameter, ``pid``, which likely stands for product ID. It is difficult to ascertain what ``did`` is, but we'll note down this URL for reference as well.

The other tabs are simply different payment methods for what used to be the way to obtain legitimate unlock/activation codes for Puzzleball 3D. They all require a response from the payment processor's server which is well outside the scope of this project.

## Static Analysis
After having established that reverse-engineering may be possible with Puzzleball 3D, I performed some basic static analysis using PE-bear to sift through the binary for string references and Ghidra to provide a graphical view of the functions.

### PE-bear Findings
Often times, the protection mechanism for an application is found in a separate binary or executable like a DLL file. This file runs in tandem with the main application and acts as a "security guard".

This is why "cracks" back then came with 2 files; a **modified .EXE** executable and a **modified DLL** file, both of which you would have to copy over to the root directory of the relevant application in order to overwrite and neuter whatever protection mechanism was in place.

In our case, a proprietary DLL file does exist in the root directory of Puzzleball 3D, which we'll refer to as ``RA.dll`` for both the sake of simplicity and obfuscation. The following are strings I found that stuck out the most from each binary:

#### Puzzleball 3D.exe
- ``radll_HasTheProductBeenPurchased`` @ offset 2aaa0
- ``unittest_GetBrandedApplicationID`` @ offset 2ab7c
- ``The DRM dll has been altered since it was generated and is not deemed to be secure`` @ offset 2ac70
- ``radll_GetUnlockCode`` @ offset 2a514

#### RA.dll
- ``Decryption Key Data=A/ZTZDGDQ7YGUERNPMP3VVVT4XFWSBBL`` @ offset b8818
- ``GlobalUnlock`` @ offset c6fde<br>
- ``radll_GetUnlockCode`` @ offset c7280<br>
- ``unittest_DecryptUnlockCode`` @ offset c73df<br>
- ➟ ``unittest_ValidateUnlockCode`` @ offset c7553<br>
- ``Action Fail Wrong Unlock Code`` @ offset c8660<br>
- ``Form Edit Control Containing Unlock Code`` @ offset c86f4<br>
- ``UnlockCode`` @ offset c8728<br>

Aside from the intriguing "Decryption Key Data" string in the DLL file, there were no hardcoded unlock codes found in either of the examined binaries. More importantly, the 3rd string listed under the main .EXE (@ offset 2ac70) essentially confirms that the copy protection or DRM (Digital Rights Management) is tied to the DLL file.

While all of these strings looked enticing to follow, for the purposes of this project, ``unittest_ValidateUnlockCode`` seemed like the one closest to our goal of decoding the activation mechanism. Looking this up with Ghidra's search tool led to a function with that exact name.<br>

### Ghidra Findings
<img width="1280" height="720" alt="validateunlockcode func" src="https://github.com/user-attachments/assets/8036a678-83e4-42d0-aa5e-47b77beeddb4" /><br>

The above is a "Ghidra representation" of the function,  ``unittest_ValidateUnlockCode``, translated from machine code into human-readable form. There are 3 variables being declared:
- cVar1, a single character data type.
- local_c and local_8, both integer data types.<br>

The routine seems to begin with some processing applied to the supplied arguments (*param_1 and *param_2) along with the aforementioned variables.

A new value for cVar1 is derived from the processing involving function ``FUN_100180d1`` before being compared against the value ``'\0'`` at the end which is likely a null byte.

To understand this routine better, I switched to the "Function Graph" view for ``unittest_ValidateUnlockCode`` in Ghidra to examine the "assembly logic"<br>
<img width="720" height="1280" alt="validateunlockcode assem" src="https://github.com/user-attachments/assets/866262db-c2c9-4190-9a19-ccf73ae941e3" /><br>

Beginning from the top, we see that the ECX register's value is pushed twice onto the stack in place of where the ESP register's value would typically be subtracted to "make space" for local variables. This could just be a form of compiler optimization.

We also see that the function, ``FUN_1007ccbd``, is applied, if you will, in a consistent manner to both ``param_1`` and ``param_2``. The preceding and following instructions to the function call are very similar. Another function towards the end, ``FUN_1007cdca``, also seems to be applied in this way. It could be reasoned that the former is some sort of "preparation step" and the latter a "finishing step".

These two functions are likely not where the core logic or math for the activation mechanism is located but I still thought it important to delve deeper into ``FUN_1007ccbd`` in an attempt to fully deconstruct the routine.<br>

#### FUNCTION 1007CCBD
<img width="1280" height="720" alt="1007ccbd" src="https://github.com/user-attachments/assets/a4241188-18d7-4a69-8e79-6c54ddc8a0de" />

Here we see a nested set of blocks that divide the assembly into several sections. Attempting to decode this without any dynamic analysis or runtime context took an immense amount of time and effort.

Suffice to say, this function essentially allocates system memory to load the variables and arguments in an "expected state" before being processed by the other routines found in ``unittest_ValidateUnlockCode``, namely ``FUN_10018172`` and ``FUN_100180d1``.

The presence of standard C library functions like ``_strlen``, ``_strncpy``, as well as ``operator_new`` (found within sub-function ``FUN_1007ebe0`` in ``FUN_1007ccbd``) are the biggest giveaways.

#### FUNCTION 10018172
<img width="1280" height="720" alt="10018172" src="https://github.com/user-attachments/assets/bdc224dd-b5b4-4ae8-bce3-4e7738de6ff7" />

Backing out of ``FUN_1007ccbd`` and moving onto ``FUN_10018172``, we see a routine that is even more convoluted. Attempting to decode this with only information and context available through static analysis alone was near impossible, largely due to unknown items like ``DAT_100dc868`` and ``DAT_100dc86c`` as well as the numerous amount of nested functions.

All that I could ascertain from this summarized view of ``FUN_10018172`` is that if the argument ``param_2`` contains a character 'F', then further processing is applied. Otherwise, the app's workflow would skip to the end and proceed to the next routine in the parent function which would be ``FUN_100180d1``.

## Making A Decision
Reverse-engineering is a process that can get quickly overwhelming if the right tools and methodology aren't applied. But as we'll see later, even that may not be enough to fully deconstruct something like Puzzleball 3D which, while not seriously obfuscated, is a relatively ancient application with peculiar design philosophies that don't necessarily agree with modern programming standards.

After having spent countless hours trying to uncover the purpose of ``FUN_100180d1`` but ultimately ending with nothing substantial, I deemed this approach a "failure". This however, did not stop me from stepping back and taking an entirely different angle, as will be detailed in Part 2.
