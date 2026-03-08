---
name: analyze-function
description: Full analysis of a single function — disassemble, identify args, find string refs, callers/callees, RTTI, generate signature, label in GUI
argument-hint: <address> [module]
---

# /probe:analyze-function

Full analysis of a single function at a given address. Disassembles it, identifies arguments and return type, finds string references within it, maps callers and callees, checks for RTTI context, generates a signature, and labels everything in the GUI.

## Arguments

- `address` (required): Address of the function (hex). Can be absolute or `module+offset`.
- `module` (optional): Module the function belongs to (e.g., "client.dll"). If omitted, determined automatically from address range.

## Steps

### 1. Verify the address and determine module

```
probe.exe "modules list"
```

Compare the target address against each module's base + size range to identify which module it belongs to. Record the module name and base address. If the address is given as `module+offset`, resolve it to absolute.

If the address doesn't fall within any loaded module, it may be JIT-compiled or heap-allocated code. Note this and proceed with limited analysis (no xref scanning, no signature testing against a module).

### 2. Find the function boundaries

First, try to land on the function start. If the given address might be mid-function, check for a prologue:

```
probe.exe "disasm <address> 5"
```

Look for a function prologue at the address:
- `push rbp` / `sub rsp, 0x??`
- `push rbx` / `sub rsp, 0x??`
- `mov [rsp+8], rbx` / `push rdi` / `sub rsp, 0x??`
- `sub rsp, 0x??` (leaf-like)

If the first instruction is NOT a prologue, the user may have given an address mid-function. Disassemble backwards to find the start:

```
probe.exe "disasm <address - 0x80> 40"
```

Scan forward through the output looking for the prologue pattern closest before the target address. The function start is typically right after a `ret` or `int3` padding (`CC CC CC`).

### 3. Disassemble the full function

```
probe.exe "disasm func <function_start>"
```

This disassembles until a `ret` instruction. Record:
- **Function start** address
- **Function end** address (address of `ret` + instruction length)
- **Approximate size** (end - start)
- **Number of instructions**

If `disasm func` returns too many instructions (>500) or doesn't find a `ret`, the function may be very large or use tail calls. Fall back to disassembling a fixed count:

```
probe.exe "disasm <function_start> 200"
```

### 4. Identify calling convention and arguments

Analyze the function prologue and body for argument usage:

**Register arguments (Microsoft x64 ABI):**
- RCX = first arg (or `this` pointer for member functions)
- RDX = second arg
- R8 = third arg
- R9 = fourth arg
- Stack for 5th+ args

**Look for these patterns in the disassembly:**

- `mov [rsp+...], rcx` early on — saves first arg, confirms it's used
- `test rcx, rcx` / `cmp rcx, 0` — null check on first arg (likely a pointer)
- `mov rax, [rcx]` — loading vtable from first arg means RCX is a `this` pointer (member function)
- `movss xmm0, [rcx+0x??]` — reading float fields from first arg
- References to RDX, R8, R9 — function takes multiple arguments

**Return type clues:**
- `xor eax, eax` before `ret` — returns 0 (bool or int)
- `mov eax, [rcx+0x??]` then `ret` — getter returning int32
- `movss xmm0, [rcx+0x??]` then `ret` — getter returning float
- `lea rax, [rcx+0x??]` then `ret` — returns pointer to sub-object

### 5. Find string references within the function

Scan the disassembly output for `LEA` instructions with RIP-relative addressing:

```asm
lea rcx, [rip+0x????]    ; loads a string address
lea rdx, [rip+0x????]    ; second string argument
```

For each LEA with a RIP-relative operand, the target address is: `LEA_address + instruction_length + displacement`. Read the string at that address:

```
probe.exe "read_string <resolved_address>"
```

If the string is readable, it reveals the function's purpose. Common patterns:
- Error/assert strings: `"Failed to initialize %s"`
- Log strings: `"[%s] Player %d connected"`
- Debug names: `"CBaseEntity::TakeDamage"`

### 6. Find callers (who calls this function)

```
probe.exe "xref scan <function_start> <module> --type CALL --limit 20"
```

This scans the module for `CALL` instructions whose target is this function. Each result includes:
- The address of the calling instruction
- The enclosing function (from .pdata)
- The disassembled instruction

For each caller, note the enclosing function address. These are the function's callers.

If zero callers are found:
- The function may be called **indirectly** (via function pointer or vtable). Try `--type CALL,MOV` to catch indirect references.
- The function may be called from a **different module**. Try without the module filter or scan other modules.
- The function may be **inlined** by the compiler and never called directly.

### 7. Find callees (what this function calls)

```
probe.exe "xref scan <function_start> <module> --callees --limit 30"
```

This disassembles the function and collects all CALL/JMP targets. Each result includes:
- The source instruction address
- The target function address
- Whether it's a CALL or JMP (tail call)

For interesting callees, do a quick identification:

```
probe.exe "disasm <callee_address> 5"
```

Look for recognizable patterns — tiny getters, known API functions, or functions with string references.

### 8. Check RTTI context

If step 4 identified the function as a member function (uses `this` in RCX), check for RTTI:

```
probe.exe "rtti resolve <any_object_pointer_of_this_class>"
```

If you don't have an object pointer, check if the function appears in any vtable:

```
probe.exe "rtti scan <module>"
```

Search the results for a vtable entry matching the function address. If found, you know:
- The class name
- The vtable index (virtual function number)
- The inheritance chain

### 9. Generate a signature

```
probe.exe "sig <function_start> 32 <module>"
```

Test the signature for uniqueness:

```
probe.exe "pattern <signature> <module>"
```

A good signature has exactly 1 match. If it has multiple matches:

```
probe.exe "sig <function_start> 64 <module>"
```

Try a longer signature. If still not unique, note that the function has duplicate prologues and requires a longer or manually-crafted signature.

### 10. Label in the GUI

If the GUI bridge is available, label the function and navigate to it:

```bash
# Set activity status
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"activity","status":"working","message":"Labeling function..."}'

# Label the function
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"batch","actions":[
    {"action":"store","store":"label","method":"setLabel","args":["0x<function_start>","<function_name_or_description>"]},
    {"action":"navigate","tab":"disasm","address":"0x<function_start>"},
    {"action":"activity","status":"idle","message":"Done"}
  ]}'
```

Also label any identified callers/callees:

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"store","store":"label","method":"setLabel","args":["0x<caller_addr>","caller: <description>"]}'
```

### 11. Report

Present a structured analysis:

```
Function Analysis: <name or description>
======================================

Address:    0x7FF612345678 (<module> + 0x5678)
Size:       0x1A0 bytes (~96 instructions)
Signature:  48 89 5C 24 08 57 48 83 EC 20 48 8B D9 E8 ?? ?? ?? ??

Calling Convention: __fastcall (x64)
Arguments:
  RCX = this pointer (C_BaseEntity*)
  RDX = int32 damage amount
  R8  = pointer to damage info struct
Returns: int32 (remaining health)

Strings Referenced:
  +0x42: lea rcx, "CBaseEntity::TakeDamage"
  +0x8A: lea rdx, "Entity %d took %d damage"

Callers (3):
  0x7FF612340000 (<module> + 0x0000) — GameRules::ProcessDamage
  0x7FF612341200 (<module> + 0x1200) — CombatSystem::ApplyHit
  0x7FF612342800 (<module> + 0x2800) — AI_TakeDamage

Callees (5):
  +0x1F: CALL 0x7FF612350000 — GetHealth (getter, returns [rcx+0x354])
  +0x35: CALL 0x7FF612350100 — SetHealth (setter, writes [rcx+0x354])
  +0x58: CALL 0x7FF612360000 — LogMessage (references format string)
  +0x7A: CALL 0x7FF612370000 — Unknown (large function, ~0x400 bytes)
  +0xA2: JMP  0x7FF612380000 — tail call to cleanup function

RTTI: CBaseEntity (vtable index 12)
```

## If the function is very small (< 10 instructions)

Small functions are usually:
- **Getters**: `mov eax, [rcx+offset]; ret` — report the field offset and type
- **Setters**: `mov [rcx+offset], edx; ret` — report the field offset
- **Thunks**: `jmp <other_function>` — follow the jump and analyze the real function
- **Wrappers**: call one function and return its result

For these, the full analysis is overkill. Just report the function type, the field offset (for getters/setters), and the target (for thunks).

## If disasm func fails or returns garbage

- The address may not be the start of a function. Look for `CC CC CC` padding (int3 alignment) before the address and try the next instruction after padding.
- The code may be obfuscated. Look for `jmp` chains, unusual control flow, or encrypted instruction sequences.
- The address may point to data, not code. Use `dump <address> 64` to see if it looks like data patterns rather than instruction bytes.

## Tips

- String references are the fastest way to understand a function's purpose
- Callers tell you "who uses this" — essential for understanding the function's role in the system
- Callees tell you "what this depends on" — essential for understanding the function's implementation
- The combination of callers + callees + strings usually gives enough context to name the function
- Always generate a signature so the function can be found again after binary updates
