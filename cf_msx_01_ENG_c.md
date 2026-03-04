# GitHub MinoruKishi/forth-CF_MSX   
## A Beginner's Account of Modifying FORTH for MSX0 with ChatGPT's Help After First Reading CCurl/CF  
  
#  
   
  
# You:  
  
20260109_cf_test01.zip  
  
Zip Archive  
The attached file is a FORTH language system called cf. I will now develop a Z80-based FORTH language system. Please tell me the optimal development environment and source code for the Z80.  
  
# ChatGPT:  
  
Thinking time: 2m 0s  
  
Understood. The attached cf is a “compact Forth influenced by ColorForth/Tachyon, running on just three small C source files” (this approach is explicitly stated in the README). This “minimal kernel + stacking extensions via Forth source” approach is quite well-suited for the Z80 as well.   
 Here, assuming you're “developing a new Z80-based Forth,” we'll compile **recommended development environments (prioritizing practicality)* * and compile existing strong source code as a starting point.   
  
---   
  
## **1) Optimal Z80 Development Environment (This is the absolute must-have)**  
  
### **A. Cross-Development (Build on PC → Transfer to Emulator/Hardware)**   
  
* z88dk: Development kit for Z80/8080 series (supports mixed C+ASM, multiple targets). Its strength lies in providing a complete suite of Z80 tools.  
   
* SjASMPlus: A modern Z80 cross-assembler. Well-documented and works on Windows/Linux/macOS.  
* Emulators (especially useful for CP/M systems)  
* z80pack: A mature emulator for multiple 8080/Z80 systems. Easy to run, including CP/M environments.  
   
  
Why this setup is good  
When developing Forth, “dictionaries,” “threaded code,” and “I/O” are critical, requiring frequent debugging. Having an emulator that runs on a PC drastically increases development speed.  
  
---   
  
### **B. Develop on Actual Hardware (Recommended Entry-Level Hardware)**  
  
* RC2014: A standard Z80 DIY system, easy to operate via serial terminal.  
* Z80-MBC2: Supports “disk” operations via SD card and can run CP/M 2.2/3. Strong as a foundation for developing Forth.  
  
---   
  
## **2) Ready-to-Use “Starting Point Forth Source Code” Candidates**  
  
Listed in order of alignment with your cf direction (small core, easy-to-build VM/dictionary).  
  
### **Candidate 1: CamelForth/Z80 (ANSI-oriented, packed with classic strengths)**  
  
* Official CamelForth/Z80 (Brad Rodriguez)   
* Port optimized for RC2014 (example using z88dk for builds)  
   
  
Recommended points:  
It includes a complete set of “dictionary, interpreter, and compiler,” allowing you to learn the “standard approach” to Z80 Forth while modifying it.  
  
---   
  
### **Candidate 2: eForth Z80 (Top choice if you want to start with a minimal core)**    
* eForth's Z80 implementation (includes EFZ80.ASM)  
   
  
Recommended Points:  
Starts small, aligning with cf's “minimalist approach.” Easily build a working core first, then layer your own vocabulary and OPCODE design on top.  
  
---  
  
### **Candidate 3: Firth (ROM-resident oriented, easy to approach for embedded systems)**   
  
* A Forth implementation designed to run from Z80 ROM  
  
---  
  
## **3) Leveraging the fact that you “have cf” (the realistic route)**  
  
As stated in cf's README, cf strongly emphasizes the concept of **“OPCODE sequence” + ‘primitives’ + “dictionary”**. If you're porting to Z80, this sequence is the shortest path:   
  
1.  Decide the target (CP/M environment or bare metal (direct ROM/RAM access))  
*  CP/M environment: Easier I/O, easier debugging (tools like z80pack are useful)  
*  Bare metal: Ultimately becomes “your machine”. However, you're likely to get stuck on the initial I/O.  
2.  Build a minimal kernel in Z80 assembly  
   
* Data stack / Return stack  
* NEXT loop (threaded code execution)  
* Minimal I/O (KEY/EMIT equivalent)  
3. Extend with high-level language via Forth source (cf-boot.fth style)  
4. Once comfortable, optimize cf's “OPCODE/primitive design” for Z80  
   
  
At this point, CamelForth or eForth already have a “minimal kernel, dictionary structure, and interpreter,” so you can converge on “your cf-style design” faster than building everything from scratch.  
  
---   
  
## **Essential Initial Setup (Conclusion)**  
  
* Development Tools: z88dk + SjASMPlus  
* Initial Runtime Environment: z80pack (CP/M emulator)  
* Base Source:  
* Start Solidly: CamelForth/Z80  
* From Minimal Core: eForth Z80  
   
  
---  
If needed, examine the attached CF files (cf.c / system.c / cf-boot.fth etc. in the zip) here to understand:  
** “If we want the same structure for the Z80 version, which C functions/concepts correspond to which Z80 routines/tables?”** into a mapping table, then concretize the minimal kernel design (memory map, dictionary format, NEXT method, primitive table) all at once.   
  
# You:  
   
  
cf is an independent FORTH system evolved from forth-dwc. While cf currently supports only basic functionality, we plan to explore expanding it during development. The target environment for cf is the MSX0 with a Z80 CPU. Applications for cf are written in the FORTH language. In some cases, we may modify the C source code within cf. The core code will be written in Z80 assembler. The approach will be as follows:  
   
Development Tools: z88dk + SjASMPlus  
  
Execution Environment (Initial): MSX0 (or using an MSX emulator)  
  
Base Source: Assembler, C source code, and FORTH source code within the cf ZIP file   
  
# ChatGPT:   
  
Thinking time: 1m 48s   
  
That's good. That approach (writing Forth apps in cf's Forth / touching C if needed / core in Z80 ASM) aligns well with cf's design (“OPCODE sequence + primitive table”).   
  
Just one initial fact check: The attached ZIP does not contain the core Z80 assembler code (it's the “cf running on PC” suite: cf.c / system.c / cf-boot.fth / makefile, etc.). So the core for the MSX0 (Z80) version will need to be created from scratch (which is fine per the plan). .   
  
Below, tailored to your setup (z88dk + SjASMPlus, MSX0/emulator), I'll outline the development environment setup and the specific steps to port cf to MSX0 via the shortest route.  
  
---  
  
## **1) Execution Environment: MSX0 or openMSX (openMSX recommended initially)**   
  
* openMSX excels at console/debugging, making Z80 step execution, breakpoints, and memory/slot checks straightforward.  
   
* For editor + VSCode users, the DeZog (VSCode Z80 debugger) + openMSX combo is also popular.  
   
  
Conclusion: Starting with openMSX to develop the core and then moving to a real MSX0 hardware is the fastest approach (I/O and memory issues become immediately visible).  
  
---  
  
## **2) Decide Output Format: ROM Boot or MSX-DOS (COM)**  
  
Either works, but startup speed differs.  
  
### **A. ROM (Cartridge) Format**   
  
* z88dk has startup routines and ROM configuration for MSX ROMs, which can be helpful.  
* Generating ROM binaries is also standard practice with SjASMPlus.  
  
### **B. MSX-DOS .COM (Disk Boot)**  
  
* There's a lot of existing know-how, and testing on an emulator is often easier (run it from a .dsk).   
  
If unsure: Start with MSX-DOS .COM (easier debugging) → Convert to ROM if it freezes.  
  
---  
  
## **3) Creating the “Framework” for Porting cf to MSX0(Z80)**  
  
The core concept in cf's README is this:  
  
* “Program = OPCODE sequence”   
* “Interpret OPCODE as primitive number, LIT, or XT (CODE slot)”  
(This design makes it easy to implement as a VM/interpreter on Z80)  
  
Therefore, dividing the port into 3 layers makes progress cleaner.  
  
### **Layer 1: Z80 “Core” (VM Runtime) — This is your Z80 ASM responsibility**   
  
Building just this much first allows you to implement something equivalent to cf-boot.fth.  
  
* Data stack (decide cell width first: 16-bit or 32-bit)  
* Return stack  
* NEXT loop (OPCODE fetch → dispatch)  
* LIT processing  
   
* Minimal I/O (key/emit equivalent. BIOS/BDOS calls are sufficient initially)  
  
### **Layer 2: Primitive Table (Dispatch Targets)**  
  
Since cf.c primitives assume “sequential numbering,” even for the MSX version  
  
* Aligning “same number = same meaning” makes things easier later (Forth source compatibility is preserved)   
* First implement only the minimal set needed to pass cf-boot.fth → expand later  
   
  
### **Layer 3: High-Level (Forth) — This is where apps/extensions reside**   
  
* Turn cf-boot.fth into the “MSX boot”  
* Your application (your FORTH code) also goes here  
  
---  
  
## **4) Toolchain specifics: z88dk + SjASMPlus role division**  
  
Your approach is fine. Recommended division:   
  
* SjASMPlus: Core VM (Z80 ASM), ROM header/slot initialization, ultra-low-level I/O  
(SjASMPlus documentation is also fairly new and well-organized)  
* z88dk: Auxiliary C (if needed), linking, reuse/research of target-specific crt0  
You can also refer to examples of MSX target ROM/boot code.  
   
  
※ “Not using C” is also fine, but z88dk's templates and crt0 can be helpful hints for the **initial “proof-of-concept”**.  
  
---  
  
## **5) Repository Structure (Recommended)**  
  
The current ZIP is the “PC version cf”, so it's safe to keep it as-is while developing the MSX0 version in parallel.   
  
cf/  
  
host_pc/ # Existing ZIP's cf.c/system.c/cf-boot.fth (reference implementation running on PC)  
  
target_msx/  
  
asm/ # Z80 core (NEXT, stacks, prims at minimum)  
  
forth/ # MSX boot and application  
  
tools/ # Build scripts (detailed later)   
  
build/ # Output .COM / .ROM  
---  
  
## **6) First Milestones to Tackle (in order of least likely to cause issues)**  
  
1. Decide cell width (start with 16-bit recommended: faster on Z80. Extend to 32-bit if needed)  
2. Create the smallest .COM or ROM that runs the " key→emit echo" in openMSX   
3. Implement VM's NEXT + LIT + several primitives on top of that  
   
4. Create a method to load cf-boot.fth on the MSX side (initially bundled in a fixed area is OK)  
5. Subsequently, increase primitives and Forth vocabulary  
  
---   
  
## **7) Tasks I Can Materialize From Here Onward (Next Steps)**  
   
  
Based on the contents of your ZIP (cf.c/cf.h/cf-boot.fth/README),  
  
* Extract the minimal set of primitives first needed for the MSX version (those required to pass cf-boot.fth)  
* **Z80-side dispatch table (number table)** and  
 * Draft memory map for the minimal VM (dictionary, HERE, stack area, string area)   
  
For this purpose, the next step involves reading the “memory/dictionary/cell width” sections from the ZIP's cf-boot.fth and cf.h files and translating them into a minimal kernel design for MSX0 (since you stated “the kernel is Z80 ASM,” we can even generate Z80 routine templates).  
  
Proceed as is if necessary.   
  
# You:   
  
"6) The ‘First Milestone’ to tackle first (in order of least difficulty)   
  
1.  Determine the cell width (start with 16-bit recommended: faster on Z80. Expand to 32-bit if needed)  
  
2.  Create the smallest .COM or ROM that makes “key→emit echo” work in openMSX"  
Let's go with 16-bit cell width. Can you create sample files for each component needed to build the COM file?  
   
  
# ChatGPT:  
  
Thinking time: 47s  
  
Yes, I can. Here, assuming a cell width of 16 bits, I'll first create a minimal sample set that works as an MSX-DOS .COM file for “key input → direct output (echo)”.   
  
MSX-DOS .COM files are loaded and executed starting at 0x0100. BDOS calls are invoked with **CALL 0005h (function number in C register)**.  
  
---   
  
# **1) File Structure (Minimal)**  
  
msx_cf_skel/  
  
src/  
  
echo.asm  
  
Makefile (For Linux/macOS. For Windows, build.bat is also OK)  
  
build.bat (For Windows)  
  
README.md  
---  
   
  
# **2) src/echo.asm (SjASMPlus・MSX-DOS .COM)**  
  
* ORG 0x0100  
* Uses BDOS 0x06 (Direct Console I/O)  
* E=0xFF for key input (no echo)  
* E=character for console output  
* Terminates with ESC(0x1B) (BDOS 0x00)   
※ The basic way to call BDOS in MSX-DOS is CALL 0005h.  
   
  
; echo.asm - Minimal MSX-DOS .COM: key -> emit (echo)  
  
; Assemble: sjasmplus --raw=echo.com src/echo.asm  
  
; Run on MSX-DOS: A:>ECHO  
  
DEVICE ZXSPECTRUM48 ; (SjASMPlus feature. Optional)   
  
ORG 0100h  
  
BDOS EQU 0005h  
  
; BDOS functions (CP/M compatible in MSX-DOS)  
  
F_TERM0 EQU 00h ; terminate program  
  
F_DIRIO EQU 06h ; direct console I/O (E=FFh input, else output)   
  
start:   
  
main_loop:   
  
; --- read key (no echo): A = char, 0 if none (implementation dependent)   
  
LD C, F_DIRIO   
  
LD E, 0FFh   
  
CALL BDOS   
  
OR A   
  
JR Z, main_loop ; no key => poll again   
  
CP 1Bh ; ESC to exit   
  
JR Z, exit   
  
; --- emit char   
  
LD E, A   
  
LD C, F_DIRIO   
  
CALL BDOS   
  
JR main_loop   
  
exit:   
  
LD C, F_TERM0   
  
CALL BDOS  
  
Note: The behavior of BDOS 0x06 (input polling, etc.) can vary by environment. First verify “key → display” works with this, then switch to BIOS CHGET/CHPUT in the next step for stability. CHGET/CHPUT itself is a standard MSX BIOS routine.  
  
---   
  
# **3) build.bat (for Windows)**  
  
@echo off  
  
setlocal  
  
rem Assumes sjasmplus.exe is in PATH. Use full path if not.  
  
if not exist build mkdir build  
  
sjasmplus --raw=build\echo.com src\echo.asm   
  
if errorlevel 1 (    
echo Build failed.    
  
exit /b 1  )  
  
    
echo Built: build\echo.com    
  
endlocal    
---    
# **4) Makefile (Linux/macOS - Optional)**    
  
SJASMPLUS ?= sjasmplus    
   
  
all: build/echo.com  
  
build/echo.com: src/echo.asm  
  
@mkdir -p build  
  
$(SJASMPLUS) --raw=$@ $<  
  
clean:  
  
rm -rf build  
---  
  
# **5) README.md (How to Run)**   
  
# MSX-DOS COM minimal echo sample (SjASMPlus)  
   
  
## Build (Windows)  
  
- run `build.bat`  
  
- output: `build/echo.com`  
  
## Build (Linux/macOS)   
  
- `make`  
  
## Run  
  
- Copy `echo.com` to an MSX-DOS disk image (.dsk) or to your MSX0 storage  
  
- On MSX-DOS: `A:>ECHO`  
  
- Press keys -> characters appear  
  
- Press ESC to exit  
---  
   
  
# **Next (the “minimal skeleton” to connect to cf)**  
  
Once this echo.com works, the standard approach is to add the **16-bit cell stack (DSP/RSP)** and NEXT (dispatch), then implement only LIT and a few primitives.  
  
If you'd like, in the next turn,   
  
*   Layout proposal (memory map) for the 16-bit cell data stack/return stack  
*   Minimal form of the NEXT loop (Z80)  
*   Identification of the minimal set of primitives needed to run cf-boot.fth from cf  
  
 I'll continue providing concrete file templates there.   
  
# You:  
  
UnderstandingFORTH_FIG79_MEMO.zip  
  
Zip Archive  
Here, I'm attaching the 8086 source code written in FORTH-79, which was the first FORTH I learned.  
  
The first thing to do is rename each 8086 machine code instruction to a macro name. Fill each macro with its corresponding Z80 code.   
As a concrete example, change the machine language instructions in the following assembler code to corresponding macros.  
  
; ****FORTH_INNER_INTERPRETER****  
  
DPUSH: PUSH DX  
APUSH: PUSH AX  
NEXT: LODSW ; AX <- (IP)  
MOV BX,AX ; IP <- IP+2  
   
NEXT1: MOV DX,BX  
INC DX ; SET W  
JMP WORD PTR [BX] ; JUMP TO (IP)  
  
; ****FORTH_DICTIONARY****   
  
; LIT   
  
DB 83H   
DB ‘LI’   
DB ‘T’+80H   
DW 0 ; end of dictionary   
LIT: DW $+2   
LODSW   
JMP APUSH  
  
Based on this, for example, rewrite the part  
MOV BX,AX ; IP <- IP+2   
 can also be rewritten to call and execute a single macro. In that case, MOV is the macro name, and AX, BX correspond to the macro's variables. Since we're running this on a Z80, we'll write Z80-specific code. Keeping the macro definition in a separate file seems convenient for later use with other CPUs.  
   
While I personally understand register names like AX and BX better, I think using FORTH-style names like IP, SP, RP, and W for variables inside the macro is preferable.  
Page 180 of “Standard FORTH by Satoshi Inoue, Kyoritsu Publishing Co., Ltd.” provides examples for both Z80 and 6809, so I'll note them here.  
   
  
◎ For Z80  
IP = BC  
SP = SP  
RP = IX  
W = DE  
NEXT = (IY)  
  
NEXT: LD A,(BC)  
INC BC  
LD L,A  
LD A,(BC)  
INC BC  
LD H,A  
NEXT1: LD E,(HL)  
   
INC HL  
LD D,(HL)  
EX DE,HL  
JP (HL)  
  
DOCOL: DEC IX  
DEC IX  
LD (IX+0),C  
LD (IX+1),B  
INC DE  
LD C,E  
LD B,D  
JP (IY)  
   
  
SEMIS: DW $+2  
LD C,(IX+0)  
LD B,(IX+1)  
INC IX  
INC IX  
JP (IY)  
  
◎For 6809  
IP = Y  
SP = U  
RP = S  
W = X  
   
  
NEXT LDX ,Y++  
JMP [,X]  
  
DOCOL PSHS Y  
LEAY 2,X  
BRA NEXT  
  
SEMIS FDB *+2  
LDY ,S++  
BRA NEXT  
  
# ChatGPT:  
  
Thinking time: 1m 23s  
  
Yes, it can. First, prepare **“source code replacing 8086 instructions with ‘macro calls’”** and, in a separate file, **“definitions implementing those macros in Z80 code”**.   
 (Initially, focus only on the *INNER INTERPRETER and LIT* areas you provided. The remaining dictionary vocabulary can be expanded using the same method.)  
   
  
The key here is to **“write using the 8086 register names (AX/BX...) as-is”** while mapping them to Z80's Forth register assignments.  
  
---   
  
## **0) Z80 Register Assignments (Fixed per your notation method)**  
  
Aligned with your notation (Standard FORTH p.180):  
  
* IP = BC  
* SP = SP  
* RP = IX  
* W = DE  
* NEXT = (IY) (The convention of using IY as the “address to jump to NEXT”)  
   
  
Additionally, for common usage in Z80 implementations:  
  
* HL is used for temporary registers/XT retrieval (standard form for NEXT)   
  
---  
  
## **1) Example file with 8086 code replaced by “instructions = macros”**  
  
Example filename: src/inner_8086_macroified.asm  
  
This retains the 8086-style appearance, but the actual content consists solely of “macro calls”.  
  
; inner_8086_macroified.asm   
  
; Version with all 8086 instructions replaced by macro calls (pre-Z80 stage)   
  
INCLUDE “macros_iface.inc” ; Declaration of instruction macro “interface” (Z80 implementation separate)   
  
; ****FORTH_INNER_INTERPRETER****   
  
DPUSH: I_PUSH DX   
  
APUSH: I_PUSH AX   
  
NEXT: I_LODSW ; AX <- (IP) , IP <- IP+2   
  
I_MOV BX, AX ; BX <- AX (Here, “W candidate” is placed in BX by convention)  
  
NEXT1: I_MOV DX, BX  
  
I_INC DX ; SET W  
  
I_JMP_IND BX ; JMP WORD PTR [BX]  
  
; ****FORTH_DICTIONARY****   
  
; LIT   
  
DB 83H   
  
DB ‘LI’   
  
DB ‘T’+80H   
  
DW 0   
  
LIT: DW $+2   
  
I_LODSW   
  
I_JMP APUSH   
  
Point:   
  
* Individual instructions like PUSH / MOV / INC / JMP are all macros.  
* LODSW is specific to 8086, so this is also made into a macro called I_LODSW (implements “read 16-bit of (IP) → IP += 2” on Z80).  
  
---   
  
## **2) Example Macro Definition File (Z80 Implementation)**  
  
Example filename: src/macros_z80.inc (assumes SjASMPlus)  
  
Here, it takes “8086 names” like the above I_PUSH AX and outputs code that actually maps to Z80's IP/W/RP...   
  
### **2-1) First, map “8086 register names” to their Z80 counterparts**  
   
  
This is the crucial part. It's safest to proceed with this mapping initially:  
  
* AX … HL (TOS/working register. Also receives values from LIT into HL)  
* BX … You might want to make it HL or DE, but to avoid disrupting NEXT's standard convention, lean towards **“treating BX as HL”**  
   
* DX … HL (W) (as per your note)  
  
In other words, at this stage, we keep the “names” “AX/BX/DX” while operating primarily with HL/DE on the Z80.  
  
### **2-2) macros_z80.inc**  
  
; macros_z80.inc (SjASMPlus)  
  
; Implementing 8086 instruction macros in Z80 code   
  
; --- Forth register assignment (fixed) ---   
  
; IP = BC   
  
; RP = IX   
  
; W = DE   
  
; NEXT vector = IY (IY points to routine address or is used as JP (IY) target)   
  
;   
  
; Convention for 8086-like names:   
  
; AX -> HL   
  
; DX -> DE   
  
; BX -> HL (we treat BX as HL to keep NEXT idiom simple)   
  
; ========== helpers ==========   
  
MACRO I_JMP label   
  
JP label   
  
ENDM   
  
MACRO I_JMP_IND reg   
  
; 8086: JMP WORD PTR [reg]   
  
; Z80: JP (regpair) ※reg assumes HL (BX=HL)   
  
JP (reg)   
  
ENDM   
  
MACRO I_INC reg   
  
; 8086: INC DX etc.  
  
INC reg  
  
ENDM  
  
MACRO I_MOV dst, src  
  
; 8086: MOV DX,BX / MOV BX,AX etc.  
  
; Z80: Assumes 16-bit registers (HL/DE/BC/IX etc.)   
  
LD dst, src  
  
ENDM  
  
MACRO I_PUSH reg  
  
; 8086: PUSH AX/DX  
  
; Z80: PUSH HL/DE  
  
PUSH reg  
  
ENDM  
  
MACRO I_POP reg  
  
POP reg   
  
ENDM  
  
; 8086: LODSW (AX <- (IP), IP <- IP+2)  
  
; Z80: Read 16-bit from (IP=BC) into HL (=AX). Increment BC by 2  
  
MACRO I_LODSW   
  
LD A,(BC)  
  
INC BC  
  
LD L,A  
  
LD A,(BC)  
  
INC BC  
  
LD H,A  
  
ENDM  
---  
   
  
## **3) At this point, it's in the “form you want”**  
  
* inner_8086_macroified.asm has “all instructions as macros”  
* Replacing macros_z80.inc makes it easy to port the same “instruction sequence” to another CPU  
* And, as you said, since only the instruction macro definitions are in a separate file, CPU differences are confined there  
   
  
---  
## **4) Recommended next steps (aligning with your notes on NEXT/DOCOL/SEMIS)**  
  
Current I_LODSW and I_JMP_IND are “instruction-level” macros.  
However, for Forth's performance and clarity, ultimately  
   
  
* NEXT (the standard Z80 NEXT)  
* DOCOL  
* SEMIS  
  
should be prepared separately as **“word macros” rather than instruction macros**.  
  
The Z80 version of NEXT in your memo is a prime example of this “word macro” approach.   
  
---  
  
## **5) It is also possible to “automatically” replace the 8086 source in the attached ZIP using this method**  
  
The 4TH_dos.txt file in the UnderstandingFORTH_FIG79_MEMO.zip you attached contains precisely these DPUSH/APUSH/NEXT/LIT instructions.   
Therefore, next,  
  
* I can read the relevant section of 4TH_dos.txt  
* Generate the “continuation (EXEC/BRANCH/0BRANCH...)” by replacing instructions with I_macros  
  
All the way through.  
  
---  
  
If proceeding further, which approach is preferable?   
  
1.  First build this sample with openMSX + MSX-DOS(COM) and verify operation (create a minimal boot framework)  
2. First, batch macro-ize the dictionary vocabulary (EXEC/BRANCH/0BRANCH…) in 4TH_dos.txt and fill in the Z80-side implementation.   
  
(Either way, I can provide a “ready-to-use file set” here.)  
   
  
# You:  
  
I ran sjasmplus in the following directory, but it resulted in an error. What could have gone wrong?  
  
src ---- --test_echo.asm  
build -- --  
include --MACROS_Z80.I  
  
$ sjasmplus --inc --inc=include --raw=test_echo.com src/test_echo.asm   
SjASMPlus Z80 Cross-Assembler v1.21.1 (https://github.com/z00m128/sjasmplus)  
   
Pass 1 complete (0 errors)  
Pass 2 complete (0 errors)  
test_echo.asm(6): error: Unrecognized instruction: MACROS_Z80.I   
Pass 3 complete   
Errors: 1, warnings: 0, compiled: 38 lines, work time: 0.001 seconds  
   
  
# ChatGPT:  
  
The cause is almost certainly this:  
  
The syntax (or extension) of `INCLUDE “MACROS_Z80.I”` is not being interpreted as an INCLUDE statement by SjASMPlus, so the string `MACROS_Z80.I` on line 6 is being read as an “instruction”.   
  
The log message  
  
Unrecognized instruction: MACROS_Z80.I  
  
appears when SjASMPlus determines that the line contains a “command-like” string (MACROS_Z80.I) instead of an INCLUDE ... statement.  
  
---  
  
## **Fix (most reliable)**  
  
### ** ✅ 1)** Change line 6 of **test_echo.asm** to this:**   
  
SjASMPlus requires `INCLUDE` (just writing `include` is insufficient).  
   
  
INCLUDE “MACROS_Z80.I”  
  
* Do not place labels or other elements at the beginning of the line  
* Always write `INCLUDE`  
* Enclose filenames in quotation marks  
   
  
---  
  
## **✅ 2) Also recommended: “Change the extension to **.inc**”**  
  
While MACROS_Z80.I looks like a header file, .inc is standard for SjASMPlus and reduces errors.  
  
Rename it to   
include/MACROS_Z80.inc,  
   
  
INCLUDE “MACROS_Z80.inc”  
  
is safer.  
  
---  
  
## **✅ 3)** Also clarify the --inc option usage (your command is close to correct)  
  
Your  
  
sjasmplus --inc --inc=include --raw=test_echo.com src/test_echo.asm   
  
means “add the include directory to the include path,” so the direction is correct.  
  
However, the interpretation of --inc can vary slightly depending on the SjASMPlus version or build. A more reliable way to write it is this (both are OK):  
  
sjasmplus --inc=include --raw=test_echo.com src/test_echo.asm  
---  
   
  
## **Other Causes of the Same Error (Just in Case)**  
  
The following scenarios also produce the same symptoms:  
  
* A semicolon (;) precedes INCLUDE, making it a comment  
* ; INCLUDE “MACROS_Z80.I” ← This won't be loaded  
* MACROS_Z80.I ← The next line is treated as an instruction and causes an error  
* Writing INCLUDE as INCLUDES or #include (doesn't work in SjASMPlus)   
* Full-width spaces are mixed in, preventing recognition as INCLUDE (rare)  
   
  
--- ## **Minimal working example (around line 6)**Try starting src/test_echo.asm like this:ORG 0100hINCLUDE “MACROS_Z80.I”BDOS EQU 0005h ; ...---If you paste just the first 10 lines of src/test_echo.asm, I can pinpoint exactly where it's not being recognized as an `INCLUDE` and return a fixed version.[(Next)](./CF-MSX_09_a.md)[(Back to README.md)](../README.md)   
