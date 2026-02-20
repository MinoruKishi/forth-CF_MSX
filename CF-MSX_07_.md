## CF-MSX\_07\_「system.c Function Explanation」

# Me:

~~~
Before moving on to the next step in forth_cf_msx_demo.asm, I want to organize system.c. Could you tell me the following 1) and 2)?
1) A list explaining the roles of functions inside system.c
2) The internal structure of the main function and the processing flow
 

◎Functions inside system.c
void ttyMode(int isRaw)
int qKey()
int key()
void ms(cell sleepForMS)
cell timer()
void zType(const char* str)
void emit(const char ch)
cell fOpen(cell name, cell mode)
 
void fClose(cell fh) { fclose((FILE*)fh); }
cell fRead(cell buf, cell sz, cell fh)
cell fWrite(cell buf, cell sz, cell fh)
cell fSeek(cell fh, cell offset)
char tib[256];
void repl()
void boot(const char *fn)
 

◎main function
int main(int argc, char *argv[]) {
 
cfInit();
addLit(“argc”, (cell)argc);
strcpy(tib, “argX”);
for (int i=0; (i<argc) && (i<10); i++) {
tib[3] = ‘0’ + i;
addLit(tib, (cell)argv[i]);
 
}
boot((1<argc) ? argv[1] : 0);
while (1) { repl(); } 
return 0;
 
}

~~~

## ChatGPT:

Below, I will organize and explain the functions in the provided `system.c` and the `main()` function to clarify “their purpose” and “where they are used within the VM/REPL” (written under the premise that `system.c` serves as the “minimal OS layer” for the forth-dwc / cf family).

---

## **1) Roles of Functions Inside system.c (Explanation List)**

### **Terminal (Keyboard I/O & Raw Input) System**

* **`void ttyMode(int isRaw)`**

* Switches the terminal (TTY) between **raw/non-raw** modes.

* Setting it to raw disables line buffering, echo, etc., enabling **character-by-character** reading (suitable for games or immediate key input in REPL).

* Typical use case: setting it to raw when starting REPL and reverting it upon exit.

* **`int qKey()`**

* A function that checks “**Is a key being pressed?**” **without blocking** (query key).

* Implementation often uses mechanisms equivalent to `select()` or `kbhit`.

* Commonly used in `ms()` wait loops or non-blocking input processing.

* **`int key()`**

* Reads one character (typically **blocks**).

* Entry point directly connected to REPL input capture or Forth's `key` primitive.

* **`void emit (const char ch)`**

* Outputs one character (equivalent to Forth's `emit`).

* Can be used for terminal display or internally in `zType()`.

* **`void zType(const char* str)`**

* Outputs a string (a “C-side utility” equivalent to Forth's `type`).

* Implementation often takes the form `while(*str ) emit(*str++);`.

* Used for startup messages, error displays, REPL prompts, etc.

---

### **Time (Sleep/Timer) Functions**

* **`void ms(cell sleepForMS)`**

* Sleeps for the specified number of milliseconds (equivalent to Forth's `ms`).

* Implementation uses `usleep`/`nanosleep`/`Sleep` on Windows, etc.

* **`cell timer()`**

* Returns " current time“ or ”time elapsed since startup" (unit depends on implementation).

* Used in Forth to measure elapsed time with `timer` or as a seed for random numbers.

---

### **File I/O (Forth file primitives)**

* **`cell fOpen(cell name, cell mode)`**

* Opens a file (the return value is often a handle packed into `cell`, typically a `FILE*`).

* `name` is often assumed to be a C string pointer (after conversion from Forth string to C string).

* `mode` is a pointer like `“r” " w“ ‘a’ ”rb"`, or a mode number converted to a string internally, depending on the implementation.

* **`void fClose(cell fh)`**

* Closes the `FILE*` (as indicated, `fclose((FILE*)fh)`).

* **`cell fRead(cell buf, cell sz, cell fh)`**

* Reads `sz` bytes from `f h` into `buf`.

* The return value is typically the actual size read (the return value of `fread`).

* **`cell fWrite(cell buf, cell sz, cell fh)`**

* Writes `sz` bytes from `buf`. The return value is the size written.

* **`cell fSeek(cell fh, cell offset)`**

* Moves the file position in `fh` (equivalent to `fseek/fseeko`).

* Some minimal implementations fix the whence (start/current/end) based on `offset`, while others use three arguments to pass `whence`.

* As discussed in the 64-bit conversion of forth-dwc, OS/implementation differences in `fseek`'s offset type cause variations here. The goal is to make this a “buffer” that absorbs those differences.

---

### **Input Buffer (TIB) / REPL Startup System**

* **`char tib[256];`**

* A **Text Input Buffer**.

* Temporarily holds a line read by the REPL or strings injected by `boot()`.

* Its reuse for generating key names in `addLit(“argX”, argv[i])` is precisely for this “temporary string buffer” purpose.

* **`void repl()`**

* Function that executes **one cycle** (or one line) of the Read–Eval–Print Loop.

* Typically:

* Read input into `tib`

* Set `>in` to 0

* Run the interpreter (`interpret`)

* Output `ok`

* The “outer infinite loop” is usually handled by `main()`, while `repl()` handles “one iteration”.

* **`void boot(const char *fn)`**

* Startup processing.

* Typically:

* Initialize terminal mode (raw, etc.)

* If `fn` exists, read and evaluate that file (execute script)

* If not, search for equivalent default `boot.fth`, etc.

* Crucial for the design: “Forth's initial dictionary is created in C, then expanded by the startup script.”

---

## **2) Internal Structure and Execution Flow of the main Function**

We'll break down the provided `main()` in a way that makes its processing meaning clear.

### **Step 0: Initialize VM/Dictionary**

`cfInit();`

* Initialize the Forth VM core.

* Typically:

* Initialize stack/return stack

* Initialize dictionary pointers (HERE/LAST, etc.)

* Register primitives (Adds C-defined words to the dictionary)

* Initializes variables (`base`, `state`, `>in`, etc.)

### **Step 1: Make `argc` accessible from Forth**

`addLit(“argc”, (cell)argc);`

* Adds a " Name→Value“ literal (constant/variable entry) added to dictionary.

* In Forth, executing the word `argc` typically pushes `argc` onto the stack.

### **Step 2: Register `argv[i]` as `arg0` to `arg9` in the dictionary**

~~~
strcpy(tib, ”argX");
 
for (int i=0; (i<argc) && (i<10); i++) { 
tib[3] = ‘0’ + i; 
addLit(tib, (cell)argv[i]);
 
} 
~~~

* Use `tib` as a temporary buffer to create `“arg0” .. “arg9”`,

* and register `argv[i]` (string pointer) for each value.

* This way, in Forth, simply reading `arg1` 
“C string pointer for startup argument 1”.

* If Forth has a conversion word like `cstr>string`, it can then handle it.

### **Step 3: Execute Startup Script (Optional)**

`boot((1<argc) ? argv[1] : 0);`

* If there is at least one argument (= `argv [1]`), pass it to `boot()` as the “boot file name”.

* That is, when booting with `./cf hello.fth`, it is expected to first load and evaluate `hello.fth`.

* If no arguments are present, `boot(0)`.

* In this case, whether `boot()` “searches for a default file,” “does nothing,” or “only displays a startup message” depends on the implementation.

### **Step 4: Run the REPL Infinitely**

`while (1) { repl(); }`

* `repl()` processes “one interactive unit (one line or one command string)”, and when it returns, it moves on to the next one.

* If `repl()` is designed to internally swallow exceptions/errors and return `ok`, the interaction can continue indefinitely.

### **Step 5: Formal return**

`return 0;`

*   Since it's actually an infinite loop, this is rarely reached.

---

### **Supplement: Perspective Revealing the “Organization Points” of This Structure**

If you're organizing system.c, dividing it into sections by functionality makes it much easier to read.

*   Terminal Layer: `ttyMode/qKey/key/emit/zType`

* Time layer: `ms/timer`

* FS layer: `fOpen/fClose/fRead/fWrite/fSeek`

* REPL layer: `tib/repl/boot`

* main: `cfInit` and “argument → dictionary injection” and “boot → repl”

---

If needed, the next step could be **"system. c reordering proposal (with header comments)“** and, considering that `addLit(‘argX’, argv[i])` is a ”string pointer", we can also propose a design for **a Forth-friendly receiving mechanism (e.g., making it a length-specified string like `arg1$`)**.

## Me:
Please also provide a list explaining the roles of the functions appearing in cf.c.
 

~~~
static void push(cell x) 
static cell pop () 
static void rpush(cell x)
static cell rpop()
static void comma(cell n)
static int changeState(int newState)
static void checkWS(char c)
static int nextWord()
static DE_T *addWord(char *w)
static DE_T *findWord(const char *w)
static void compileNumber(cell n)
 
void addLit(char *name, cell val)
static void cfInner(cell pc)
static int isNumber(const char *w)
static void executeWord(DE_T *dp)
static void compileWord(DE_T *dp)
 
static int isStateChange() 
void cfOuter(const char *src) 
void cfInit() 
~~~
## ChatGPT:

Understood. The `cf.c` file contains the core components of Forth (stack/dictionary/interpreter/compiler), with functions broadly categorized into **(A) Stack**, **(B) Dictionary/Compiler**, and **(C) Outer Interpreter**, **(D) Inner Interpreter**. Below is the role list: --- ## **A. Data Stack / Dictionary / Compiler**, **(C) Outer Interpreter**, **(D) Inner Interpreter**. Below is the role list. --- ## **A. Data Stack / Dictionary / Compiler**, **(C) Outer Interpreter**, **(D) Inner Interpreter**. Below is the role list. --- ## **A. Data Stack / Dictionary / Compiler**, **(C) Outer Interpreter**, **(D) Inner Interpreter**. Below is the role list. --- ## **A. Data Stack / Dictionary / Compiler**, **(C) Outer Interpreter**, **(D) Inner Interpreter**. Below is the role list. --- ## **A. Data Stack / Dictionary / Compiler**, **
 Dictionary/Compiler**, **(C) Outer Interpreter**, and **(D) Inner Interpreter**. Below is a list of roles.

---

## **A. Data Stack / Return Stack**

* **`static void push(cell x)`**

* Pushes 1 cell onto the data stack.

* Forth operations are fundamentally pushed here and proceed.

* **`static cell pop()`**

* Pops 1 cell from the data stack.

* Retrieves arguments during operations or word execution.

* **`static void rpush (cell x)`**

* Pushes a cell onto the return stack.

* Used for DO..LOOP control values, return addresses, and managing inner interpreter returns.

* **`static cell rpop()`**

* Pops a cell from the return stack.

---

## **B. Dictionary and Compilation Support**

* **`static void comma(cell n)`**

* Function to “write 1 cell to the dictionary area (HERE) and advance HERE”.

* Similar to the C implementation of Forth's `,` (comma).

* Core for concatenating tokens/addresses/immediate values during compilation.

* **`static DE_T *addWord(char *w) `**

* Adds a new word (dictionary entry) to the dictionary and returns a `DE_T*` (dictionary entry).

* Responsible for saving the name, updating the link (`last`), and initializing the code field, etc.

* **`static DE_T *findWord(const char *w)`**

* Searches the dictionary for a word matching the name `w` (returns `DE_T*` if found, NULL otherwise).

* Essential for the outer interpreter to determine “Is the input token a word?”.

* **`static void compileNumber(cell n)`**

* Compiles the numeric literal `n` into the dictionary as “code to push onto the stack at runtime”.

* Typical: Place `LIT` (or a `doLit`-like primitive) and `n` side by side using `comma()`.

* **`void addLit(char *name, cell val)`**

* Utility to create a “word name” `name` and register it with the runtime effect “push `val`”.

* This is what `main()` used to create `argc` and `arg0..`.

---

## **C. Input Tokenization / Outer Interpreter Support**

* **`static void checkWS(char c)`**

* Whitespace detection (space/ tab/newline, etc.).* Supports `nextWord()`, which extracts words from input.* **`static int nextWord()`*** Retrieves the next “word” while referencing the input buffer (e.g., `tib`) and `>in`.* Often places the extracted word in a work area and returns success/failure (input end).* **`static int isNumber(const char *w) `*** Determines if `w` can be interpreted as a number (often checks `base` too).* The return value is “true if numeric, false otherwise” + the actual value is often stored in a separate variable/global.* **`static int isStateChange()`*** Helper function to determine if the last word read “changes the STATE?”.* Example: `:` switches to compile mode, `;` to interpreter mode, `[` `]` for temporary switching... etc.* Also relates to how “immediate words” are handled.* **`static int changeState(int newState)`*** Common processing to switch the `state` variable to `newState`.* Many implementations group ancillary processing here (e.g., starting header creation for `:`, appending `EXIT` and finalizing for `;`).* **`void cfOuter(const char *src)`*** Outer Interpreter.* Places `src` (input string) into `tib`, etc., and reads words using `nextWord()` →* If found by `findWord()`, “execute or compile”* If not found, numerically interprets with `isNumber()` ` to interpret numerically and “push or compileNumber”* If neither, error* “The core of Forth-like interactive processing”.---## **D. Execution (Inner Interpreter) and “Execute/Compile Branch”*** **`static void cfInner(cell pc)`*** Inner Interpreter.* Receives `pc` (equivalent to the program counter in threaded code), and sequentially fetches tokens from the token stream to execute the corresponding primitives/words.* Implementation varies between token-threaded / direct-threaded / subroutine-threaded, but its role is the “execution loop”.* **`static void executeWord(DE_T *dp)`*** Executes dictionary entry `dp` “immediately”.* If a primitive, calls the C function; if a colon definition, enters `cfInner()`, etc.* **`static void compileWord(DE_T *dp)`**
* “Compiles” dictionary entry `dp` into the dictionary.

* For normal words, insert a “call token” using `comma()`.

* For immediate words, execute `executeWord()` even during compilation—rules like this are handled here.

---

## **E. Initialization**

* **`void cfInit()`**

* Initializes the entire VM: stack initialization, dictionary area initialization, major variable initialization, primitive registration, etc.

* The “engine startup” called first by `main()` in `system.c`.

---

## **Summary (Reading Tips)**

* **`cfOuter()`** is “Reads and interprets text (where REPL or boot calls to)”

* **`cfInner()`** “Runs compiled code (the execution loop for colon definitions)”

* Connecting these is **`executeWord/compileWord/compileNumber/addWord/findWord`**.

---

If needed, next, map these to the **MSX-DOS .COM token-threaded VM** (forth\_cf\_msx\_demo.asm) you're currently building, you can organize:

* Where the `cfOuter` equivalent is

* Where the `cfInner` equivalent is

* `comma/compileNumber/compileWord` construct the “token stream”

into a **“C version cf.c → Z80 asm version VM”** comparison table.

# # Me: 

~~~
#define PRIMS(X) \ 
X(DUP, “dup”, t=TOS; push(t); ) \ 
X(SWAP, ‘swap’, t=TOS; TOS=NOS; NOS=t; ) \ 
X(DROP, “drop”, pop(); ) \ 
(omitted below)
~~~
 Please explain this again.
 

I understand that all WORDs written in C are defined using `#define PRIMS(X) \` and are utilized as follows depending on their purpose:
~~~
#define X1(op, name, code) op,
#define X2(op, name, code) NCASE op: code
#define X3(op, name, code) { op, name, 0 } , 
~~~
.
Are all CF WORDs written in C described by this PRIMS? Are `cf-boot.fth` and `disk.xf` loaded as external files and added to `PRIMS(X)`?

## ChatGPT:

Your understanding is largely correct. The key points are:

* * *`PRIMS(X)` is a “list of *primitives (lowest-level words)* implemented in C”**

* **`cf-boot.fth` and `disk.xf` are “data loaded at boot time that ‘adds words to the dictionary’”**

* * *External files are not “added” to `PRIMS(X)`.**

...is correct.---

## **1) Does `PRIMS(X)` define all “C primitives”?**

As you said, typically this single list defines

* `X1(op,name,code) op,` → **enum listing of opcodes**

* `X2(op,name,code) case op: code` → ** The switch execution body for the inner interpreter**

* `X3(op,name,code) { op, name, 0 },` → **The “opcode ↔ name” table (for dictionary initialization)**

are generated.

Therefore, the design is often \*\*“List all primitives you want exposed as C words in PRIMS”\*\*.

However, there are two important points to note.

### **Note A: Some C functions are not “words”**

Auxiliary functions like `push/pop/findWord`, even if written in C, **are not Forth words**.
Only those “registered in the dictionary and possessing a name” .

### **Caution B: Exceptions exist where words might be registered outside PRIMS**

For example:

* Directly calling `addWord()` to individually register “special words”

* Handling “state transitions” (`:` `;` `[` `]`) as special cases in the outer interpreter, not via PRIMS

Such implementations exist (your mentioned ` isStateChange()`/`changeState()` you mentioned smacks of this).

In conclusion:

* **“Primitives” implemented in C are usually centered around PRIMS**

* **But it's also possible for `:` etc. not to be in PRIMS (handled by special outer processing)**

* **“C functions” outside PRIMS often refer to internal components, not words**

---

## **2) Are `cf-boot.fth` and `disk.xf` added to PRIMS?**

**No, they are not added.** 
`PRIMS(X)` is a **compile-time fixed** “list of C primitives”.

External files like `cf-boot.fth` are loaded at startup via `boot()` → `cfOuter()`, etc.,

* **Colon definitions* for `:` to `;` *

* **High-level definitions** like `constant` / `variable`

* Additional loading via `include`

In this way, new entries (DE_T) are \*\*dynamically added to the dictionary*\*.

In other words, it is \*\*“adding to dictionary memory”, not “adding to PRIMS”\*\*.

---

## **3) So, are all “CF WORD” operations just PRIMS?**

**No.** (In most cases)

* PRIMS \= Minimal kernel (basics like `dup swap drop + -`)

* `cf-boot.fth` \= “Standard-like vocabulary” and convenience words built on top (`2dup`, `over`, `.` surroundings, control structure sugar, etc.)

* `disk.xf` \= Judging by the name, either a \*\*“external dictionary image/binary format”\*\* to expand the vocabulary further or **additional scripts** (this depends on your actual CF implementation)Therefore, the “total words available to the user” is typically**PRIMS (C primitives) \+ Forth definitions loaded at startup (.fth, etc.) \+ (if present) dictionary images (.xf, etc.)**combined.---## **4) What does `disk.xf` do?**I haven't examined your specific CF implementation, so I'll avoid making definitive statements, but there are two common patterns:1. **Dictionary storage image (fast startup)** A method that loads a pre-built dictionary instead of repeatedly interpreting and recompiling `boot.fth`.2. **A “script” of additional words** (extended Forth source)Loads a collection of extended vocabulary.In either case, it does not **add to PRIMS**, but rather **increases/restores dictionary space**.---If you could provide the first few dozen bytes of `disk.xf` (ASCII-like/binary-like) or the code within `boot()` that handles `disk.xf` (the part using fOpen/fRead), I can clearly determine and explain whether this is a dictionary image method or a Forth source method.## Me:`void cfInit()`'s `struct { char *nm; cell val; } nvp[]` ` in `void cfInit()` become FORTH word variables registered with `CF`? Given that it ends with `{ 0 ,0 }`, I suspect these are defined later and can grow.## ChatGPT:You've noticed a very good point 👍To conclude,\**The `struct { char *nm; cell val;` } nvp[]` within `cfInit()` is * *“FORTH ‘variable/constant words’ pre-registered at CF startup”.*However,**it is **not** a mechanism that automatically grows later.**Growth occurs **on the dictionary (DE_T) side**; `nvp[]` itself is a fixed table.Below, I'll organize this step by step.---## **1️⃣ What is `nvp[]`?**Typically, within `cfInit()`, you'll find an array like this (simplified):`static struct {` `char *nm;` `cell val;` `} nvp[] = {` `{ “base”, (cell) &base },` `{ “state”, (cell)&state },` `{ “>in”, (cell)&toIn },` `{ ‘here’, (cell)&here },` `{ “last”, (cell)&last },` `{ 0, 0 }` `};`
### **What is this?**

👉 \*\*This is the initial table for registering C-side variables (pointers) as “named words” accessible from Forth.

---

## **2️⃣ From CF/Forth's perspective, is this a “variable word”?**

Yes, **from Forth's perspective, it becomes a “variable word”.**

For example:

`base`

When executed,

* The word `base`

* **Pushes the address of the C variable `base`**

* **onto the stack**

This is the operation performed.

In other words, from a Forth perspective:

`base @ \ Read the current base`
`10 base ! \ Set the base to decimal`

becomes possible.

🔑 **Key Points**

* What `nvp[]` contains is not a “value” but **“an address (pointer)”**

* Therefore, `@` / `!` directly access the C-side variable

-- -

## **3️⃣ Why is `{0,0}` in `nvp[]` “added later”?**

This is a **point prone to misunderstanding**, but

`{0,0}` is
**not “added later,” but rather**
**a “termination marker to stop scanning in the C for loop.”**

.

Typical code looks like this:

`for (i = 0; nvp[i].nm; i++) {`
`addLit(nvp[i].nm, nvp[i].val);`
`}`

* Terminates when `nm == 0`

* A standard C idiom to **avoid holding the array size separately**

➡ **`nvp[]` itself is static and fixed**

---

## **4️⃣ So where do “later-increased FORTH variables” go?**

**They go into the dictionary (DE\_T).**

### **Example**

`variable foo`

When executed:

* `addWord(“foo”)` is called

* A **new DE\_T** is added to the dictionary

* Cell space is allocated in memory

This:

* Is unrelated to `nvp[]`

* Increases dynamically during execution

---

## **5️⃣ Relationship Between `nvp[]` and `addLit()`**

`cfInit()` typically looks like this:

`for (i=0; nvp[i].nm; i++) {` 
`addLit(nvp[i].nm, nvp[i].val);` 
`}`

The meaning of `addLit(name, val)` is:

**“Create a word that pushes `val` onto the stack when executed”**

Therefore:

| nvp entry　　　 | View from Forth |
| ----- | ----- |
| `{ “base”, &base }` | `base` → Address of C variable base |
| `{ “here”, &here }` | `here` → Address of dictionary pointer |
| | |
| | |
| | |
| | |

---

## **6️⃣ Big Picture (Important)

`┌────────┐`
`│ PRIMS(X) │ ← C primitives (dup, swap, +, …)` 
`└────────┘` 
`│` 
`┌────────┐` 
`│ nvp[] │ ← Exposes C variables to Forth (base, state, >in, here…)` 
`└────────┘` 
` │` 
`┌────────┐` 
`│ boot.fth │ ← High-level Forth vocabulary (: ; if do loop …)` 
` └────────┘` 
`│` 
`┌─────────┐` 
`│ user scripts │ ← User-defined` 
`└─────────┘`

---

## **7️⃣ Summary (Core)**

* ✅ Items written in `nvp[]` are **FORTH variables (or constants) registered at CF startup**

* ❌ `nvp[]` itself cannot be expanded later

* ✅ What can be expanded later is the **dictionary (DE\_T)**

* 🔑 `{0,0}` is the **array termination marker**

* 🔑 **Direct design link** between C global variables and Forth's `@ !`

---

Understanding this structure is 
**proof you grasp the core aspects** for reconstructing CF on Z80/MSX.

To proceed further,

* How to represent `nvp[]` * *how to express `nvp[]` in the MSX-DOS version of CF**

* **how to allocate `base/state/>in` in Z80-side RAM**

* **how to translate `addLit()` into LITs for a token-threaded VM**

## Me:

These areas are the natural next steps.

Then tell me `enum { STOP, LIT, J MP, JMPZ, NJMPZ, JMPNZ, NJMPNZ, PRIMS(X1) };`.

## ChatGPT:

Let me break down this line as **the CF execution model (token-threaded VM) itself**.

`enum { STOP, LIT, JMP, JMPZ, NJMPZ, JMPNZ, NJMPNZ, PRIMS(X1) };`

This defines **“a list of opcodes the CF virtual machine can understand”**. *

---

## **1️⃣ What does this define (1-line summary)**

**It's a number table for **“all instruction codes (VM instructions)”** that the CF inner interpreter (`cfInner()`) processes via `switch(op)`**


In Forth terms:

* **The true nature of the “tokens” that appear in thread code (compiled code)**

* **Numbers corresponding to things like `JP I_DUP` in the Z80 version**

---

## **2️⃣ The first half of the enum: “Special Instructions” for VM control**

`STOP, // 0`
`LIT, // 1`
 
`JMP, // 2` 
`JMPZ, // 3` 
`NJMPZ, // 4` 
`JMPNZ, // 5` 
`NJMPNZ, // 6`

### **🔹 STOP**

* **Terminate execution**

* Instruction to exit the `cfInner()` loop

* Used at the end of colon definitions or for error escape

For Z80 VM:

`I_STOP:` 
`ret`

---

### **🔹 LIT**

* **Push the next cell as an immediate value**

* Core of Forth's “numeric literals”

Compilation example:

`123`

→ Thread code:

`LIT 123`

---

### **🔹 JMP**

* **Unconditional jump**

* Foundation of control structures (IF/BEGIN/AGAIN, etc.)

`JMP addr`

---

### **🔹 JMPZ / JMPNZ**

* **Conditional jump**

* System that pops TOS for evaluation

| Instruction | Meaning |
| ----- | ----- |
| JMPZ | Jump if TOS \== 0 |
| JMPNZ | Jump if TOS \!= 0 |

---

### **🔹 NJMPZ / NJMPNZ**

* **Jump with “N” (Optimization-focused, often skip-pop type)**

* Implementation differences:

* No pop

* NOT condition

* Requires next-token

etc.

👉 **If making an MSX/Z80 version, these are commands you can initially ignore**
(Add later as optimization)

---

## **3️⃣ Later: The True Nature of `PRIMS(X1)`**

This is the **core of the X macro**.

### **Definition of X1 (Your Assumption)**

`#define X1(op, name, code) op,`

### **Expansion Result of PRIMS(X1) (Example)**

`enum {` 
`STOP,` 
`LIT,` 
`JMP,`
 
`JMPZ,` 
`NJMPZ,` 
`JMPNZ,` 
`NJMPNZ,`

`DUP,` 
`SWAP,` 
`DROP,` 
`ADD,` 
`SUB,` 
`...` 
`};`

👉 **The 
---

## **4️⃣ Why enum? (Design Intent)**

### **① Thread code becomes a “sequence of integers”**

The code listed in the dictionary is:

`[DUP][DUP][ADD][STOP]`

→ The actual implementation is:

`int code[] = { DUP, DUP, ADD, STOP };`

### **② Faster inner interpreter**

`switch (op) {`
`case DUP: ... break;`
`case ADD: ... break;`
`}`

* C compiler can generate jump tables

* Lighter than function pointers

👉 **The mainstream approach for token-threaded VMs**

---

## **5️⃣ Are PRIMS(X1) “all words written in C”?**

Here's a crucial clarification.

### **✅ What belongs in PRIMS(X)**

* **“Primitive words” implemented in C**

* e.g., `dup swap drop + - emit key`

* **Possess an opcode**

### **❌ What does NOT belong in PRIMS**

* **Syntax control words** like `:` `;` `[ ]` Syntax control**

* Dictionary manipulation words like `variable constant`, etc.

* These are:

* Specially handled by the outer interpreter

* Or defined in boot.fth

---

## **6️⃣ Does cf-boot.fth increase the enum?**

**No, it does not.**

`enum { … PRIMS(X1) };`

is **determined during C compilation**.

`cf-boot.fth` only:

* uses `:` to

* rearrange existing opcodes (DUP, JMP, LIT, etc.)

* **forming new words**

only.

👉 **New “vocabulary” increases, but no new “instructions” are added**

---

## **7️⃣ When replacing with the Z80 / MSX version**

This enum becomes directly:

`CF VM Instruction Set Specification`

.

For Z80:

| C enum | Z80 VM |
| ----- | ----- |
| LIT | I\_LIT |
| DUP | I\_DUP |
| JMP | I\_JMP |
| STOP | I\_EXIT |

What you're doing

8086 / C → Z80 macro conversion
Rebuilding the token-threaded VM

is a **fully compatible design**.

---

## **8️⃣ Summary (Core)**

* `enum { STOP, LIT, JMP, …, PRIMS(X1) };`

* \= **CF Virtual Machine Instruction Set**

* First half: Control instructions (for VM)

* Second half: PRIMS \= C primitive instructions

* External `.fth` files **only rearrange instructions, not add new ones**

* In Z80/MSX versions, they **become the opcode table directly**

---

Next logical steps:

* **Expanding `PRIMS(X2)`** in `switch(op)`

* **How `PRIMS(X3)` is used for dictionary initialization**

* **How `LIT / JMP / JMPZ` correspond to Forth's IF/DO/LOOP* *

Which one should we tackle first?

## Me:

Thank you for all the explanations. I'm still organizing it in my head. My current understanding is as follows. Is this correct? 
・In the core part of CF written in C, only the defined stack area and word variables have their contents added after startup.
 
・The core part of CF written in C is typically unchanged; only the external .fth files are modified as needed.

## ChatGPT:

That's a very good way to organize it.
To conclude, \*\*your understanding is “essentially correct”\*\*.
However, **one additional point will make it more accurate.**

Below, I'll explain by separating “✔ Correct points” and “⚠ Points that become more accurate with additional context”.

- --

## **①**

**In the core part of the CF written in C,**
**only the defined stack area and word variables**
**have their contents added after startup.**

### **✔ Essentially Correct**

The “core of CF written in C” provides:

* VM instruction set (enum / PRIMS)

* Stack (DS / RS)

* Dictionary area (HERE, LAST)

* Inner/outer interpreters

It provides only these,

**there are no “automatically growing C-side structures” after startup.**

### **⚠ Note (This is a deeper level of understanding)**

To be precise, **what grows after startup is:**

1. **Dictionary entries (DE_T)**

* `:` definitions via `.fth`

* `variable`, `constant`

2. **Data area (HERE advances)**

* Compiled token sequences

* Variable allocation areas

In other words,

“The stack area grows”

is less accurate than

** “the dictionary and its underlying memory grow”**is a more accurate description.🔑 The stack* does “expand and contract,” but* its “size or structure does not increase”Adding this point makes the explanation complete.---## **②****The core parts of CF written in C are typically unchanged,** ** only the external .fth files are modified as needed.**### **✔ This is completely correct (and at a philosophical level)**This is **the core of Forth/CF design philosophy**.* C side \= **immutable virtual machine*** `.fth` \= **mutable language/environment/application**This is the division of roles.### **Why is this design strong?*** The VM is **small and stable*** Language extensions are **handled by Forth itself*** During porting (Z80 / MSX / FPGA)* The C side (or ASM side) remains largely unchanged* `.fth` files can be reused👉 **The work you are currently doing****“mapping the C version of CF to Z80/MSX”****perfectly follows this philosophy.**---## **③ Organized in Diagram (Final Form)**`┌───────────────────────┐` `│ CF Core Written in C │ ← Typically Fixed` `│ │` `│ - VM Instructions (enum/PRIMS) │` `│ - Data/RS Stack │``│ - Dictionary Management │``│ - inner / outer │``└─────────┬─────────────┘``│``▼``┌──────────────────────┐``│ Things added at startup │``│ │` `│ - Variable words in nvp[] │` `│ - Definitions in boot.fth │` `│ - Included vocabulary │` `│ │` `│ → All added to dictionary │` `└─────────┬────────────┘` `│` `▼` `┌──────────────────────┐` `│ User-defined (.fth) │ ← Modified as needed` `│ │` `│ - New words │` `│ - Applications │` `│ - Experiments │` `└──────────── ─────────┘`---## **④ In a nutshell (your final understanding)****CF is** **“a system where Forth builds its own language and environment** **on top of a fixed VM written in C.”** **That's it.**This is already **the implementer's perspective**.---# # **Next Natural Questions (No Need to Answer Now)*** For the Z80/MSX version:**How much of the “C core equivalent” should be fixed?*** What is the minimal set of `PRIMS`?* How much can `boot.fth` be trimmed?* Where is the boundary between token-threaded and direct-threaded?Currently “under review,” so **Once this understanding clicks, we're already in the design phase.**No rush—this current review phase is very healthy.## Me: Looking at `system.c` and `cf.c` myself, I felt these files are a collection of temporary components shaping the FORTH system called `CF`. The true essence of `CF` only reveals itself when `main()` executes. Is this expression accurate? ## ChatGPT:Yes—That expression ** hits the nail on the head.**Moreover, it's not a “user perspective,” but **an implementer's understanding**.Let's refine it slightly for greater precision.---# **Your Expression**system.c and cf.c area collection of temporary components that form the FORTH system called CF,and the actual entity of CF only reveals itself when main() is executed.---# **Conclusion**✅ **Conceptually correct.**However,🔧 Rather than “temporary components,”👉 \*\*“static mechanisms”\*\* is even more precise.---# **Why this perception arises**## **1️⃣ Compile time**system.c / cf.c contain:* Instruction set (enum)* PRIMS* push/pop* inner interpreter* outer interpreter* dictionary management* stack areaare written.But at this point:* The dictionary is still empty* The stack is still empty* boot.fth hasn't been executed yet* There are no user-defined items👉 In other words, it's just the “engine” state.---## **2️⃣ The Moment main() is Called**`cfInit();``addLit(...);``boot(...);``while(1) repl();`Only now:* The dictionary is initialized* C primitives are registered* nvp\[\] is registered* boot.fth is loaded* colon definitions are generated* HERE advances* The actual language system is formed👉 **At this moment, “CF as Forth” finally boots up**---# **To go deeper**CF's essence is:`static code (C) + dynamic dictionary memory`combined.C code alone is still just “potential.”
Forth is:

a language that assembles itself at runtime

That's why you

felt it “reveals itself when main() runs”

.

That's an extremely accurate intuition.

---

# **Put another way**

system.c / cf.c is:

`a structure`

After main() executes:

`a living system`

Or, to put it more Forth-like:

`static VM + dynamic dictionary = Forth`

---

# **A deeper perspective (important)**

Actually,

CF's “body” isn't C code

either.

The body is:

`Dictionary memory (a chain of threaded code)`

.

The inner interpreter is merely an execution engine.

Even in the Z80 version you're building now,

* If C disappears

* And it becomes Z80

As long as there's “a dictionary and a token sequence”
That 
becomes the core of CF.

---

# **Summary**

Your understanding is:

✅ Conceptually correct
✅ Correct at the implementation level
✅ Aligns with Forth's philosophy

To be more precise:

CF is not a collection of C code, but rather
 
---

The view you currently see is not

the “perspective of someone using Forth”
\*\* but the “perspective of someone creating Forth”\*\*.

You're in a very good place.

---

If you like, an interesting question to consider next:

In the Z80/MSX version,
where is the “moment equivalent to main()”?

Finding the answer there
will complete another layer of your VM design.

## Me: 

I considered “Where is the ‘main() equivalent moment’ in the Z80/MSX version?”, but it feels like a rather difficult question.
If something tangible emerges from nothing tangible, it would truly become a compiler, but I don't think FORTH is like that. When CF starts up, it enters a state waiting for messages and input. From the inputted character string, either dictionary registration or execution occurs.
 
When I was looking into FIG_FORTH, I found it strange that during startup, it went through “Initial Setup” → “ABORT” → “QUIT” and ended. However, it turns out that inside “QUIT” is an infinite loop. Within that loop, “Key Input” → ‘INTERPRET’ executes the WORD names or numeric values entered from the keyboard. If an error occurs, after the error handling completes, “QUIT” is executed, and it returns to waiting for key input (as if nothing had happened). It's incredibly clean, but the actual behavior is hard to grasp—it leaves me with a strange feeling. 

## ChatGPT:

That “strange feeling” is precisely **the core of Forth's startup model** . The flow you described for FIG-FORTH (initial setup → ABORT → QUIT, though QUIT contains an infinite loop) perfectly aligns with the idea of \*\*“a minimal execution mechanism always present, with the language built upon it”\*\* rather than “something material emerging from nothing.”

In response to your question, for the Z80/MSX version, we transform the \*\*“main() equivalent moment”\*\* into a Forth-like “visible form.” --- ## **1) “main() equivalent moment” = “moment entering QUIT”** 
---

## **1) “The moment equivalent to main()” = Close to “the moment entering QUIT”**

In the C version of CF, main is:

1. `cfInit()` (The VM's foundation becomes operational)

2. `boot()` ` (loads startup vocabulary to grow the dictionary)

3. Runs `repl()` (waits for input → interprets → executes)

That was it.

In FIG-FORTH, this becomes:

1. Initial setup (stack, dictionary pointer, I/O, variables)

2. `ABORT` (prepares state, creates exception return point)

3. `QUIT` (**enters infinite-loop interactive world**)

.

In other words,

**In the Z80/MSX version, “the moment CF reveals itself” = the moment it first enters QUIT (or its equivalent outer loop) 
.

It feels less like “the compiler starts” and more like \*\*“an interactive loop that runs forever starts”\*\*.

---

## **2) The essence of “no entity → entity appears” is “the dictionary grows”**

Forth is less a compiler and more, simply put:

* **What resides permanently:**

* The stack

* The inner interpreter (execution mechanism)

* The outer interpreter (splits input into words + finds and executes/compiles)

* The dictionary (initially small)

* **What “grows” after startup: **

* Dictionary (colon definitions, variables, constants, convenience words)

This “growing dictionary” is what brings the \*\*“world of Forth”\*\* the user sees into existence.

So what you call

something appearing from nothing

is more accurately

**a minimal core (VM) exists, and the dictionary growing brings the entity (language environment) into being**

.

---

## **3) Why it appears to end with “ABORT → QUIT”**

This is Forth's beauty (i.e., its trick). The actual process is:

* `ABORT` is the \*\*“reset the environment to its initial state”\*\* word

* Empty the stack

* Sets `state` to the interpreter

* Sets `base` and `>in`

* Sets the **exception return point** (depending on implementation)

* `QUIT` is the \*\*“enter interactive loop”\*\* word

* Input

* Interpret

* OK

* Then input again...

So, when an error occurs:

1. Error handling (displaying messages, etc.)

2. Jump to “cleanup processing” equivalent to `ABORT`

3. **Return to `QUIT` again**

So, visually, it appears as:

Returning to input waiting as if nothing happened

.

This is designed so the “program never ends,” similar to an OS's main loop.

---

## **4) When ported to Z80/MSX, the “main equivalent” can be written like this**

Translated into your MSX-DOS `.COM` world, it roughly becomes these stages:

### **(A) Initialization (= cfInit equivalent)**

* Initialize SP/RS

* Initialize variables (base/state/\>in/here/last, etc.)

* Register primitive dictionary (PRIMS equivalent)

* Prepare screen/BDOS I/O

### **(B) Boot (= boot / cf-boot.fth equivalent)**

* If a boot script exists, load and interpret it

* For MSX-DOS, `OPEN/READ` and feed it to `cfOuter`

* Or use built-in “minimal boot” to enable only `:` and `;`

### **(C) QUIT Loop (= repl infinite loop)**

* Wait for input (BDOS single-character input or line input)

* Tokenize

* Dictionary lookup

* Execute or compile

* If error, return via ABORT

* Wait for input again

✅ This (C) represents the “first moment you entered” – the very moment you felt \*\*“CF revealed itself”\*\*.

---

## **5) The True Nature of “It Feels Clean but Hard to Grasp”**

The reason it's hard to grasp is simple:

* C and general languages: main → a single path of function calls

* Forth: \*\*Always in the QUIT world (infinite loop)\*\*, and from there

* Each time input arrives, the world changes a little (the dictionary grows)

* Errors don't “crash with exceptions” but “return to the world's entrance”

In other words, the program's ‘shape’ is

**not a “straight path” but a “ring”**

.

---

## **6) A super-short mnemonic for clear understanding**

* **CF's body \= Dictionary**

* **CF's heart \= Inner interpreter**

* **CF's face \= QUIT (outer loop)**

* **Error recovery \= ABORT → Return to QUIT**

- --

If you map the current structure of your `forth_cf_msx_demo.asm` (start → execution → end) to this FIG-FORTH-like

* INIT

* ABORT

* QUIT (infinite loop)

Let's map it to a diagram.
If you minimize the “inside of QUIT (input → INTERPRET)” on the Z80 side, the whole thing suddenly becomes graspable.

## Me:
I was wondering, “Where should I start to build the whole thing from scratch?” Could you show me the mapped diagram?

# # ChatGPT:

Certainly. Here, we'll map the **C version of CF (system.c/cf.c)** and your **Z80/MSX-DOS .COM version (forth\_cf\_msx\_demo.asm)** as a 

INIT → ABORT → QUIT (infinite loop)
(Within QUIT: KEY input → INTERPRET → Execute/Compile → ok → Further KEY)

as a **mapping diagram**.

---

## **Overall Mapping Diagram (C Version CF ⇔ Z80/MSX Version)**

### **C Version CF (main performs the “ceremony”)**

`main()` 
`├─ cfInit() … Initialize VM core (stack/dictionary/PRIMS registration)`
 
`├─ addLit(arg...) … Adds variable/constant words (argc, arg0..)`
`├─ boot(file?) … Loads startup script (e.g., boot.fth)`
`└─ while(1) repl() … REPL/QUIT loop (input → interpretation → execution → ok)`

## # **Z80/MSX-DOS Version (start performs “ceremony”)**

`ORG 0100h`
 
`JP start`

`start:`
`├─ INIT equivalent … Prepare SP/RS/variables/dictionary/HERE/LAST/PRIMS`
`├─ BOOT equivalent … (Optional) Read/embed startup script equivalent` 
`└─ QUIT equivalent(loop): … Infinite loop` 
`├─ KEY/ACCEPT … Input (1 character or 1 line)`
 
`├─ INTERPRET … Tokenize → Dictionary lookup → Execute/Compile` 
`├─ Display ok … Prompt` 
`└─ ABORT on error … Restore state and return to QUIT`

---

## **FIG-FORTH's “INIT→ABORT→QUIT” Decomposed by Functionality**

This is the key. Breaking down FIG-FORTH by “what it does” makes it easier to implement even on Z80.

`【INIT】 (Once only)`
`- Initialize data stack`
`- Initialize return stack`
`- Initialize variables (BASE, STATE, >IN, HERE, LAST…)`
`- Create initial dictionary state (register PRIMS/prepare header structure)`
`- Initialize I/O (BDOS/display/ key input)`

`↓`

`【ABORT】 (Called anytime, “return ritual”)`
`- Empty data stack`
`- Set STATE to interpreter`
`- Set >IN to 0`
`- Display error message if necessary`
`- (If there's an exception return point, go there)`

`↓`

`【QUIT】 (Infinite Loop)`
`loop:`
`- Get input (ACCEPT/KEY)`
`- INTERPRET (word → find → execute or compile)`
`- Output ok`
` - goto loop`

\*\*The “main equivalent moment” for the Z80/MSX version is the “first entry moment” into this QUIT loop\*\*.

---

## **Further concretization: Z80/MSX .COM “box diagram”**

Breaking down your `forth_cf_msx_demo.asm` into the following “boxes” reveals where to start building.

`┌────────────────────────────────┐`
`│ [A] start / INIT │ ← Equivalent to cfInit`
`│ - SP/RS setup │`
`│ - Initialize variable area │` 
`│ - Initialize dictionary (HERE/LAST) │` 
`│ - Register PRIMS table │` 
`│ - Display startup message │` 
`└───────────────┬────────────────┘` 
`│` 
`┌───────────────▼────────── ──────┐` 
`│ [B] ABORT (Resume Routine) │` 
`│ - Clear DS / STATE=0 │` 
`│ - >IN=0 / Display if error │` 
`│ - Jump to QUIT │` 
`└────────────────────── ──────────┘` 
`│` 
`┌───────────────▼────────────────┐` 
`│ [C] QUIT (Outer Loop) │ ← equivalent to repl` 
`│ loop: │`
 
`│ - Get input via ACCEPT/KEY │`
`│ - INTERPRET │ ← equivalent to cfOuter`
`│ - Display ok │`
`│ - JP loop │`
`└───────────────┬────────────────┘`
`│`
`┌───────────────▼──── ──────────────┐` 
`│ [D] INTERPRET (Outer Interpreter) │ ← cfOuter` 
`│ - nextWord (Word Extraction) │` 
`│ - findWord (Dictionary Lookup) │` 
`│ - Branch based on state: │` 
`│ * Execute: executeWord │`
 
`│ * compile: compileWord/LIT │`
`│ - If numeric, LIT or push │`
`└───────────────┬──────────────────┘`
`│`
`┌───────────────▼─────────── ──────┐` 
`│ [E] INNER (Inner Interpreter) │ ← cfInner` 
`│ - Execute token sequence (equivalent to switch) │` 
`│ - PRIMS (dup/swap/drop, etc.) │` 
`│ - Resume at STOP │` 
`└────────────────────────────────┘`

---

## **Implementation Order for “Where to Start?” (Order to Reduce Confusion)**

Instead of building the whole thing at once, creating the \*\*“minimum functional loop”\*\* first makes the mystery disappear.

### **Step 1: Build Only the QUIT Skeleton (Input → Echo)**

* Read one character with BDOS, output one character (echo)

* This gives you the **“wait and return” loop**
(= captures the feel of QUIT)

### **Step 2: Implement nextWord equivalent (split into words)**

* Pick up words separated by spaces (tib and \>in)

* No dictionary needed yet

### **Step 3: Minimal dictionary (just a few words) + findWord**

* Just a few words like `dup` `drop` `.`

* If found, just display “Identified”

### **Step 4: Connect inner (execute token)**

* Here, `PRIMS` and `cfInner` equivalents finally gain meaning

* The flow “dictionary entry → execution code” begins

### **Step 5: Add ABORT**

* Even on error, **always return to QUIT**

* This completes that experience of “returning as if nothing happened”

---

## **In a nutshell**

The ‘mystery’ you feel comes from

**not a “linear program,” but** 
**a “world that always returns to the entrance (QUIT)”**

.

And in the Z80/MSX version,

**creating the QUIT loop first makes the whole thing suddenly click**

.

---

If you're willing, please post the current label structure of your `forth_cf_msx_demo.asm` (just the main label names after start is fine).
 I'll return a \*\*custom diagram for you*\* directly mapping \[A\] to \[E\] in this diagram to the labels in your actual file.
 

## Me:

Before that, a question: I've researched and understood the “ABORT” → “QUIT” flow in FIG-FORTH. How is that part structured in CF? 

## ChatGPT:

Good question. Your understanding of FIG-FORTH (**ABORT → QUIT** meaning “return to input waiting as if nothing happened”) is implemented in CF with **almost the same philosophy**. However, rather than prominently featuring a “word named ABORT” like FIG-FORTH, CF ** C-side structure (how repl/cfOuter returns) to implement the equivalent of ABORT/QUIT**. 

Based solely on what's visible in the CF `main()` you shared, I'll summarize the correspondence (※ Since I don't have your actual `repl()` / `cfOuter()` code on hand, I'll explain using a typical CF-style structure here. If there are differences, please share the relevant function, and I can redraw it based on your actual CF implementation).

- --

## **Where is the “QUIT equivalent” in CF?**

In CF, **`while(1){ repl(); }`** is almost directly equivalent to \*\* QUIT (outer loop) \*\*.

* FIG-FORTH: `QUIT` \= infinite loop performing “input → INTERPRET → more input”

* CF: `main()`'s infinite loop \+ `repl()` handles this

Image:

`main:` 
`cfInit()` 
`boot(...)` 
`while(1) repl() ← This is the QUIT world (outer loop)`

Inside `repl()`, it generally does these things:

1. Puts input into `tib` (equivalent to ACCEPT)

2. Call `cfOuter(tib)` (equivalent to INTERPRET)

3. Output `ok`

4. Return (main's while loop calls repl again)

---

## **Where is the “ABORT equivalent” in CF?* *

In CF, the equivalent of ABORT typically appears not as a single function name, but in **two layers**.

### **Level 1: Lightweight ABORT (reset state and continue)**

`repl()` performs the following **before and after each input**:

* Reset the stack (empty it or recover from underflow)

* `state=0` (interpreter mode)

* `>in=0`

* Discard partial compilation (if necessary)

It performs this kind of “restoration process” and **returns to waiting for the next input**.

This is the same job as FIG-FORTH's ABORT does.

### **Level 2: Strong ABORT (Exception Escape)**

Depending on the CF implementation, when an error occurs,

* `setjmp/longjmp`

* or `cfOuter()` returns an error code and `repl()` proceeds to the recovery process

either way, it can have a structure that **immediately returns to the REPL from deep nesting (during execution)**.

This strong ABORT is closer to the “return method” of FIG-FORTH's “ABORT returning to QUIT”.

---

## **Mapping CF to FIG-FORTH's ABORT→QUIT**

Your CF, mapped by visible function names alone, looks like this:

### **FIG-FORTH**

`INIT → ABORT → QUIT(infinite)` 
`└─ ACCEPT → INTERPRET → …`

### **CF**

`cfInit → boot → main's while(1) { repl(); }` 
`└─ (ACCEPT) → cfOuter(tib) → …` 
`└─ If error (equivalent to ABORT recovery handling)`

In other words, for CF,

* **QUIT ≠ main's while loop + repl**

* **INTERPRET ≠ cfOuter**

* **ABORT ≠ error recovery from repl (or cfOuter)**

is the clearest way to understand it.

---

## ** Does CF also give that “return as if nothing happened” feeling?**

Yes. The reasons are the same.

* Errors are designed to “return to the interactive loop” rather than “terminate the process”

* Consequently, the user sees *the next input prompt (ok)* again

However, depending on the CF implementation, the “ceremonial feel of ABORT” may not be as prominent as in FIG-FORTH.

* `cfOuter()` displays the error and returns

* `repl()` outputs ok and proceeds to the next input

This often results in a form that **appears natural as C control structures**.

---

## **To confirm the following as “your CF implementation itself” 
Please paste the contents of one or both of the following (preferably both), even if brief:

* The contents of `void repl()`

* The contents of `void cfOuter(const char *src)` (especially error handling)

This will clarify:

* **Which variables CF's “ABORT equivalent” resets**

* whether error recovery uses **return type or longjmp type**

* how “compile-time errors” recover

can be finalized in a **one-to-one correspondence diagram** with FIG-FORTH's ABORT/QUIT.

[ Back to Top](README.md) 

[ Back to Top](README.md)