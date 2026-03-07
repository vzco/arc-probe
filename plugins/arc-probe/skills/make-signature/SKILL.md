---
name: make-signature
description: Generate a byte signature for a function that survives binary updates
---

# /probe:sig

Generate a byte signature for a function that survives binary updates.

## Arguments

- `address` (required): Address of the function (hex)
- `module` (optional): Module name for context

## Why signatures matter

Binary updates change function addresses but rarely change the instruction sequence. A byte signature captures the function's unique bytes with wildcards for parts that change (relocations, offsets). This lets you find the function again after every update.

## Steps

1. **Disassemble the function start**:
   ```
   probe_disassemble address=<addr> count=15
   ```
   Look at the first 15-20 instructions. The function prologue and early logic are usually the most unique.

2. **Auto-generate a signature**:
   ```
   probe_generate_signature address=<addr> size=32
   ```
   Returns a pattern with `??` wildcards for bytes that are likely to change between builds.

3. **Test uniqueness**:
   ```
   probe_test_signature address=<addr> module=<module>
   ```
   A good signature has exactly 1 match. If it has multiple matches, you need a longer or more specific signature.

4. **If not unique**, extend the signature:
   - Read more bytes: `probe_generate_signature address=<addr> size=64`
   - Or manually craft a signature using the disassembly — pick instruction bytes that are distinctive

5. **Verify the signature finds the right function**:
   ```
   probe_pattern_scan pattern=<signature> module=<module>
   ```
   The result should be the original function address.

## What to wildcard (use ??)

**Always wildcard:**
- **RIP-relative displacements** (4 bytes after `48 8B 05`, `48 8D 0D`, `E8`, etc.) — these change every build
- **Stack frame sizes** that depend on local variables — compiler may reorder
- **Immediate operands** that reference build-specific constants

**Never wildcard:**
- **Opcodes** (`48 8B`, `48 89`, `E8`, etc.) — these define the instruction type
- **Register encodings** — which registers the function uses rarely changes
- **ModR/M bytes** — encode addressing modes, very stable

## Manual signature crafting

If auto-generation fails, read the raw bytes and apply wildcards manually:

```
Address     Bytes                      Instruction
7FF6A000    48 89 5C 24 08            mov [rsp+8], rbx      ; stable
7FF6A005    57                         push rdi               ; stable
7FF6A006    48 83 EC 20               sub rsp, 0x20          ; stack size might change
7FF6A00A    48 8B D9                  mov rbx, rcx           ; stable
7FF6A00D    E8 XX XX XX XX            call SomeFunc          ; wildcard the call target
7FF6A012    48 85 C0                  test rax, rax          ; stable
7FF6A015    74 XX                     je short label         ; wildcard the branch offset
```

Resulting signature:
```
48 89 5C 24 08 57 48 83 EC ?? 48 8B D9 E8 ?? ?? ?? ?? 48 85 C0 74 ??
```

## Good vs bad signatures

**Good:**
```
48 89 5C 24 08 57 48 83 EC 20 48 8B D9 E8 ?? ?? ?? ?? 48 85 C0
```
- Specific prologue pattern (mov [rsp+8], rbx + push rdi)
- Stable register usage
- Unique call sequence

**Bad:**
```
48 89 5C 24 ?? 48 83 EC ??
```
- Too short — hundreds of functions start with mov [rsp+?], reg + sub rsp
- Too many wildcards — matches almost anything

**Rule of thumb:** 16-32 bytes with 2-4 wildcards is usually enough. If you need more than 48 bytes, the function might not be unique enough — consider combining with a module name filter.

## Tips

- **Function prologues vary by calling convention** — `__fastcall` (default x64) typically saves non-volatile registers and allocates stack
- **Leaf functions** (no calls) often have no prologue at all — they just start with the first instruction
- **Inlined functions** don't have their own prologue — the signature must capture the surrounding context
- **Virtual functions** can be found via RTTI vtable index instead of a byte signature — this is often more reliable since vtable index doesn't change
