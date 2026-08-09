---
title: "Legacy App Reverse Engineering P3 - Decoding User Input"
date: 2026-01-05 00:00:00 +/-TTTT
categories: [research, reverse engineering]
tags: [windows, reverse engineering, ghidra]     # TAG names should always be lowercase
image: "/assets/images/reverse_p3.png"
---
## Tracing User Input
One crucial step I missed in Part 2 was to verify if the function, ``unittest_ValidateUnlockCode``, was even a relevant part of Puzzleball 3D's binary used during any actual code validation process ie. _when inputting an unlock code and clicking submit._

If we looked closer to examine the name of the function alone, the word "unittest" here is a hint. 
In software programming, "unit testing" is a method for verifying if individual components of an application are working as expected. In this case, the ``unittest_ValidateUnlockCode`` function block is likely tied to ensuring that user input validation (like an unlock key) executes as expected.

This does not necessarily mean that the actual function used for validating real user input has a similar form to the "unit test" counterpart. Often times, it is good practice to design specific and isolated code blocks that utilize automation for unit testing. But as we will see later, this isn't always the case with Puzzleball 3D.<br>

## New Approach
> GOAL: Verify validity of unittest_ValidateUnlockCode.

Determining the relevancy of ``unittest_ValidateUnlockCode`` is not something that could be done through static analysis alone. To verify an application's behavior would typically require "stepping through code" with a debugging tool.

To do this, I used x64dbg to set a memory breakpoint on ``unittest_ValidateUnlockCode``, and then proceeded to go through the activation process within Puzzleball 3D's launcher.

### ♦️ Hypothesis
If the memory breakpoint in x64dbg is triggered, then we could revisit the function with a different approach. Otherwise, we would need to trace the user input to the function that **actually performs the activation** or validation step.

### ♦️ x64dbg Testing
Setting a memory breakpoint in x64dbg can be done through the "Symbols" tab. This is where all of the modules (other libraries needed for app's functionality) are listed along with their imported and exported "symbols" which can include variables, data structures, and functions.

<img width="1280" height="720" alt="x64dbg symbols" src="https://github.com/user-attachments/assets/d8a44470-7d96-4bf5-b13d-35d4b1f95dae" />

As we've established earlier, ``ra.dll`` is the binary where the protection mechanism is likely located and is where the ``unittest_ValidateUnlockCode`` function is found under the Symbols tab as expected. 
To set a breakpoint here, we simply right click on the target function and select "Toggle Breakpoint" or press the F2 shortcut key.

This breakpoint will then be listed under the "Breakpoints" tab in x64dbg with a "hit count" next to it to indicate the number of times the breakpoint was triggered during app runtime.

Now, all that's left to do is to carry out a test to see if the hypothesis is true or false. I navigated to the sub-menu shown previously past the ``Already Paid`` button, then clicked on the hyperlink to bring me back to the window where the product key was displayed along with an input field for an unlock code.

<img width="1280" height="720" alt="not connected" src="https://github.com/user-attachments/assets/3679889d-e5f6-488c-bb22-a96da99c0c9a" />

I entered a random string of characters (234A) and then clicked the ``SUBMIT`` button. If the ``unittest_ValidateUnlockCode`` function was indeed tied to the activation mechanism here, then the  breakpoint set in x64dbg should have been hit and the hit counter increased by 1.

It is also typical for the application being debugged to appear "frozen" or "halted" when this happens because of its execution being "held in place" at the breakpoint, so to speak, by the debugger (x64dbg in this case).

Unfortunately, neither of these events occurred and all I got was a pop-up window telling me that I've entered an "unrecognized unlock code".

<img width="1280" height="720" alt="Unrecognized Code" src="https://github.com/user-attachments/assets/600f91fa-bf4d-460c-a0c8-37288203732d" />

With this, we can confirm that ``unittest_ValidateUnlockCode`` **is not involved** in the activation process. It could even be vestigial code from early development or debugging that isn't necessarily reflective of the actual math or logic behind the real activation mechanism of Puzzleball 3D.

Now we need to find the actual activation mechanism or function responsible for validating unlock codes.

___

## Recalibrating
> GOAL: Locate the routine responsible for activation.

At first, I tried a "brute-force" approach by setting breakpoints on all the of "named functions" listed under ``ra.dll`` as exports in the Symbols tab.

I then proceeded to repeat the activation process on the launcher.

While some breakpoints for certain functions were hit at the startup of Puzzleball 3D, like even ``unittest_GetBrandedApplicationID``, the result upon clicking the submit button was the same. An "unrecognized unlock code" message box.

So then, if the function responsible for validating unlock codes was not one of the exported ones from the DLL file, could it be located in the main application executable? This was an assumption that I had that led me on a long and frustrating goose chase that ultimately ended with nothing substantial. After some research, I decided to try tracing the user input again but with a different method involving the **Windows Messages** system.

### ♦️ An Intro to Messages
**Windows Messages** are units of data used to communicate events - _like mouse clicks and keyboard presses_ - between the OS and applications.

Each application window in the **Windows** operating system is placed under an "Application Thread" that contains an internal queue of these events. Each event can be thought to possess a label like ``WM_LBUTTONDOWN`` for a mouse click.

If this mouse click happened after a keyboard press (``WM_KEYDOWN``) for example, then you can visualize the event queue as being a chronological order like this:<br>
> ``WM_KEYDOWN``⤵︎<br>
> ``WM_LBUTTONDOWN``

### ♦️ The Message Flow
Typically, most if not all elements in a modern app (like buttons and text boxes) are their own "windows" in that they have a unique ID or handle called an ``HWND`` (Window Handle). An example of this might be something like, ``00451233``.

A function called ``GetMessage`` within these apps runs a "Message Loop" that keeps track of all these events and handles. The typical flow from user input to app output is as follows:

1. You click a button, kernel identifies the ``HWND`` of the window right beneath the mouse cursor.
2. The OS sends a message (``WM_LBUTTONDOWN``) along with the corresponding HWND to app's event queue.
3. ``GetMessage`` will see this event and pass it to another function called ``DispatchMessage``, which performs a check - _tracing the ``HWND`` to the specific window ie. button_ - before handing over to the corresponding window's ``WNDPROC`` or Window Procedure routine.
4. ``WNDPROC`` is a "central" callback function that contains the **logic** for the window. This is where the execution steps following a specific event is initiated. For example, a mouse click causing a part of the app to change color or launch a new instance.

Developers typically override or customize the ``WNDPROC`` function and its name to suit the intended functionality.

### ♦️ Hypothesis
If we could find the exact name of the ``WNDPROC`` implementation in Puzzleball 3D and trace the ``HWND`` of the user input field or ``SUBMIT`` button through the application's binary, we might be able to locate the core activation mechanism.

### ♦️ Spy++ Testing
We can utilize a tool called Microsoft Spy++ to easily uncover the properties of a target window such as the HWND and parent class of a button.

To do this, we simply locate the process with the application's name (Puzzleball 3D in this case) within Spy++ and open the Messages window. Here we can see a long list of events:

<img width="1280" height="720" alt="spy++ test" src="https://github.com/user-attachments/assets/6d628381-0d40-4285-8b08-e319187d8f7d" />

We then perform an interaction with the app - _like clicking a button_ - and then locate the corresponding event from the list. Puzzleball 3D's launcher also seems to run a continuous "hit test" as can be seen with the numerous ``WM_NCHITTEST`` messages. I didn't think too much of this at the time but the reason will become apparent in a later part.

Anyway, the ``HWND`` has been identified as ``00080428``. However, pointing or clicking on any space within the launcher window (apart from the top menu bar) with the "Finder Tool" in Spy++ gives us the same ID as well.

I restarted Puzzleball 3D and performed the same test. The ``HWND`` was now ``007303BA``. It seemed that the handle changed with each run but stayed constant for all elements displayed on the launcher window. 

<img width="1280" height="720" alt="spy++ inspect" src="https://github.com/user-attachments/assets/9fd7f959-e003-465a-b604-ba1f114141fb" />

### ♦️ Another Curveball
The plan was to locate the specific ``HWND`` tied to either the input field or the ``SUBMIT`` button, and then to trace it to the activation mechanism in Puzzleball 3D's binary by setting a "handle breakpoint" using that ``HWND`` with x64dbg.

As the handle was shared between all elements or "windows" on Puzzleball 3D's launcher, this led to a lot of frustration. Even utilizing the "Handles" feature in x64dbg to set a breakpoint on the specific hardware event listed on the Spy++ Message Window (like ``WM_LBUTTONDOWN``) led to a lot of dead ends. Almost none of the numerous message breakpoints tested were hit.

After a bit of research, it turned out that the launcher for Puzzleball 3D is a "custom-drawn window" where elements like the buttons are simply graphical effects that trigger when the cursor hovers above them and are not real buttons that possess their own handle. These "fake buttons" share the same ``HWND`` with the rest of the launcher window because it is not a separate element.

Puzzleball 3D ran a custom routine that tracked the cursor's coordinates within the launcher window and correlated that data with events like mouse clicks in order to trigger specific functions. This tracked with the observation of the launcher being un-resizeable. This was likely done to prevent the tracking functionality from breaking.

Even attempting to locate the heart of this tracking routine in the app's binary - _with no symbols, no breakpoints, and no documentation/reference_ - was incredibly difficult.

After much frustration, I deemed this approach as another "failure" and reconsidered my options.

___

## Continued 1
The complications I encountered throughout trying to decode Puzzleball 3D with its relatively opaque routines and unconventional design led me to take a few steps back to re-evaluate my approach.

I decided to pick a simpler part of the application to modify this time in order to **gather more information** on how it responds to analysis and debugging tools. After the previous failures, I wanted to eliminate any possible compatibility issues and ensure there were no "invisible" obfuscation steps that made using tools more challenging than necessary.

> GOAL: Fix typo on launcher window.<br>

To go about this, I wanted to remedy a misspelling of the word "computer" on the launcher window where the product key was displayed.

<img width="631" height="88" alt="typo" src="https://github.com/user-attachments/assets/2dfb1fc2-3a78-4aeb-8225-25d73ff8f6f9" />

The word "compter" here is missing a vowel, which means we'd have to add an extra byte somewhere in the program's binary or maybe even an external resource file in order to correct the word.

Performing a quick string search through the main .EXE and even the ``ra.dll`` file for the word "compter" returned nothing. It was very unlikely that the application had a function to "construct" words and sentences procedurally, especially on a custom-drawn window. Moreover, the relatively low text quality was also a giveaway; natively rendered text wouldn't exhibit as much aliasing as can be clearly seen in the screenshot images.

So I looked through the root directory of Puzzleball 3D and came across a file called ``Arcade.dat``. This file was of a non-standard "DAT" type. To quickly inspect its contents, I attempted to open it with **Notepad** which presented what looked like a header with unintelligible characters, followed by readable text. Using the "find" tool, I was able to locate the misspelled word.

<img width="1280" height="720" alt="arcade notepad" src="https://github.com/user-attachments/assets/f6d89b1a-a536-4585-a1af-224681e918f6" />

It should be stated here that reading and modifying files with arbitrary extensions using a tool like Notepad is ill advised as it could lead to corruption. I knew even at the time that doing so was less than ideal, but it allowed for quickly sifting through the contents of ``Arcade.dat`` in order to locate the desired word.

Changing the word "compter" to "computer" through Notepad seemed trivial, and the launcher even started up as expected. However, clicking on the ``Already Paid`` button to access the sub-menu where the product key was displayed presented this error:

<img width="1280" height="720" alt="fatal not found" src="https://github.com/user-attachments/assets/c953b7e0-a388-4db8-8891-a6d615f7f5cf" />

This essentially confirms that some if not all of the elements on the sub-menu draw from the Arcade.dat file. This also adds up with the fact that the contents of that file resemble a framework, perhaps used for constructing the launcher's graphical interface.

What's interesting is that reverting the word change doesn't seem to fix or undo this error. It should also be noted that the title of the error, "Fatal Not Found", alludes to the application keeping track of the .dat resource file in some way, perhaps with an internal ID or hash. Tampering with it with the Notepad edit likely caused this internal ID to change or become corrupted.

The text within that relatively large error dialog box also resembles "verbose language" that would be used in a development environment for debugging purposes. Certainly not something intended for the end-user.

## Taking A Detour
> GOAL: Trace the "File Not Found" error.<br>

At this point in the project, I assumed that a validation mechanism of some sort kept track of Arcade.dat in order to detect tampering. So I searched through both the main .EXE and ra.dll with Ghidra for any references to the error string "Fatal Not Found".

I ended up finding a row of defined strings in both binaries that contained all the text shown in the previous error dialog box.

<img width="1280" height="720" alt="fatal not found ghidra" src="https://github.com/user-attachments/assets/97939581-3d13-4f5d-beba-ad7728c41235" />

This was an interesting bit of redundancy. The text duplication could just be residual data that survived post-development, in which case it would be next to meaningless for our goal.

To verify if this was the case, and to see which binary (main .EXE or ``ra.dll``, if either) provided the data for the "Fatal Not Found" dialog box, I made a slight change to that string reference in Ghidra (starting with the main .EXE) to see if it would show the next time the error dialog was drawn.

However, doing this caused Puzzleball 3D to throw the following error message on startup:

<img width="1280" height="720" alt="CRC error" src="https://github.com/user-attachments/assets/da3c25b9-a100-4fd7-bb34-51e73c8526d9" />

This error indicates a CRC (Cyclic Redundancy Check) failure of some sort, which might point to the existence of a routine in the main .EXE that verifies its integrity.

I tried following this "CRC failed" string through the XREF shown in Ghidra and landed on the function ``FUN_0040283b``. Attempting any sort of modification to the assembly here, or anywhere for that matter, caused the application to throw another error dialog stating that "Game Files Are Corrupt".

Tracing this new error through the main .EXE brought me to function that seemed to be a common denominator between the two errors ► ``FUN_00401006``  

<img width="1280" height="720" alt="401006 assembly" src="https://github.com/user-attachments/assets/801456be-3a41-4bb9-8dc5-6da7c7403ef9" />


In order to understand how Puzzleball 3D was making decisions with regard to which error dialog to show, I decided to perform a full breakdown of ``FUN_00401006``. The fact that this routine possessed numerous inbound XREFs hinted at it being a global initializer of some sort.

Both ``LAB_0040102b`` and ``LAB_00401031`` appeared to load data segments into registers before either returning or jumping to a different section as if "prepping" the application. They are likely responsible for loading resource files like ``Arcade.dat`` and populating internal structures.

The first prominent function call here seems to be ``FUN_00401d0b`` which is part of the routine chain that eventually leads us to the location of the error strings.  

```
int * __fastcall FUN_00401d0b(int *param_1)
{
uint * _Str;
uint *puVar1;
size_t sVar2;
HANDLE hFindFile;
undefined4 uVar3;
int *piVar4;
CHAR local_944 [2048];
_WIN32_FIND_DATAA local_144;

FUN_004027ab((int)param_1);
param_1[0x4c] = 0;
param_1[2] = 0;
param_1[3] = 0;
GetCurrentDirectoryA(0x104,(LPSTR)(param_1 + 0x4d));

FUN_00401e50();
FUN_004025bf((int)param_1);
_Str = (uint *)(param_1 + 9);
FUN_00401e36((LPSTR)_Str,0x104);
puVar1 = FUN_004170f0(_Str,&DAT_0042e510);
while (puVar1 !=  (uint*)0x0){
  puVar1 = FUN_004170f0(_Str,&DAT0042e510);
  FUN_00417890(_Str,(uint*)((int)puVar1 + 1));
  puVar1 = FUN_004170f0(_Str,&DAT_0042e510);
}

puVar1 = FUN_004170f0(_Str,&DAT_0042e50c);
while (puVar1 != (uint*)0x0){
  puVar1 = FUN_004170f0(_Str,&DAT_0042e50c);
  FUN_00417890(_Str,(uint*}((int)puVar1 + 1));
  puVar1 = FUN_004170f0(_Str,&DAT_0042e50c);
}

sVar2 = _strlen((char*)_Str);
FUN_00417890((uint*)(sVar2 + 0x21 + (int)param_1), (uint*)&DAT_0042e508);
hFindFile = FindFirstFileA((LPCSTR)_Str,&local_144);
if (hFindFile == (HANDLE)0xffffffff){
  FUN_00417890(_Str,(uint*)s_RAW_001.exe_0042e4fc);
}

FindClose(hFindFile);
FUN_00401e36(local_944,0x800);
FUN_004027c6(param_1,local_944);
piVar4 = param_1 + 0x4a;
uVar3 = FUN_004027c2((int)param_1);
FUN_00401091(s_RA\RAW_002.wdt_0042e4e0,param_1,uVar3,piVar4);
return param_1;
}
```

The above is the decompiled view of ``FUN_00401D0B`` as presented in Ghidra. Decoding the assembly for this routine was a long and arduous task. So, for the sake of brevity, I will skip the full analysis and list down the most important findings.

At this point of the project, ``FUN_00401D0B`` seemed like an initializer for integrity checks relating to file corruption. It starts by gathering directory and module names and attempts to locate files\binaries like ``RAW_001.exe`` and ``RAW_002.wdt``, the former being an existing file in the root directory of Puzzleball 3D. Interestingly, ``RAW_002.wdt`` was not a file that existed anywhere on the system (root, Documents, Local App Data, etc.) and may have been a "fall back" for if ``RAW_001.exe`` was not found.

The call to function ``FUN_00401091`` which is found under ``FUN_00401DEB`` is likely what leads to the construction of the "Game Files Are Corrupt" error dialog.

By using x64dbg, I was able to determine that with the current modifications done to Puzzleball 3D's binary, the app took the path towards ``FUN_004027C6`` which is presumed to be a wrapper for the "CRC fail" error. This is our next point of examination.  

```
**FUN_004027C6**
undefined4 __thiscall FUN_004027C6 (void * this, LPCSTR param_1)
assume FS_OFFSET=0xffdff000

undefined4  EAX:4  <RETURN>
void *  ECX:4(auto)  this
LPCSTR  Stack[0x4]:4  param_1

PUSH EDI
MOV EDI,this
CMP dword ptr [EDI+0x4],0x0
JNZ LAB_0040282A
PUSH ESI
CALL FUN_00402830
PUSH dword ptr [ESP+param_1]
PUSH EAX
CALL FUN_0040283B
POP this
MOV ESI,PTR_s_RA\RAW_003.wdt_0042E6C0
POP this
MOV this,dword ptr [PTR_s_RA\RAW_003.wdt_0042E6C0]
JMP LAB_004027FC


**LAB_004027EE**
PUSH this=>s_RA\RAW_003.wdt_0042EB70
PUSH EAX
CALL FUN_0040283B
POP this
ADD ESI,0x4
POP this
MOV this,dword ptr [ESI]=>DAT_0042E6C4


**LAB_004027FC**
TEST this,this
JNZ LAB_004027EE
MOV this,dword ptr [PTR_s_RA\Background.jpg_0042E6C8]
MOV ESI,PTR_s_RA\Background.jpg_0042E6C8
JMP LAB_0040281B


**LAB_0040280D**
PUSH this=>s_RA\Background.jpg_0042EB50
PUSH EAX
CALL FUN_00402913
POP this
ADD ESI,0x4
POP this
MOV this,dword ptr [ESI]=>PTR_s_RA\button_normal.jpg_0042e6cc


**LAB_0049281B**
TEST this,this
JNZ LAB_0040280D
PUSH EAX
CALL FUN_00402834
POP this
MOV dword ptr [EDI+0x4],EAX
POP ESI


**LAB_0040282A**
MOV AL,0x1
POP EDI
RET 0x4
```

This function is quite evidently loading a "list" of items into memory with references to asset files like ``RAW_003.wdt``,``background.jpg`` and ``button_normal.jpg``. Some of these resources are used when drawing the custom launcher window.

Functions ``FUN_0040283B`` and ``FUN_00402913`` are called to perform some processing on those assets before the call to ``FUN_00402834`` which returns a result that is stored in ``[EDI+0x4]``, then moved to the EAX register just before the final RET instruction.

This result may be the value that indicates the expected or computed "CRC state".

### ♦️ Hypothesis
Since ``FUN_004027C6`` seems to compute some sort of hash or value based on a "static list" of items and then stores this result in ``EDI+4``, and then in EAX, before it gets passed back to the parent, ``FUN_00401D0B``, theoretically, editing the value of EAX right before the last step, to something matching an original and untampered version of the main executable, should allow us to bypass the "Game File Are Corrupt" check.

### ♦️ WinDbg Testing
I first needed to observe the expected value in the EAX register at the right moment. To do this, I set a memory breakpoint at address ``00402826``, which was the start of ``FUN_00402834`` (called under ``LAB_0049281B`` as can be seen above) through WinDbg with the command ► ``bp 00402826``

The value contained in the EAX register at this moment was ``5C1D48A2``. I noted this down and relaunched Puzzleball 3D, this time with the "modified" version of the executable that threw out the CRC errors.
<img width="780" height="222" alt="EAx mod" src="https://github.com/user-attachments/assets/3dc13070-5d00-40c6-8fd5-fe83df72f6e7" />

I changed the ``MOV EAX, dword ptr [ESP + param_1]`` line to instead move an explicit value (``5C1D48A2``) into the EAX register with the following instruction ► ``MOV EAX,0x5C1D48A2``

I then "nullified" the following line by changing ``NOT EAX`` to a simple ``NOP`` instruction.

This then returned the expected value in EAX right before the call to function ``FUN_00401091`` in ``FUN_00401D0B``.

After performing this modification, the application launched normally through the modified main .EXE and we are now free to make alterations to that executable without tripping any integrity checks. It was now time to check the launcher's sub-menu again and examine the typo.

### ♦️ Back On Track
With the CRC and "corruption" checks neutered, we could now load the version of Puzzleball 3D's executable with the edited strings to see if the "Fatal Not Found" error dialog reflected anything different.
<img width="711" height="470" alt="same error" src="https://github.com/user-attachments/assets/83ebc6f6-2742-433d-aeec-99ca69fffdfe" />

Unfortunately, the error dialog seems unchanged. Modifying any of the string elements here that are found in the main .EXE doesn't break the application now but the changes don't seem to be reflected for some reason. I tried restarting the VM in order to eliminate any possibility of the app or OS maintaining a cache for the launcher's elements but this did not change the result either.

There was only one possibility I could think of for this scenario ► Puzzleball 3D was drawing the text elements from another source. The DLL file,``ra.dll``, was the next likely culprit. It contained an exact copy of all the strings found in the error in almost the exact order in its binary.

<img width="896" height="793" alt="dll fatal not found" src="https://github.com/user-attachments/assets/b6c8e6cb-37cf-45ba-8fb8-f6af92f2dc28" />

Modifying the strings here through Ghidra is once again trivial. Simply right-click on the defined string and select "Patch Data", enter the desired set of characters, and then press the "O" key for the shortcut to compile and output the modified DLL.

However, performing this now caused Puzzleball 3D to throw a new error dialog on startup.
<img width="368" height="129" alt="dll app error" src="https://github.com/user-attachments/assets/fde27c52-dd43-445f-a3e8-892112680302" />

The message displayed here is quite generic in that there were no codes or detailed information to go off of. I did however notice a new log file generated in Puzzleball 3D's root directory called ``RA Error.txt`` right after the error above. Opening it to examine the contents revealed the following:
<img width="1280" height="720" alt="DRM dll altered" src="https://github.com/user-attachments/assets/83965227-c508-494b-b766-cedf586fc681" />

This essentially confirmed the existence of another "internal check" for the DLL's integrity. I double-confirmed this by restoring the original DLL file, which allowed the app to launch normally again. We now had to neuter this DLL validation mechanism as well before being able to proceed further.

>GOAL: Modify the app to allow for modded DLL.

Bypassing the DLL integrity check was a much more convoluted process than bypassing the one in the main .EXE, with assembly instruction patches leading to various assertion failures, application crashes, and even seemingly obvious "bit flip" opportunities actually causing Puzzleball 3D to freeze before a fatal error.

For the sake of brevity, I will skip to the exact sub-routine in the main .EXE where the mechanism for verifying the DLL's integrity was found. This was done by first searching for the string "There was an error initializing the application. It will now exit", that was shown in the previous dialog box, which eventually led me to a function called ``FUn_004076C1``.

Decoding this required plenty of trial-and-error and "app behavior comparisons" between the original ``ra.dll`` file and a tampered one through extensive use of WinDbg. Below is the breakdown.

<img width="526" height="338" alt="4076c1" src="https://github.com/user-attachments/assets/b0229320-aecd-4269-bcd2-6e8d021e978d" />

``LAB_004076C1`` is a sub-routine of the parent function, ``FUN_004075C0`` which performs specific checks on different parts of the application. ``LAB_004076C1`` seemed tied to Puzzleball 3D's DRM functionality.

Examining the first function call here to ``FUN_00407160``, I discovered that it invoked modules such as ``CryptImportKey``, ``CryptHashData``, and ``MS Base Cryptographic Provider v1.0`` which are all cryptography related functions. This was a telling sign that ``FUN_004076C1`` was indeed responsible for verifying the integrity of the DLL.

Delving into the process of attempting to crack the above hashing algorithms may have been an even more daunting task. But if we look closer at the above sub-routine, we can see that there is a TEST instruction right before a jump or JNZ, and critically, a call to the ``Kernel32.dll`` module for the FreeLibrary function. This likely meant that the call to ``FUN_00407160`` with all its crypto-related modules only served to perform hash and signature related functions on ``ra.dll`` and the process of rejecting or unloading the actual DLL was performed farther down the assembly.

### ♦️ Hypothesis
<img width="1280" height="720" alt="patching JNZ" src="https://github.com/user-attachments/assets/8d061906-e3c0-4ede-be42-6d94d3a62325" />

Patching the assembly here to return an expected non-zero value right before the JNZ instruction might be sufficient. This might force the application's flow to "escape" the following lines of assembly in this sub-routine that unload the DLL and move to construct the error message found in the log file.

## Register Injection with WinDbg
I first set a memory breakpoint at the address ``004076d0`` which was right before the ``TEST EAX,EAX`` instruction. I then performed a "live register injection" by passing the following command into WinDbg ► ``r eax=1``

Injecting this non-zero value was sufficient to force the application to take the path to ``LAB_00407701`` where instructions were executed to initialize the DLL as if it were a genuine, untampered binary.

But this "live patch" was not very practical as it required repeating the steps each time Puzzleball 3D was launched. So I proceeded to create a more permanent workaround by modifying the instruction at ``004076D0`` in ``LAB_004076C1`` from ``AND EAX,0xff`` to ``MOV EAX,0x1``. This was better than simply changing the nearby JNZ to a JMP as an improper EAX value might adversely affect the application's flow down the line.

Fortunately, this patch was sufficient for the application to once again resume regular functionality. We could now move back to modifying ``Arcade.dat`` and attempt to fix the typo in the next part.

___

## Continued 2
With the integrity checks for the main executable and the DLL both bypassed, Puzzleball 3D's files should now be free to modify without worry of breaking app functionality. To confirm this, we can revisit the previous attempt to narrow down the source for the ``Fatal Not Found`` error dialog's text elements.

<img width="445" height="446" alt="list sucks" src="https://github.com/user-attachments/assets/bb3eb745-96b8-427f-bb23-20dbb9423acd" />

As can be seen above, the change made to the string in Part II is now reflected in the dialog box. This proves that the error message's text is indeed **pulled from the** ``ra.dll`` **binary.**

While this does not solve the actual problem causing the error in the first place, it does help give direction for the next step; to examine ``ra.dll`` for a way to fix the  ``Fatal Not Found`` error with the confidence that there are no integrity checks or "noise" now that might get in the way.

## Bypassing Arcade.dat Check
>GOAL: Disable or fix "Fatal Not Found" error.

As mentioned previously, the wording in the above error dialog indicates that the information was likely intended for the original developers of Puzzleball 3D, not the end user. There are also bizzare details like ``Arcade Main 1024`` which is not a file or path that exists on the system. There is no apparent "Fonts" list or "Resources" folder in the root directory either but we will get back to this in Part III.

At this point in the project, if there is a mechanism for validating the ``Arcade.dat`` file before it is loaded into memory perhaps and used for the launcher menu's construction, it could be similar in structure to the "integrity-check" implementations seen previously for the main .EXE and the DLL.

I began by searching both of the binaries for references to ``Arcade.dat`` and ``Arcade Main 1024`` among others. I then attempted to trace them "up" to a parent function in order to determine an appropriate breakpoint location. Below is a snippet of some of the tests:

```
String 1 ➜ "The game is looking for some data or a file..."
    • Searching for this in the DLL led us to LAB_004168B3.
    • There is an exact duplicate of this string in the main executable.
    • Patching JNZ LAB_10082432 => JMP LAB_10082432 under function FUN_10082675
      causes the app to crash when "Already Paid" button is pressed to access the launcher's sub-menu.
    • There is another address in the DLL, 1008267B, where a similar function block can be found.
    • Modifying this DID NOT seem to produce an effect.

String 2 ➜ "Arcade.dat"
    • There is one reference in the DLL at 10015C50.
    • Not much is revealed other than that the parent function, FUN_10015C0D, returns a 1-byte bool in AL.
    • There are several similar-looking function blocks for ra.dll, Application.dat, Channel.dat and so on.
    • Modding the FUN_10015C0D block to mimic expected flow (as with original Arcade.dat) DID NOT cause an effect.

String 3 ➜ "Arcade Main 1024"
    • Searching for this led to LAB_10006A5B which is a block under FUN_1000697E.
    • FUN_1000697E has one parent XREF => radll_EnterMenuSession.
    • From testing, we know that setting a breakpoint at radll_EnterMenuSession actually triggers
      when the button to access the launcher sub-menu is clicked.
    • This is true for both genuine and modified Arcade.dat files.
    • This is an indicator that the mechanism that does validation for Arcade.dat
      may be located somewhere in the radll_EnterMenuSession sub-routines.
```
Suffice to say, most of the leads discovered for testing at this phase led to dead ends. Some instruction modifications had no observable effect while others caused Puzzleball 3D to crash on pressing the ``Already Paid`` button.

After much trial and error, I discovered a sub-routine called ``FUN_1007F63F`` in the DLL, which through preliminary observation with WinDbg seemed to perform a "loader" type of functionality for components of Puzzleball 3D like the ``Application.dat``, ``Channel.dat``, and of course, ``Arcade.dat`` resource files that are all found in the root directory.

<img width="524" height="239" alt="arcadeloader" src="https://github.com/user-attachments/assets/b56e59ff-bb40-4994-8c7d-a59d84e9f62e" />

The above is only a snippet of the function's beginning section. ``FUN_1007F63F`` is massive with numerous branches of sub-routines that loop over each other, where even the decompiled view would likely take up multiple pages worth of space on this repo. 

Attempting to decode this function with the previous methodology of making comparisons between the expected and actual values in memory, registers, etc. and correlating that information with context from stepping through Puzzleball 3D's code with WinDbg, was not efficient.

I eventually understood that this function was performing an extremely long routine (hundreds of loops) of enumerating system details alongside loading almost every "piece" of each component (``Application.dat``, ``Channel.dat``, ``Arcade.dat``, etc.) that was used to draw the launcher menu's elements; everything from text box dimensions to button colors.

It was simply not feasible to trace each loop in hopes of discovering the chain that would lead to the typo. I deemed this approach another "failure" as performing such "grunt work" with no real certainty of the outcome would likely lead to endless frustration.
