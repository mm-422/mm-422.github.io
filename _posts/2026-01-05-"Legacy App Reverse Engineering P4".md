---
title: "Legacy App Reverse Engineering P4 - Deciphering The Story"
date: 2026-01-05 00:00:00 +/-TTTT
categories: [research, reverse engineering]
tags: [windows, reverse engineering, ghidra]     # TAG names should always be lowercase
image: "/assets/images/reverse_p4.png"
---
## The Story
In the previous part, we saw how methodology alone could become insufficient when put up against a complex routine with little to no context. What worked well for uncovering the earlier validation mechanisms, became quickly diminished in the face of a relatively opaque function that made even brute-forcing unfeasible.

We should remember that **no tool or technique** in Reverse Engineering is a one-size-fits-all type of solution. They are simply a means to an end, which is to piece together the original developer's intent behind the design of an application.

This "pieced-together intent" is what I like to call the ``Story``.<br>
Essentially, this is the expected flow of an application and/or its components from start to end. It is not just about **what** the application does, but **how** it does something.

Gaining a solid understanding of an application's ``Story`` will allow us to be more measured and accurate when analyzing binaries as well as reduce the likelihood of falling into "self-made traps" that might stem from bad assumptions. 

## The Activation Mechanism Story
For local applications like Puzzleball 3D that don't strictly require any server-side verification, the ``Story`` for its activation/validation mechanism might be as simple as gathering an input, applying some math or processing, and then producing an output based on the result.

Since all the resources needed for the proper and complete functionality of the application is contained within its binary and the system it is hosted or installed on, it should only be a matter of **time, effort, and most importantly understanding,** before every detail is unobfuscated and every "secret" unraveled.

## Gathering Historical Context
In order to flesh out the ``Story`` for Puzzleball 3D's activation mechanism, we first need to get into the mind of the original developer(s). This requires some research on period-relevant information, hence the "historical context".

### ♦️ Early Application Design
Back in the 2000s, the ``Story`` of a particular application usually revolved around a specific and procedural path, especially with regard to its activation/validation system.

At it's core, the system would begin with gathering an input, applying some logic and/or processing to that input, and then constructing an appropriate output based on the result.

For applications that relied on external modules for validation routines ― like a proprietary DLL in the case of Puzzleball 3D ― there would exist an extra "hand off" phase or "bridge" in between the input and the logic block.

This flow can be visualized as follows:
```
User Input ➜ Bridge ➜ DLL ➜ Logic/Processing ➜ Output
```
Essentially, the application's main executable would act like a **messenger**, collecting the user input ― like through a prompt, button click, and so on ― before passing it to the associated DLL with the help of a "bridge" function.

The DLL then runs that input through some processing where a "Judge" function would ultimately decide whether it is valid. Finally, the DLL will construct an output based on the result of the processing.

### ♦️ What is the Judge?
``The Judge`` is simply a routine that determines if an input is valid or matches an expected output.

Since the logic for Puzzleball 3D's activation mechanism is likely located in ``ra.dll``, and the launcher is custom-built, there must exist an intermediate function or "bridge", that helps pass the collected input ― which would be the unlock code in this case ― to the DLL in order for the ``Judge`` to run it through the "meat grinder" so to speak.

If we could locate the ``Judge`` and modify it to return a desired result **after** the processing ― like a 1 instead of 0 ― we could bypass the activation mechanism.

But before that, we should take a look at the custom-drawn UI for Puzzleball 3D to see how it relates to the ``Judge`` and how it might affect other parts of the application's design.

### ♦️ Custom Render Engines
It was not uncommon for applications in the early 2000s to implement their own "mini rendering engine" for constructing custom interfaces. This was done to make the apps stand out and bypasses the standard Windows controls in order to provide benefits like a potentially improved user experience. A great example of this is the popular media player (at least back then) called Winamp.

When we throw this "mini rendering engine" into the mix, the ``Story`` of an application's validation mechanism changes slightly.

Instead of the DLL simply constructing and reporting the output at the end, it will attempt to communicate with the function that is actually responsible for constructing the appropriate output based on an "internal ID" system.

Let's say the result after processing an unlock code is ``Unrecognized Code`` instead of ``Wrong Code`` ― both being different but essentially meaning "invalid code" ― which carries a specific internal ID of ``101``. This ID will then be passed from the DLL, through possibly another "bridge", over to the function responsible for constructing a message tied to that ID.

<img width="1280" height="720" alt="Unrecognized Code" src="https://github.com/user-attachments/assets/0f9be2cf-25a7-44e6-80b1-912e667e462b" />

A quick sift through the ``Arcade.dat`` file with Notepad again, revealed that most if not all of the text found on this part of Puzzleball 3D's launcher ― like the word "compter" and "Unrecognized Unlock Code" ― is pulled from that specific resource file.

There must exist a routine in either the main .EXE or ``ra.dll`` that will act as a "Librarian" to grab the correct error string to display based on the internal ID system mentioned previously.

### ♦️ What is the Librarian?
This is usually a function responsible for loading items/resources into memory and then constructing a "map" to keep track of them. It then retrieves the required item straight from a location in memory through the use of pointers or IDs.

Why load those items into memory instead of pulling them straight from the disk?
This is largely due to hard drives being prolific in the 2000s. This mainstream storage medium was much slower than system memory. If an application had to call the ``Librarian`` to fetch a specific resource each time a user interaction ― like a button click ― was performed, the UI would incur stutters and lead to poor user experience.

This is a critical bit of understanding that helps shed light on the flow of Puzzleball 3D's routines and components while aiding in making decisions like knowing when to search for data through the system memory instead of the local storage device.  

### ♦️ Tracking the Librarian
Uncovering the location of the ``Librarian`` could come in handy later when attempting to say, trace error messages up to the ``Judge`` through the use of pointers or IDs in memory.

Instead of setting a breakpoint on string references, we could consider placing them on calls to Windows APIs that allow the application to communicate with external data/resource files like ``Arcade.dat``.

Since error strings like ``Unrecognized Unlock Code`` are found in ``Arcade.dat`` ― which is an external file ― it would be more practical to set a breakpoint on API calls like ``CreateFile`` and ``ReadFile``, which are more than likely used to open ``Arcade.dat`` and read from it before pulling the required string.

We can trace our way to the ``Librarian`` by:
1. Loading Puzzleball 3D on WinDbg.
2. Navigating to the input field in the launcher's sub menu.
3. Setting a breakpoint through WinDbg with the command ➜ ``bp kernel32!ReadFile``
4. Entering a fake code to initiate the process of drawing the error dialog.
5. Looking at the call stack when the breakpoint hits.
6. Finding the parent class ➜ This is the ``Librarian``.

It is important to note that an application's ``Librarian`` is usually responsible for handling requests from multiple sources and is almost never tied to just loading a single file; otherwise the ``ReadFile`` API alone would be sufficient.

Below is what the call stack in WinDbg looked like after the breakpoint on ``ReadFile`` was hit:

<img width="697" height="317" alt="librarian" src="https://github.com/user-attachments/assets/370d4fe7-d608-4563-9fe8-d6c57529dfc6" />

It was still unclear which exact routine was the "core" ``Librarian`` and which ones were simply child functions. This is why I placed temporary labels on the image in order to use as a reference later on.

Since Puzzleball 3D relies on more than one external file ― like ``RAW_003.wdt``, ``Arcade.dat``, ``Channel.dat`` and so on ― the breakpoint was hit a total of 137 times subsequently. I had to correlate this data with observations in Procmon in order to narrow down the exact breakpoint that is tied to the loading routine for ``Arcade.dat``.

To do this, I set Procmon up with the following filters in addition to the defaults:

<img width="1027" height="757" alt="procmon filters" src="https://github.com/user-attachments/assets/50c5c570-6318-4c38-9a06-c7d7641d7095" />

I then "stepped the execution forward" in WinDbg while monitoring the Procmon window for events.

<img width="990" height="722" alt="procmon readfile" src="https://github.com/user-attachments/assets/5bc3337d-f5b1-4065-93e5-5d8141360904" />

Upon repeated tests with the same breakpoint location as above, ``Arcade.dat`` is confirmed to load on the fourth ``ReadFile`` operation each and every time. Checking the call stack in WinDbg now shows us the exact stack structure that we should investigate:

<img width="697" height="317" alt="readfile exact" src="https://github.com/user-attachments/assets/993644e0-97ed-4df4-bc7f-429cfaaffb87" />

Using that as context to perform "tracing work", I was able to establish the overall flow of the application ― from the first routine that runs after clicking the ``SUBMIT`` button, to the moment the error string is "read" from ``Arcade.dat`` with ``ReadFile`` ― as follows:
```
radll_Initialize
  ⤷FUN_1007f63f aka ArcadeLoader
    ⤷3rd call of FUN_10083b51 aka ArcadeLoaderCORE
      ⤷FUN_100837c2
        ⤷FUN_10084328
          ⤷FUN_10083f39
            ⤷FUN_1008008f (Wrapper. Multiple calls found)
              ⤷LAB_10080bb5 sub-routine under FUN_100809ee
                ⤷FUN_100a26de
                  ⤷FUN_100a270d
                    ⤷FUN_100a900a
                      ⤷LAB_100a90d5 sub-routine under FUN_100a906F
```
With this critical bit of context established, we can now go back to locating the ``Judge`` and move towards decoding and bypassing Puzzleball 3D's activation mechanism in the next part.

___

## Continued 1
> GOAL: Locate the Judge.

As a quick reminder, the ``Judge`` will be a routine in the program that:
- Receives user input.
- Performs some math or processing on that input.
- Determines if the input is valid or invalid and sets a flag based on the result.
- Hands over execution to a ``Librarian`` that will construct the appropriate dialog.

## Finding the Judge
There are 2 ways to locate the ``Judge`` based on all the testing performed up to this point:

### ♦️ Option 1
```
• Trace the error string in the DLL back up to the Librarian.
• Locate the internal ID for the error, if there is one.
• Follow this ID back to the function that calls for the "mini rendering engine".
• This function likely receives a command from the Judge to draw the error dialog.
• The Judge should be close in the call stack.
```

### ♦️ Option 2
```
• Load the program into WinDbg.
• Enter a fake unlock code into the launcher menu.
• Locate the code in memory and set a hardware breakpoint on read.
• Step forward in WinDbg.
• "Catch" the first function that accesses the unlock code.
• This function likely hands over the user input (code) for processing.
• The Judge should be close in the call stack.
```

With ``option 1``, there is a risk of getting tangled inside routines that are simply constructing individual elements on the launcher menu and have nothing to do with the activation mechanism.

With ``option 2``, we stand a higher chance of success as the ``Judge`` and "downstream" routines must be able to receive and read the user input first before it could determine if a key is valid or invalid.

The function that attempts to access or read the unlock code stored in memory, soon after the ``SUBMIT`` button is clicked which initiates the activation process, is very likely to be closely related to the ``Judge``.

### ♦️ Extra Consideration for Option 2
The second method actually has a prerequisite in that we need the application to be in a "frozen" state before we could sift through the memory for the user input. This means that we'll need to set a breakpoint somewhere in Puzzleball 3D's code which triggers as soon as the ``SUBMIT`` button is clicked and definitely before the ``Judge`` is able to get its hands on the unlock code.

This is where the earlier unit testing function, ``unittest_ValidateUnlockCode`` comes in.
In Part I, we confirmed that this function is never ran through or utilized by Puzzleball 3D during normal operation. But function blocks for unit testing often take a similar form to the "real" routine that is actively used.

<img width="1280" height="722" alt="the 2 functions" src="https://github.com/user-attachments/assets/c9265719-2080-4e0d-bb19-d4a47d498de3" />

We know that the functions, ``FUN_10018172`` and ``FUN_100180d1`` in ``unittest_ValidateUnlockCode``, likely perform some form of processing on an input in order to validate it. These are functions that serve a specific purpose and are likely only found in one other routine if any.

I searched for references to these functions and was able to find a parent routine in ``ra.dll`` called ``FUN_1000B555`` that contained calls to both of these functions.

<img width="1218" height="627" alt="1000b555 assembly" src="https://github.com/user-attachments/assets/74b1192a-1c82-46cf-bbb2-13a7f4ad8022" />

After setting a breakpoint at the beginning of ``FUN_1000B555``, I launched Puzzleball 3D again through WinDbg and attempted to go through the activation process. Fortunately, clicking on the ``SUBMIT`` button now causes the application to freeze, indicating that the breakpoint was indeed hit in WinDbg.

<img width="1259" height="720" alt="windbg bp" src="https://github.com/user-attachments/assets/5cd09570-293a-4272-a31d-16c413079e8b" />

### ♦️ Hypothesis
Now that we have a reliable breakpoint to use for "freezing" the application right at the start of what is likely the activation/validation routine, we can most likely work our way to the ``Judge`` with some help from WinDbg.

If we carry out the testing outlined in ``Option 2`` and find any overlap in function calls/references between this and ``FUN_1000B555``, then we have found the ``Judge`` or a routine very close to it.

## Option 2 Test with WinDbg
First, we set a breakpoint on ``FUN_1000B555``. This should cause the application to "freeze" when we click the ``SUBMIT`` button after entering an unlock code as demonstrated previously.

<img width="716" height="137" alt="CAKE input" src="https://github.com/user-attachments/assets/fb3889cc-4dfd-4049-aacc-4f0cb7015f3b" />

We then attempt to locate the entered user input by using the command ``s -a 0 1?80000000 "CAKE"``. This command basically searches the entire memory space for the ASCII string "CAKE", which is a word I decided to use to prevent the possibility of finding similar but unrelated strings.

<img width="615" height="219" alt="cake windbg" src="https://github.com/user-attachments/assets/24e4b9e0-db08-4347-891b-cce54937d66f" />

We then set hardware breakpoints on all the memory addresses found where the user input resides and/or has been copied to. In this case, it was addresses, ``0248b081``, ``051e66d1``, and ``0520aac1``.

After that, simply step forward in WinDbg to see which breakpoint gets hit and look at the call stack; paying special attention to the return address column.

<img width="528" height="423" alt="call stack overlap" src="https://github.com/user-attachments/assets/a6f82028-f87f-4d92-8fcc-2cfb79a48288" />

The return address for the item on top of the stack is ``1000B565`` which is actually located in ``FUN_1000B555``. This is the overlap we are looking for.

The likelihood of us being inside the right chain of routines is made even greater when we look at the second item on the call stack ➜ address ``10002915``.
This is located in ``FUN_1000286C``, the parent function of ``FUN_1000B555``, and is right where the ``TEST AL,AL`` instruction is.

<img width="933" height="137" alt="290c" src="https://github.com/user-attachments/assets/2c8ed547-cb98-4139-9553-e55a16b30bbe" />

The TEST instruction placed right after the call to ``FUN_1000B555`` indicates that the result from this operation might act as a "flag setter" for the application to decide if it should take the JZ path to ``LAB_10002969``.

That sub-routine could possibly lead to error type categorization and/or error dialog construction. To confirm this, we need to examine what the register values look like right before the TEST instruction.

This can be done by simply setting a breakpoint at the last line in ``FUN_1000B555`` which is the RET instruction, and then proceeding through the activation process again.

<img width="969" height="266" alt="registers" src="https://github.com/user-attachments/assets/e26c9800-54a4-460f-a476-923aa186e089" />

The above is a screenshot of the register values and flag status as displayed in WinDbg. We can see that the AL portion of EAX is filled with ``00``. This indicates there is an expected value for the AL register which will correlate with the validity of the unlock code supplied by the user.

A TEST instruction is essentially a bitwise AND operation that temporarily computes the result between 2 values and sets the Zero flag or ZF based on it. Since the AL register contains a zero value, the TEST instruction with the AL register against itself will return a zero and set ZF to 1.

This means that the application's flow takes the path towards the ``LAB_10002969`` sub-routine when supplied with an invalid unlock code.

### ♦️ The Judge
From the previous call stack, going further upstream from ``FUN_1000286C`` is not possible with static analysis alone. This is because there is only one XREF point for this function and it is a memory address. Any part of the program can point to this address dynamically during runtime.

<img width="639" height="180" alt="xref286c" src="https://github.com/user-attachments/assets/88c7a3a9-9c00-4ada-9bcc-cfd02d60a823" />

With this, we can ascertain that ``FUN_100286C`` is an isolated or "modularized" routine that is called upon by some part of the main application through a pointer in memory to execute specific operations like verifying unlock codes and initiating error dialog construction.

``FUN_1000286C`` is the ``Judge``, albeit this determination can be subjective. What matters most is the ultimate goal of bypassing the activation mechanism.

### ♦️ Hypothesis
Patching the assembly or modifying the register value to essentially flip the application's flow to skip the JZ instruction in ``LAB_1000290C`` might get us the desired result and bypass the activation.

## Bypassing Activation with WinDbg
We can set and clear flags in WinDbg by using the following command:
```
r <flag alias> = <0 to clear, 1 to set>
```
We can quickly test to see if this method would work for bypassing the activation mechanism by setting a breakpoint right at the ``TEST AL,AL`` instruction in ``LAB_1000290C``, going through the activation process in Puzzleball 3D's launcher menu, and then flipping the flag before stepping forward.

<img width="626" height="99" alt="zflag" src="https://github.com/user-attachments/assets/48d19567-a2b9-40f6-96bb-a658e7303a41" />

Success!

<img width="600" height="81" alt="success" src="https://github.com/user-attachments/assets/b2fecb14-f1fa-41e5-863f-8a06c21a2bea" />

A permanent patch is trivial to create at this point and simply involves modifying the assembly instructions within ``FUN_1000B555`` to ensure that the AL register contains a non-zero value (1) right before the TEST instruction in ``LAB_1000290C``.

___

## Continued 2
With the main goal achieved, I wanted to tie up loose ends by revisiting the "typo problem" to see if a refreshed approach could get me to a working solution this time. Just like with decoding the activation mechanism, I focused on gaining good understanding of the application's behavior with context from historical research first before applying any techniques and methodology.

> GOAL: Fix the typo in Arcade.dat

## General Anti-Tamper Mechanisms
In the early 2000s, developers utilized a few common tricks to ensure their data files weren't tampered with. Some of these are as follows:  
**The Checksum / CRC Trap**  
```
• App reads every byte in the data file.
• Adds them up or runs an algorithm like CRC32.
• Compares the result to a hardcoded value.
```

**The File Size Trap**
```
• App asks Windows, "How big is data file X?"
• If reported size =/= internally expected size,
  then app assumes the file is corrupt.
• Ex. Changing "compter" to "computer" in Arcade.dat
  required adding an extra byte.
```

**The Offset/Pointer Table Trap**
```
• Custom UI files often have a header at the start.
• This header states the starting point of elements like strings.
• Modifying the data file by adding an extra byte shifts the entire body forward.
• This causes a mismatch between the starting point in the header and the body.
```

### ♦️ Testing Checksum Logic
An easy test to determine the existence of a CRC check for external resource files like ``Arcade.dat`` is to modify the contents while keeping the **file size the exact same.**

We simply open ``Arcade.dat`` in a hex editor like HxD and perform a byte change while keeping the total number of characters the same.<br>
Example ➜ changing the word "compter" to "computr".<br>
If the app works as expected then we are likely not dealing with a checksum or CRC check.

Below is a snippet of documentation from the preliminary tests performed on ``Arcade.dat``.

<img width="814" height="241" alt="crc test" src="https://github.com/user-attachments/assets/cb0d0c9a-1106-498c-a3a9-21e42c7d198d" />

## Magic Bytes & Headers
Earlier in the project, I mentioned performing a quick edit to the "compter" typo in ``Arcade.dat`` using Notepad, a simple text editor. This was of course not ideal as corruption can occur from improperly modifying the contents of a file without first establishing its structure and format.

There are other ways to verify a file's type aside from examining the extension which can be arbitrarily set. With a more appropriate tool like HxD (hex editor), we can quickly view the first few bytes of resource files like ``Arcade.dat`` to uncover important clues.

<img width="1249" height="377" alt="magicbytes" src="https://github.com/user-attachments/assets/7be9d7b1-fd63-4371-958f-cbc95a316a7c" />

The first four bytes, ``50 4B 03 04`` are the magic bytes for a ZIP archive and this makes sense considering the type of application we are working with.

It was incredibly common for developers in the early 2000s to bundle resources and assets for video games into a standard ZIP file, and then simply rename the extension to something like ``.dat``, just like how Java uses ``.jar`` for example.

Below is the file structure of ``Arcade.dat`` as shown in the Windows command prompt after decompression.

<img width="412" height="420" alt="arcade files" src="https://github.com/user-attachments/assets/aea4b380-818c-4162-acc6-2190d4995193" />

### ♦️ The ZIP Archive Structure
Just like how a website's html code consists of sections like the header, body, and footer, the structure of a ZIP file has the following parts:  
**Local File Headers**
```
• Each file inside the ZIP archive starts with a header.
• These contain the filename and the compressed/uncompressed size.
• Easily located in HxD by searching the file's path in the decoded text view column.
```

**Payload**
```
• The actual data or "body" of the resource file.
• Contains items such as text and error strings.
• Usually not very readable through HxD alone.
```

**The Central Directory**
```
• This is located at the end of a ZIP file's binary.
• It is a "master map" that contains the exact position of every file in the archive.
• The app looks for files based on the byte offset listed here.
• This is more efficient than scanning through the entire archive for the desired file.
• Position is indicated by the byte sequence ``02 01 4b 50`` or PK\x01\x02 in ASCII.
```

**The EOCD**
```
• Stands for End of Central Directory.
• Located at the end of the Central Directory.
• Contains a byte offset that points to the start of the Central Directory.
• Indicated by the byte sequence ``50 4B 05 06``.
```

### ♦️ Why Arcade.dat Became Corrupted
Based on what we know about the typical structure of a ZIP archive, we can see how simply adding arbitrary bytes with a text editor like Notepad will cause the internal map to get corrupted.

This is because the process of adding bytes will shift the file header positions forward without updating them in the Central Directory. This mismatch means that the application will be unable to locate the files in the archive properly and either reports them as being corrupted or missing, hence the "Fatal Not Found" error seen in earlier tests.

This also tracks with why overwriting a byte instead of adding an extra one ― like changing the word "compter" to "computr" ― worked without issues. This is because the file header positions are preserved through this modification and the byte offsets in the Central Directory remain true.

### ♦️ Hypothesis
If we could locate the Central Directory of ``Arcade.dat`` and manually update the byte offsets to account for the extra byte, the application should be able to parse the ZIP file's structure and show us the modified or fixed word on the launcher sub-menu without issues.

## Testing with HxD
First, I located the misspelled word in ``Arcade.dat`` through HxD.

<img width="621" height="501" alt="compter in HxD" src="https://github.com/user-attachments/assets/e135af04-c136-4009-820a-dd4205417f2f" />

I then added the missing byte or vowel and then moved to locate the EOCD in order to start manually updating the byte offsets. However, I encountered a problem at this step.

<img width="623" height="123" alt="eocd missing" src="https://github.com/user-attachments/assets/1f399263-fd8b-4fd6-8251-1eaa5f45dec3" />

The expected sequence for the EOCD, ``50 4B 05 06``, is not present here. The closest set of bytes is ``52 45 05 06`` but this is not a standard indicator for any part of a ZIP file's structure.

How was the application able to parse through the ZIP archive without an EOCD?<br>
To uncover this bit of critical information, I relied on Procmon once again to observe the loading behavior for ``Arcade.dat``.

<img width="819" height="343" alt="procmon offset comparison" src="https://github.com/user-attachments/assets/39d2f8ae-929f-4501-bc0e-22db1525ec54" />

It should be noted that the sequence for the ReadFile operations were exactly the same from run to run when launching Puzzleball 3D with the original ``Arcade.dat`` file.

The first offset is at byte 508,748 which matches exactly with the Windows reported size for the original ``Arcade.dat``.

We then see the following read operations performed at specific offsets, beginning with byte 496,269. Over 130 read operations are performed on ``Arcade.dat`` before the application loads fully.

With the **modified** ``Arcade.dat`` however, we can already see some discrepancies. While the first ReadFile operation starts at the correct offset ― byte 508,749 which includes the extra character added to the word "compter" in order to form "computer" ― the following ReadFile operations begin "disintegrating" after byte 496,269.

This indicates that the application cannot parse ``Arcade.dat`` properly and this may be due to improper offsets. But without an EOCD, how does the app know to start at byte 496,269?

I then went back to the end of the ZIP archive's binary to compare the byte sequence found earlier to what an expected EOCD should be.

**Standard EOCD**  
```
• 50 4B 05 06
• stands for PK\x05\x06
• PK are the initials for Phil Katz, founder of the ZIP format.
```

**Byte Sequence in Arcade.dat**  
```
• 52 45 05 06
• stands for RE\x05\x06
• Very similar to standard EOCD but has different "initials".
```

Was the application looking for this different byte sequence instead? To confirm this, I searched through the main .EXE in Ghidra for any references to these byte sequences and found the function ``FUN_10083F39`` which is an extremely large routine containing CMP instructions with values like ``0x6054b50``, ``0x4034b50``, ``0x2014b50``, and ``0x6054552``.

<img width="729" height="290" alt="zip function" src="https://github.com/user-attachments/assets/f36abdf1-469b-4f17-b03b-4da2af5a0611" />

Considering that in 32-bit assembly, bytes are stored in reverse order, ``0x6054b50`` translates to ``50 4b 05 06``. This is "PK\x05\x06" in ASCII, which is the byte sequence for the EOCD or End of Central Directory section in a ZIP file's structure.

The standard EOCD byte sequence was not found in ``Arcade.dat``, probably because the dev replaced "PK" (initials for Phil Katz, founder of the ZIP format) with "RE" (likely referencing company name or a term like "Resource Editor").

Therefore, to allow the application to read the file properly, we need to revisit the ZIP binary in HxD and make extra modifications.

<img width="635" height="95" alt="eocd pointer" src="https://github.com/user-attachments/assets/2767d23f-7226-4802-bf3e-c421a8f19f9a" />

There is an extra set of bytes at the end which don't seem to match any standard indicator. Converting this to decimal gives us the value 496265 which aligns with the start of the header of the first file read by the main application. This obviously must be shifted forward by an extra byte in order to fix the starting pointer.

Changing this to ``8a 92 07 00`` which translates to 496266, and then patching the file now results in the app being able to call ReadFile successfully on all resources contained in the ``Arcade.dat`` archive.

We can now observe the fixed typo in the sub-menu.

<img width="655" height="155" alt="finalmsg" src="https://github.com/user-attachments/assets/6e36c6cd-21ff-45d3-9a5c-dba853981f1c" />

___

## Project Summary
```
Tools and techniques are only means to an end,
  Understanding the intent behind an application's design is the ultimate goal.
```
Multiple times throughout the course of this project, the importance of understanding how an application behaves through gathering historical context and correlating that information with data gained from careful testing, was clearly demonstrated.

Often times, RE projects start with a quick search for an "entry point", followed by simple hacks to flip a result or outcome, and then ending with a fully unlocked or unrestricted version of an application.

With Puzzleball 3D, a decades old program with legacy design, it was when I tried getting into the mind of the original developer, that the tests and assumptions stopped leading to dead ends and began improving drastically.

Instead of following a typical and linear process, I began thinking about how the original creators might have designed the application and its components and what possible considerations/trade offs they may have had to make.

In the case of the activation mechanism, it was when concepts like "The Judge" and "Librarian" were formed to give structure to the often-abstract assembly, that the routines began to seem a lot less overwhelming. While the solution in the end may have been a simple "bit flip", the path towards that final deciding instruction was anything but straight forward.

``Arcade.dat`` also presented its own set of curveballs. One of them being the integrity-check mechanisms lined with duplicated strings in both the main executable and the primary DLL file, ``ra.dll``. A simple hex edit in the end was also thwarted by a proprietary EOCD sequence, which was discovered with the help of a behavioral analysis tool, Procmon.

While today's tools are very advanced and do a lot of the heavy lifting, the mind of the original creator is the most valuable asset one could possess when reverse engineering an application.

This is of course not quite possible to obtain. So the next best thing is to reconstruct what thoughts and considerations may have gone in to the building of an application by careful use of tools and techniques to compile the crucial historical and behavioral data.
