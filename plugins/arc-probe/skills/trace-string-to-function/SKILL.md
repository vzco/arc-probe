---
name: trace-string-to-function
description: Find a string in memory, trace xrefs to the functions that use it, analyze and label them — the "IDA string search to xref to function" workflow automated
argument-hint: <text> [module]
---

# /probe:string-trace

Given a string the user knows exists in the target process (error message, log text, UI label, config key, etc.), find it in memory, trace cross-references to the code that uses it, analyze those functions, and label everything in the GUI.

This automates the classic reverse engineering workflow: **string search --> xref --> function --> analysis**.

## Arguments

- `text` (required): The string to search for (or a substring of it)
- `module` (optional): Module to search in (narrows the search, faster)

## Why this works

Nearly every function references at least one string — error messages, log output, format strings, assert text, UI labels, config keys, SQL queries, file paths, protocol messages. These strings live in the `.rdata` section (read-only data) and code references them via `LEA reg, [RIP+offset]`. Finding that LEA instruction puts you directly inside the function that uses the string.

## Steps

### 1. Search for the string

```
probe.exe "strings find <text> <module>"
```

If no module is specified, omit it to search all loaded modules.

The response contains matches with:
- `address` — where the string lives in memory
- `section` — which PE section (should be `.rdata`)
- `module` — which module contains it
- `text` — the full string (may be longer than your search term)

If **multiple matches** are found, pick the one in the most likely module. For games, `client.dll` or the main `.exe` are common. For other apps, look for the module that matches the feature you're investigating.

If **no matches** are found, see "If the string isn't found" below.

### 2. Find code references (xrefs) to the string

For each string match:

```
probe.exe "strings xref <string_address> <module>"
```

This scans the module's `.text` section for `LEA` instructions that load the string address via RIP-relative addressing. Each result gives you:
- `address` — the code address of the LEA instruction
- `instruction` — the disassembled LEA (e.g., `lea rcx, [rip+0x15D421C]`)
- `function` — the enclosing function (from .pdata, if available)

If **multiple xrefs** are found, each one is a different code location that uses this string. They may be in different functions or in the same function at different points.

If **zero xrefs** are found, see "If xref returns zero results" below.

### 3. Analyze each referencing function

For each xref, find and disassemble the enclosing function:

**a. Find the function start.** If the xref response includes a `function` field, use that. Otherwise, scan backwards:

```
probe.exe "disasm <xref_address - 0x60> 30"
```

Look for the function prologue — the first instruction after `ret` or `int3` padding:
```asm
push rbp           ; 55
sub rsp, 0x??      ; 48 83 EC ??
```
or
```asm
mov [rsp+8], rbx   ; 48 89 5C 24 08
push rdi           ; 57
sub rsp, 0x??      ; 48 83 EC ??
```

**b. Disassemble the full function:**

```
probe.exe "disasm func <function_start>"
```

**c. Understand how the string is used.** Look at the instructions around the LEA:

- `lea rcx, [rip+...]` then `call <func>` — the string is passed as the first argument to another function. Common patterns:
  - Passed to a **log function**: `lea rcx, "Error: %s"` + `call LogMessage`
  - Passed to an **assert macro**: `lea rcx, "condition failed"` + `call AssertFailed`
  - Passed to a **UI function**: `lea rcx, "Button Text"` + `call SetLabel`

- `lea rdx, [rip+...]` — string is the second argument (first arg is in RCX, probably `this` or a format string)

- Multiple LEAs in sequence — the function uses multiple strings (format string + arguments)

**d. Name the function** based on the string context:
- Error strings: name after the error condition (e.g., `ValidateEntity` for `"Invalid entity ID"`)
- Log strings: name after the log topic (e.g., `InitNetworking` for `"[NET] Connecting to %s"`)
- UI strings: name after the UI element (e.g., `DrawHealthBar` for `"HP: %d / %d"`)
- Assert strings: name after the asserted condition

### 4. Find callers of each function

For the most important functions found in step 3:

```
probe.exe "xref scan <function_start> <module> --type CALL --limit 10"
```

This reveals who calls this function — essential for understanding its role in the system. Each caller is another function worth noting.

### 5. Generate signatures

For each key function:

```
probe.exe "sig <function_start> 32 <module>"
```

Test uniqueness:

```
probe.exe "pattern <signature> <module>"
```

Record the signature so the function can be found again after binary updates.

### 6. Label everything in the GUI

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"activity","status":"working","message":"Labeling string trace results..."},
    {"action":"store","store":"label","method":"setLabel","args":["0x<string_addr>","str: \"<text>\""]},
    {"action":"store","store":"label","method":"setLabel","args":["0x<func1_addr>","<func1_name> (refs: \"<text>\")"]},
    {"action":"store","store":"label","method":"setLabel","args":["0x<func2_addr>","<func2_name> (refs: \"<text>\")"]},
    {"action":"store","store":"label","method":"setLabel","args":["0x<caller1_addr>","calls <func1_name>"]},
    {"action":"navigate","tab":"disasm","address":"0x<func1_addr>"},
    {"action":"activity","status":"idle","message":"Done — found N functions referencing string"}
  ]
}'
```

### 7. Report

```
String Trace: "Failed to initialize entity"
============================================

String found at:
  0x7FFB22EA4098 (client.dll + 0x1914098, .rdata)
  Full text: "Failed to initialize entity %s (id=%d)"

Code References (3 xrefs):
  1. 0x7FFB2194FE75 (client.dll + 0x3BFE75)
     Function: 0x7FFB2194FE40 — CEntitySystem::InitEntity
     lea rcx, [rip+0x15D421C]  ; format string for error log
     Signature: 48 89 5C 24 08 57 48 83 EC 20 ...
     Callers: GameWorld::SpawnEntity (0x7FFB21A01200)

  2. 0x7FFB21EBE2D0 (client.dll + 0x92E2D0)
     Function: 0x7FFB21EBE280 — EntityValidator::Check
     lea rdx, [rip+0x1585DC1]  ; passed as error description
     Signature: 40 53 48 83 EC 20 48 8B D9 ...
     Callers: EntityFactory::Create (0x7FFB21F01000)

  3. 0x7FFB22103400 (client.dll + 0xB53400)
     Function: 0x7FFB22103380 — DebugOverlay::ShowError
     lea rcx, [rip+0x0E90C91]  ; displayed on screen
     Signature: 48 89 74 24 10 57 48 83 EC 30 ...
     Callers: (no direct CALL xrefs — likely called via function pointer)
```

## If the string isn't found

1. **Try a shorter substring** — the full string may have format specifiers, prefixes, or suffixes you don't know about. Search for the most distinctive phrase:
   ```
   probe.exe "strings find damage <module>"
   ```

2. **Try case variations** — `strings find` is case-sensitive. Try lowercase, UPPERCASE, or Title Case.

3. **Try without module filter** — the string might be in a different module than expected:
   ```
   probe.exe "strings find <text>"
   ```

4. **The string might be UTF-16 (wide)** — Windows APIs often use wide strings. The `strings find` command should handle this, but if not, try searching for the UTF-16 encoding manually.

5. **The string might be encrypted/obfuscated** — some software encrypts strings at rest and decrypts them at runtime. Signs:
   - No readable strings in .rdata at all (or very few)
   - Lots of XOR/ROT operations in function prologues
   - String-like data in .data section (writable, decrypted at runtime)

   In this case, the string won't be findable by pattern. Alternative: use a hardware breakpoint on a known data location and work backwards.

6. **The string might be loaded from a file** — config files, localization files, resource files. Check if the process loads external data.

## If xref returns zero results

1. **The string may be referenced via a global pointer** instead of RIP-relative LEA. Some compilers store string addresses in a global table:
   - Find the string address as a little-endian 8-byte value
   - Pattern scan for those bytes in .rdata
   - This catches indirect references through pointer tables

2. **The string may be in .data (writable)** — runtime-constructed strings or decrypted strings aren't referenced by LEA from .text. The reference is a pointer in .data that gets set during initialization.

3. **The LEA might use an unusual encoding** — `strings xref` scans for common LEA patterns but might miss REX.R or non-standard register encodings. Try a broader xref:
   ```
   probe.exe "xref scan <string_address> <module> --type LEA,MOV --limit 20"
   ```

4. **The reference might be split** — the compiler might load the address in two steps (rare on x64, more common on 32-bit):
   ```asm
   lea rax, [rip+page_base]
   add rax, page_offset
   ```

## Multi-string trace

If the user wants to find functions related to a **concept** rather than a specific string, search for multiple related strings and find functions that reference any of them:

```
probe.exe "strings find damage <module>"
probe.exe "strings find health <module>"
probe.exe "strings find armor <module>"
```

Functions that reference multiple related strings are likely the core handlers for that concept.

## Tips

- This is the most reliable way to find functions in an unknown binary — almost always works
- Error strings are the best targets because they're specific and rarely shared between functions
- Format strings with `%s`, `%d` etc. tell you about the function's parameters
- If a function references a string like `"ClassName::MethodName"`, that IS the function name (assert/debug string)
- After finding one function via string trace, use its callers/callees to discover related functions without string references
- Always generate signatures for important functions so they survive binary updates
