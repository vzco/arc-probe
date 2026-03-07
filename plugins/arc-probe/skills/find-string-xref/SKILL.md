---
name: find-string-xref
description: Find a function by tracing from a known string to the code that references it
---

# /probe:xref

Find a function by tracing from a known string to the code that references it.

## Arguments

- `text` (required): The string to search for (error message, debug text, UI label, etc.)
- `module` (optional): Module to search in

## Why this works

Almost every function references at least one string — error messages, log output, assert text, format strings, UI labels. Strings live in .rdata (read-only data) and code loads their address via `LEA reg, [RIP+offset]`. Finding that LEA instruction puts you inside the function.

## Steps

1. **Find the string**:
   ```
   probe_strings_find text=<text> module=<module> max=5
   ```
   Note the exact address of the string. If multiple matches, pick the one in the expected module.

2. **Find code references to it**:
   ```
   probe_strings_xref address=<string_addr> module=<module> max=10
   ```
   Returns addresses of LEA instructions that load this string's address. Each one is a code location that uses this string.

3. **Disassemble each reference site**:
   ```
   probe_disassemble address=<ref_addr - 0x20> count=20
   ```
   Start a few bytes before the LEA to see context. You want to find:
   - What function this LEA is inside (look for the prologue above)
   - What the string is used for (passed to a log function? displayed in UI? used in an assert?)

4. **Find the function prologue** — scan backwards from the LEA:
   Look for these patterns (in order of likelihood):
   ```asm
   push rbx              ; 53
   sub rsp, 0x??         ; 48 83 EC ??
   ```
   ```asm
   push rbp              ; 55
   mov rbp, rsp          ; 48 8B EC  or  48 89 E5
   ```
   ```asm
   mov [rsp+8], rbx      ; 48 89 5C 24 08
   push rdi              ; 57
   sub rsp, 0x??         ; 48 83 EC ??
   ```
   The first instruction after a `ret` or `int3` (0xCC padding) is usually the start of the next function.

5. **Disassemble the full function**:
   ```
   probe_disassemble_function address=<function_start>
   ```

6. **Generate a signature**:
   ```
   probe_generate_signature address=<function_start>
   ```
   Test it: `probe_test_signature address=<function_start>`

7. **Report**:
   ```
   String: "Player took %d damage"
   Address: 0x7FFB22EA4098 (client.dll + 0x1914098, .rdata)

   Referenced by:
     0x7FFB2194FE75 (client.dll + 0x3BFE75) — lea rcx, [rip+0x15D421C]
       Function: 0x7FFB2194FE40 (client.dll + 0x3BFE40)
       Signature: 48 89 5C 24 08 57 48 83 EC 20 ...

     0x7FFB21EBE2D0 (client.dll + 0x92E2D0) — lea rdx, [rip+0x1585DC1]
       Function: 0x7FFB21EBE280 (client.dll + 0x92E280)
       Different function — likely the UI display handler
   ```

## If the string isn't found

- Try a **substring** — the full string might have format specifiers or prefixes you don't know
- Try **case variations** — `strings find` is case-sensitive
- The string might be **wide (UTF-16)** — add the `--wide` flag
- The string might be **encrypted or obfuscated** at rest — some software decrypts strings at runtime. In that case, the string won't be in .rdata. Try `probe_find_value type=string value=<text>` to search heap/stack
- The string might be in a **different module** — try without the module filter

## If xref returns zero results

- The string might be referenced via a **global pointer** instead of RIP-relative LEA. Try:
  - Pattern scan for the string's address bytes (little-endian) in the .rdata section
  - This catches cases where the address is stored in a table or array
- The string might only be used at **initialization** and the reference is in a `.CRT$XCU` section (static constructors)
- The LEA might use a **different register encoding** — `strings xref` scans for common LEA patterns but might miss unusual encodings
