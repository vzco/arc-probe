---
name: function-diff
description: Compare a function between sessions — generate a signature, find it after binary update, compare disassembly, report changes
argument-hint: <signature_or_address> [module]
---

# /probe:function-diff

Compare a function between two sessions (e.g., before and after a binary update). Use a byte signature to relocate the function in the new binary, then compare disassembly to identify what changed.

This skill has two phases:
- **Phase 1 (before update)**: Capture the function — disassemble, generate signature, record details
- **Phase 2 (after update)**: Re-find the function, compare, report changes

## Arguments

- `signature_or_address` (required): Either a byte signature pattern (e.g., `"48 89 5C 24 08 57 48 83 EC 20 48 8B D9 E8 ?? ?? ?? ??"`) or a function address (hex) to capture
- `module` (optional): Module name to search in

## Phase 1: Capture (Before Update)

Run this phase while the current binary is loaded. The goal is to record everything needed to find and compare the function later.

### 1. Locate the function

**If given an address:**

```
probe.exe "modules list"
```

Determine the module and RVA:

```
RVA = address - module_base
```

**If given a signature:**

```
probe.exe "pattern <signature> <module>"
```

Verify the signature matches exactly one location. If multiple matches, narrow the search or extend the signature.

### 2. Disassemble and record

```
probe.exe "disasm func <function_address>"
```

Record the full disassembly. Pay attention to:
- **Total instruction count**
- **Function size** (last address - first address + last instruction length)
- **String references** (LEA instructions with RIP-relative operands)
- **Call targets** (CALL instructions — record the target addresses)
- **Notable constants** (immediate values, magic numbers)

For each string reference, read the string:

```
probe.exe "read_string <resolved_lea_target>"
```

For each call target, check if it has a name:

```
probe.exe "disasm <call_target> 3"
```

### 3. Generate the signature

If not already provided as a signature:

```
probe.exe "sig <function_address> 32 <module>"
```

Test uniqueness:

```
probe.exe "pattern <signature> <module>"
```

If not unique, extend:

```
probe.exe "sig <function_address> 64 <module>"
```

### 4. Record the capture

Store this information (present it to the user to save):

```
Function Capture:
  Name:      <description or label>
  Module:    <module_name>
  Address:   0x<addr> (RVA: 0x<rva>)
  Size:      <N> bytes (<M> instructions)
  Signature: <pattern>

  Disassembly: (full listing)
  Strings:     (list with offsets)
  Callees:     (list with offsets)
```

Tell the user: **"Save this capture. After the binary updates, run this skill again with the signature to compare."**

---

## Phase 2: Compare (After Update)

Run this phase after the binary has been updated (new build, game patch, etc.). The module will be loaded at a different base address and the function may have changed.

### 5. Relocate the function

```
probe.exe "pattern <signature> <module>"
```

**If found (1 match):** The function exists at a new address. Proceed to comparison.

**If found (multiple matches):** The signature is no longer unique — the update added similar code. Try:
- Use a longer portion of the original signature
- Cross-reference with a known string — if the function referenced a specific string, find that string first, then check which signature match is near the string's xref

**If not found (0 matches):** The function was significantly modified. Strategies:

a. **Try a sub-signature** — use just the first 16 bytes of the original signature (the prologue is less likely to change):
   ```
   probe.exe "pattern <first_16_bytes_of_sig> <module>"
   ```

b. **Search by string reference** — if the function referenced a unique string, find that string and trace xrefs:
   ```
   probe.exe "strings find <unique_string> <module>"
   probe.exe "strings xref <string_addr> <module>"
   ```

c. **Search by export name** — if the function was an export, check if the export still exists:
   ```
   probe.exe "pe exports <module>"
   ```

d. **Search by RTTI** — if the function was a virtual method, the class likely still exists:
   ```
   probe.exe "rtti vtable <class_name> <module>"
   ```
   Check the same vtable index.

e. **The function was removed or completely rewritten** — report this finding. The update fundamentally changed this code path.

### 6. Disassemble the new version

```
probe.exe "disasm func <new_function_address>"
```

Record the new disassembly with the same level of detail as the capture.

### 7. Compare the two versions

Analyze the differences between the captured (old) and current (new) disassembly:

**Structural comparison:**
- Same instruction count? If different, instructions were added or removed.
- Same function size? Growth suggests new logic; shrinkage suggests simplification or removal.
- Same prologue? If different, the calling convention or stack usage changed.

**Instruction-level comparison:**

Walk through both listings instruction by instruction. Ignore expected differences:
- **RIP-relative displacements** — always change between builds (module base moves)
- **CALL targets** — absolute addresses change; compare the target function identity instead
- **Stack offsets** — may shift if local variables were added/removed

Flag actual differences:
- **New instructions** — code was added (new logic, new checks, new branches)
- **Removed instructions** — code was removed (simplification, bug fix, feature removal)
- **Changed opcodes** — different instruction at the same logical position (optimization, bug fix)
- **Changed constants** — immediate values changed (struct offset changes, enum value changes)
- **Changed branch targets** — control flow restructured
- **New string references** — new error messages or log text added
- **Removed string references** — old error handling removed

**Semantic comparison:**

For each changed region, determine the semantic meaning:
- New `cmp`/`test` + `jcc` — a new conditional check was added
- New `call` — a new function is being called (what does it do?)
- Removed `call` — a function call was dropped (inlined? removed?)
- Changed offset in field access (`[rcx+0x354]` became `[rcx+0x358]`) — struct layout changed

### 8. Check struct offset changes

If field access offsets changed (e.g., `[rcx+0x354]` in the old version vs `[rcx+0x358]` in the new), this indicates a struct layout change. A field was inserted before this offset, pushing everything down. Document:

```
Struct offset shift detected:
  Old: [rcx+0x354] (m_iHealth)
  New: [rcx+0x358] (m_iHealth — shifted by +4)
  Cause: 4 bytes inserted before offset 0x354
```

### 9. Generate new signature

```
probe.exe "sig <new_function_address> 32 <module>"
```

Test and verify the new signature:

```
probe.exe "pattern <new_signature> <module>"
```

### 10. Label the updated function in GUI

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"activity","status":"working","message":"Labeling updated function..."},
    {"action":"store","store":"label","method":"setLabel","args":["0x<new_addr>","<function_name> (updated)"]},
    {"action":"navigate","tab":"disasm","address":"0x<new_addr>"},
    {"action":"activity","status":"idle","message":"Done — function diff complete"}
  ]
}'
```

### 11. Report

```
Function Diff: CBaseEntity::TakeDamage
=======================================

Old: 0x7FFB2194FE40 (client.dll + 0x3BFE40) — 96 instructions, 0x1A0 bytes
New: 0x7FFB21A5F300 (client.dll + 0x4CF300) — 102 instructions, 0x1C8 bytes
     (+6 instructions, +0x28 bytes)

Signature:
  Old: 48 89 5C 24 08 57 48 83 EC 20 48 8B D9 E8 ?? ?? ?? ??
  New: 48 89 5C 24 08 57 48 83 EC 30 48 8B D9 E8 ?? ?? ?? ??
       (stack frame grew from 0x20 to 0x30)

Changes:
  1. [+0x1A] NEW: cmp dword ptr [rcx+0x3FC], 0
              NEW: jz +0x12
     Analysis: New null/zero check on a field at +0x3FC (new field?)

  2. [+0x42] CHANGED: mov eax, [rcx+0x354]  -->  mov eax, [rcx+0x358]
     Analysis: m_iHealth offset shifted from 0x354 to 0x358 (+4 bytes)

  3. [+0x8A] NEW: call 0x7FFB21B02000
     Analysis: New function call added — appears to be a validation check
              (target function references string "damage validation failed")

  4. [+0xC0] REMOVED: call 0x7FFB2190A000
     Analysis: Old logging call removed (was calling LogMessage with "damage applied")

Struct Offset Changes:
  m_iHealth:    0x354 -> 0x358 (+4)
  m_iMaxHealth: 0x350 -> 0x354 (+4)
  m_lifeState:  0x35C -> 0x360 (+4)
  Likely cause: 4-byte field inserted before 0x350

New Signature: 48 89 5C 24 08 57 48 83 EC 30 48 8B D9 E8 ?? ?? ?? ??
```

## Single-session mode

If you're not comparing across sessions but want to compare two similar functions in the same binary (e.g., overridden virtual function in base vs derived class):

1. Disassemble both functions
2. Skip the signature/relocation steps
3. Go straight to step 7 (comparison)
4. Focus on what the derived class adds or changes vs the base class

## Tips

- **Prologue changes** are significant — they indicate stack frame or calling convention changes
- **Offset shifts** in struct field access (e.g., +0x354 -> +0x358) almost always mean a field was inserted/removed from the struct definition
- **New conditional branches** often indicate new gameplay checks, anti-cheat additions, or bug fixes
- **Removed calls** might mean a feature was disabled or a function was inlined
- **The signature is your anchor** — always generate and save a signature during Phase 1
- **String references are stable anchors** — even if the code changes significantly, the strings it references usually stay the same
- **vtable index is very stable** — for virtual functions, the vtable slot number rarely changes between updates (but it can happen if a new virtual function is inserted in a base class)
- For game reverse engineering, run Phase 1 before every major patch, and Phase 2 immediately after to catch all offset and logic changes
