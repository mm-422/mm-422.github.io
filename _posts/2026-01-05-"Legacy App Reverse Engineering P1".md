---
title: "Puzzleball P1 - Reverse Engineering"
date: 2026-01-05 00:00:00 +/-TTTT
categories: [research, reverse engineering]
tags: [windows, reverse engineering, ghidra]     # TAG names should always be lowercase
image: "/assets/images/reverse_p1.png"
---

## Executive Summary
This project documents a thorough analysis of an early-2000s era application and its components by way of Reverse Engineering. Aspects such as the authentication mechanism, file integrity checks, UI construction and more are covered.

The goal is to demonstrate the importance of understanding an application's intended design and behavior **before** any application of technical steps and methodologies.

The legacy application in question is a video game that was primarily distributed on Windows circa early to mid 2000s. This game implemented custom routines and file validation mechanisms in order to fight off tampering.

While the original servers and publisher are no longer operational, for the sake of ethics, the exact name of this application and images of its components will be obfuscated as necessary.
Hence, the video game app will be referred to as "Puzzleball 3D" throughout the case study.  

### ♦️ Project Goals
- Analyze the structure and behavior of legacy authentication mechanisms.
- Reverse engineer authentication logic with no source code or documentation.
- Demonstrate the importance of thorough research to provide context prior to forming hypotheses and  testing.<br>

### ♦️ Overall Plan
- Start with basic reconnaissance to gather preliminary info for later analysis.
- Apply static analysis methods to identify functions and algorithm patterns.
- Move to dynamic analysis to observe application behavior.
- Determine potential vulnerabilities and attack vectors.
- Document findings.

### ♦️ Scope & Ethical Considerations
- This project focuses on understanding mechanisms and applying methodology.
- This project _**DOES NOT**_ distribute material that would encourage piracy.
- <mark>No KeyGen or hacktool creation is demonstrated.</mark>
- Any example of weaponization potential is done under an educational lens.
- While the application for this project can be found on sites like the Internet Archive, no form of source code or vendor documentation is available.<br>

### ♦️ Technical Summary
- Observed app behavior with regard to user input.
- Performed static & dynamic analysis on both the main executable and an auxiliary DLL.
- Investigated file integrity checks within application's binary.
- Identified validation logic and performed bypass.
- Evaluated security weaknesses and weaponization potential.<br>

### ♦️ Tools Used
```
Disassemblers
• Ghidra
• x64dbg

Primary Debugger
• WinDbg

Process Inspection
• Procmon
• Spy++

Auxiliary
• HxD for editing hex.
• PE-bear for quick string searches.
• Detect-It-Easy for app property inspection.
```
_All testing done in a controlled Windows 10 22H2 VM._

### ♦️ Takeaways
- Focus should be on understanding the intended design behind an application.
- Most if not all client-side validation is "doomed" with modern analysis and debugging tools.
- Clear documentation helps tremendously with prolonged debugging sessions.
- Design oversights and security mistakes seen in a 20-year old app can still be seen today.<br>

___

## The Environment
A testing environment including all the necessary tools and appropriate settings is paramount to reverse engineering, especially when analyzing (potentially) malicious software.

I prefer using a virtual machine for these tasks as they are easy to spool up with the exact configurations required. The "running state" can also be saved with a click of a button; **invaluable for long debugging sessions.**

Below are the environment details along with the specific settings applied.

### ♦️ The Virtual Machine
- Set up Windows 10 22H2 64-bit on VirtualBox v7.2.4.r170995.
- Puzzleball 3D was likely designed for Windows XP 32-bit.
- However, replicating the exact OS env is not necessary for the goals of this project.
- Allocated 4 logical cores (Zen 3 CPU) and 8GB of memory to Virtual Machine.
- This was sufficient for up to 3 instances of Ghidra + 1 instance of WinDbg simultaneously.

![VM-specs-scrnshot](/assets/images/specs.avif)

### ♦️ Files & File Paths
- Created a backup of Puzzleball 3D's files on external media.
- Installed Puzzleball 3D in default directory under "Program Files (x86)".
- Duplicated Puzzleball 3D's root directory and files to Desktop.
- All testing and modifications to the app's binaries was performed on the files here.

### ♦️ OS-level Settings
- Disabled network connection.
- Added the "Desktop" directory to Windows Defender's exception list.
- Left ASLR and default memory integrity settings like Core Isolation alone.

### ♦️ Tools & Process
Initially, I relied on just x64dbg and PE-bear to do the heavy lifting. But this was only sufficient for some static analysis. Eventually, I moved on to Ghidra + WinDbg as my potent duo, while also discovering plenty of incredible tools like Spy++ and Detect-It-Easy along the way.

I would also be remiss to not mention the incredible use I've found in leveraging LLMs like ChatGPT and Gemini in order to perform tasks such as research for application design philosophies relevant to the early 2000s era as well as sifting through hundreds of lines of assembly code.

Care should be taken when utilizing these technologies, especially when sensitive info is concerned. Considering the fact that Puzzleball 3D is openly available on sites like the Internet Archive with no relevant owners to contact and that hosting a local LLM would present its own set of challenges (cost, speed, reliability), the use of online hosted LLMs is justified in my opinion.
```
Static Analysis
• PE-bear v0.7.1
• Ghidra v11.4.2

Dynamic Analysis
• WinDbg v1.2511.21001.0
• x64dbg v0.0.2.5
• Procmon v4.0.1

Auxiliary
• Detect-It-Easy v3.11
• HxD v2.5.0.0
• Spy++ v18.00.11101
```
___

## Prologue
### ♦️ Motivation
I wanted to explore reverse engineering (an often challenging area of cybersecurity) through a real-world application rather than a purpose-built "crackme" or tutorial-style test program.

Initially, the goal was to deconstruct a target application in order to understand its components like the activation mechanism, validation routines etc.

While the early attempts were met with failure, I was able to refine my ability to reason about the intended design behind the target app and strengthen my proficiency with the relevant tools and methodology with each iteration.

### ♦️ Historical Context
Puzzleball 3D is a Windows-based video game released in the early 2000s, with a free trial that relied on CD-keys for activation; a typical mechanism for the time.

This period of software distribution was also rife with "cracks" and "keygens", which were tools designed to circumvent and/or bypass legacy copy protection and enable piracy.

The publisher for Puzzleball 3D ― who also held rights over numerous other titles ― implemented a custom launcher tied to a proprietary DLL file in order to unify all their offerings under a single banner. The DLL file also contained validation and integrity checks in order to resist tampering. This became the target for crackers.

However, in 2009, an update to the DLL rendered existing keygens and bypass methods non-functional. Not long after, the publisher also ceased operations, leaving video games like Puzzleball 3D locked in a permanent "free trial" state.

### ♦️ Initial Assumptions
I originally chose Puzzleball 3D for this project with the assumption that its "archaic" activation mechanism would be relatively simple to analyze and even overcome with a "bit flip" or conditional check modification.

I initially approached the problem with techniques and methodology that would've been more suited to modern applications, which while a lot more secure, are also standardized.

It was only after repeated failure that I realized the need to reassess my approach. Aside from documentation and symbols being absent, the application reflected design patterns that were specific to its era.

<img width="1280" height="720" alt="HDD vs RAM" src="https://github.com/user-attachments/assets/4b190b31-f82b-4115-a81a-3be3707d1fd9" />

For example, simple assets for the launcher like text preceding an input field were copied multiple times into memory instead of being "pulled straight from the disk". This may seem inefficient by today's standards but to preserve good user experience, the original developers likely made this tradeoff as hard drives back then were relatively slow.

This complicated analysis methods like tracing user input and decoding integrity checks, as will be demonstrated in a later part.

### ♦️ Analysis Structure
As outlined in the Executive Summary, the overall plan (initially at least) was to move from:<br>
``Initial Recon`` ➟ ``Static Analysis`` ➟ ``Dynamic Analysis`` ➟ ``Summarize & Document``

The sections in this analysis phase will be divided into several stages:
- Parts 2 and 3 outline the intial approaches and why they failed.
- Part 4 details the path that ultimately succeeded.

This project is presented strictly for educational purposes. No cracks, activation keys, or hack tools intended on enabling piracy are provided.
