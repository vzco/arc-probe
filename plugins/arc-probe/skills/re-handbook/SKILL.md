---
name: re-handbook
description: Reverse engineering reference — techniques, patterns, and fallback strategies
---

# /probe:handbook

Reverse engineering reference — techniques, patterns, and fallback strategies for common tasks.

Load this when starting an investigation session. It covers the "what to do when X doesn't work" cases.

---

## Sending commands

Three methods, use whichever is available:

1. **CLI**: `probe.exe "<command>"` → JSON to stdout
2. **TCP**: `127.0.0.1:9998` — send `command\n`, receive JSON
3. **HTTP Bridge**: `POST http://localhost:9996` with `{"action":"probe","command":"..."}` (requires GUI)

---

## Reading memory safely

**Always expect failure.** Every `probe_read_*` call can return an error if the address is invalid. ARC Probe wraps all reads in SEH so the target won't crash, but you need to handle errors.

**Address validation heuristics:**
- Kernel addresses (above 0x7FFFFFFFFFFF on x64) are never readable from usermode
- NULL and near-NULL (< 0x10000) are always invalid
- Addresses in the first page (0x0000 - 0xFFFF) are reserved
- Stack addresses change every thread — read registers first to get RSP, then work from there
- Heap addresses are transient — objects get freed and reallocated

**Reading large regions:**
- Max 4096 bytes per read — split into chunks for larger regions
- Use `probe_dump` for visual inspection (max 256 bytes, includes ASCII)
- Use `probe_read` for raw hex when you need to parse specific bytes

---

## Writing memory safely

**Write commands:**
- `write <addr> <hex>` — raw bytes (for instruction patching)
- `write_int <addr> <value>` — 32-bit integer
- `write_float <addr> <value>` — 32-bit float
- `write_ptr <addr> <value>` — 8-byte pointer

**Rules:**
1. **Always read first** — record the original value before writing
2. **Verify after writing** — read the address again to confirm the change
3. **Never write to code sections** unless intentionally patching instructions (and record original bytes)
4. **Networked values** (health, position in multiplayer) may be overwritten by the server on the next tick
5. **Bool fields** in Source 2 may be `uint8` (1 byte) or `int32` (4 bytes) — check the schema

**Instruction patching:**
- NOP = `90` (1 byte per NOP, use `9090909090` for a 5-byte CALL)
- INT3 = `CC` (software breakpoint)
- RET = `C3` (force immediate return)
- Always save and restore original bytes

**Common pitfalls:**
- Writing to a freed object → crash on next access
- Writing `write_int` to a `uint8` field → overwrites 3 adjacent bytes
- Writing to a pointer field with a bad address → crash when dereferenced
- Modifying vtable entries → all instances of that class are affected

---

## Identifying what's at an address

When you have an address and don't know what's there:

1. **Hex dump first**: `probe_dump address=<addr> size=128`
   - `4D 5A` at the start? It's a PE header (module base)
   - Looks like code? (short varied bytes, no long runs of 00) → Disassemble it
   - Looks like data? (pointer-like values, readable ASCII) → It's a struct

2. **Check if it's a C++ object**: Read first 8 bytes
   - Points to .rdata section of a module? → vtable pointer → C++ object
   - Use `probe_rtti_resolve` to get the class name

3. **Check if it's a string**: `probe_read_string address=<addr>`
   - Returns readable text? → It's a string
   - Garbage? Try `probe_strings_at address=<addr> wide=true` for UTF-16

4. **Check which module it belongs to**: Compare address against module base+size ranges from `probe_modules`
   - In .text section → executable code
   - In .rdata → read-only data (strings, vtables, RTTI)
   - In .data → global variables (writable)
   - Not in any module → heap or stack allocation

---

## x86-64 calling convention (Microsoft x64)

Understanding calling conventions is critical for interpreting register state at breakpoints.

**Arguments:** RCX, RDX, R8, R9, then stack (left to right)
**Return value:** RAX (64-bit), EAX (32-bit), XMM0 (float/double)
**Caller-saved (volatile):** RAX, RCX, RDX, R8, R9, R10, R11
**Callee-saved (non-volatile):** RBX, RBP, RDI, RSI, R12, R13, R14, R15

**What this means in practice:**
- When a breakpoint fires at a function entry, RCX = first arg (usually `this` pointer for methods)
- RDX = second arg, R8 = third, R9 = fourth
- After a CALL, RAX has the return value
- RBX/RBP/RDI/RSI values are preserved across function calls — if you see these in a breakpoint log, they were set before the current function

**`this` pointer:** Always RCX for non-static member functions. So `mov eax, [rcx+0x354]` means "read field at offset 0x354 from the object".

**Virtual function calls:**
```asm
mov rax, [rcx]           ; load vtable from this (RCX)
call qword ptr [rax+0x28] ; call vtable entry at index 5 (0x28 / 8 = 5)
```

---

## Struct field identification patterns

**Getters** (return a field value):
```asm
mov eax, [rcx+0x354]   ; 32-bit field at +0x354
ret
```
or
```asm
movss xmm0, [rcx+0x40] ; float field at +0x40
ret
```

**Setters** (write a field value):
```asm
mov [rcx+0x354], edx    ; write second arg (EDX) to field at +0x354
ret
```

**Boolean flags:**
```asm
movzx eax, byte ptr [rcx+0x103]  ; read 1-byte boolean at +0x103
test al, al
jz skip
```

**Bitfield test:**
```asm
test dword ptr [rcx+0x48], 0x4   ; test bit 2 of flags at +0x48
jnz has_flag
```

**Array access:**
```asm
movsxd rax, edx            ; sign-extend index (EDX) to 64-bit
lea rcx, [rcx+rax*4+0x100] ; base + index*stride + offset → int32 array at +0x100
```

**Linked list traversal:**
```asm
.loop:
mov rcx, [rcx+0x8]    ; next = current->next (offset 0x8)
test rcx, rcx          ; while (current != NULL)
jnz .loop
```

---

## When things go wrong

### "The address changed after I found it"

Process memory is dynamic. Addresses change because:
- **ASLR**: Module base addresses change on every restart
- **Heap reallocation**: Objects move when resized or garbage collected
- **Entity systems**: Entities get created/destroyed, indices change

**Fix:** Don't store absolute addresses long-term. Instead:
- Store module-relative offsets (RVA) for code/global data
- Store pointer chains for dynamic data (GlobalPtr → EntityList → Entity)
- Generate byte signatures for functions

### "The pointer chain works sometimes but not always"

Common causes:
- **Entity not spawned yet**: The pointer is NULL until the entity exists
- **Wrong game state**: Some data only exists in-game (not in menus)
- **Thread timing**: You read while the game is updating the same data
- **Stale cache**: The entity died but the pointer wasn't cleared yet

**Fix:** Always check for NULL at each step of the chain. Read the chain multiple times to confirm stability.

### "The breakpoint never fires"

- **Wrong address**: The function might have been inlined by the compiler — the code exists but at a different address embedded in the caller
- **Wrong access type**: If watching for writes, maybe the code only reads the address
- **Thread affinity**: The code runs on a specific thread — hardware BPs work on all threads, so this shouldn't be the issue
- **Code path not taken**: The function exists but the game doesn't call it in the current state

**Fix:** Try a broader approach — set the breakpoint on the function prologue instead of a specific instruction. Or use `probe_watch` to poll for value changes first, then narrow down.

### "The disassembly doesn't make sense"

- **Wrong address**: You're disassembling in the middle of an instruction. x86 is variable-length — starting at the wrong byte produces garbage
- **Data misidentified as code**: .rdata sections contain data, not code. Check the section
- **Obfuscated code**: Some software intentionally inserts junk bytes, anti-disassembly tricks, or control flow flattening
- **JIT compiled**: Some engines (V8, .NET) generate code at runtime — the code only exists in heap memory, not in PE sections

**Fix:** Look for a known instruction pattern to re-synchronize. Function prologues are recognizable. Or disassemble from a known-good address (like an export or vtable entry) and follow the control flow forward.

### "I can't find the function with a string search"

- The function might not reference any strings (pure computation)
- The strings might be **obfuscated** (encrypted at rest, decrypted at runtime)
- The strings might be in a **different module** than the function
- The function might be in a **system DLL** (ntdll, kernel32)

**Alternative approaches:**
1. **Hardware breakpoint** on the data the function touches
2. **RTTI** → vtable → disassemble virtual functions
3. **Export table** if the function is exported
4. **Call tracing** — breakpoint a known caller and follow the call chain
5. **Pattern scan** for known opcode sequences if you have the function from an older build

---

## Source 2 engine specifics

If you're inspecting a Source 2 game (CS2, Deadlock, Dota 2):

**CreateInterface pattern:**
Most Source 2 DLLs export `CreateInterface`. Walking the interface registry gives you pointers to engine subsystems.
```
probe_interfaces_list module=<dll>
```

**Entity system:**
- Entity list is a chunked array — chunks of 512 entries, each entry is a CEntityIdentity (0x70 bytes)
- Entity pointer at identity + 0x0, designer name pointer at identity + 0x20
- Entity index = handle & 0x7FFF, chunk = index >> 9, slot = index & 0x1FF

**Schema system:**
All networked classes have schema metadata at runtime. Use `probe_rtti_scan` to discover all classes, then `probe_rtti_hierarchy` to understand inheritance.

**Common base classes:**
- CEntityInstance → C_BaseEntity → C_BasePlayerPawn → C_CitadelPlayerPawn
- CGameSceneNode → CSkeletonInstance (has bone data)
- CBasePlayerController (has m_hPawn handle to resolve to pawn)

**Key offsets pattern:**
- Health: entity + 0x354 (m_iHealth, int32)
- Team: entity + 0x3F3 (m_iTeamNum, uint8)
- Position: entity + 0x330 → GameSceneNode + 0xC8 (m_vecAbsOrigin, Vec3)
- Name: controller + 0x6F0 (m_iszPlayerName, const char*)
