---
name: detect-params
description: Detect function parameters from prologue and register usage — infer types, count, and calling convention
argument-hint: <address>
---

# /probe:detect-params

Detect the parameters of a function by analyzing its prologue, register usage, and instruction patterns. Infers parameter count, types, and calling convention for Windows x64 (__fastcall).

## Arguments

- `address` (required): Address of the function start (hex). Can be absolute or `module+offset`.

## Steps

### 1. Disassemble the function

```
probe.exe "disasm func <address>"
```

If `disasm func` fails or returns too many instructions (>500), fall back to a fixed count:

```
probe.exe "disasm <address> 80"
```

Record the full disassembly output. You will scan it for register usage patterns.

### 2. Analyze the prologue for saved registers

Look at the first 5-10 instructions for the function prologue pattern:

**Non-volatile register saves (tells you how many locals the function needs):**
```asm
mov [rsp+0x08], rcx    ; saves arg1 to shadow space
mov [rsp+0x10], rdx    ; saves arg2 to shadow space
mov [rsp+0x18], r8     ; saves arg3 to shadow space
push rbx               ; saves non-volatile register
sub rsp, 0x40          ; allocate stack frame
```

**Key prologue patterns:**
- `mov [rsp+0x08], rcx` early -- function saves first arg, confirms RCX is used
- `mov [rsp+0x10], rdx` early -- function saves second arg, confirms RDX is used
- `movaps [rsp+...], xmm6` or `movups` -- saves non-volatile XMM registers (function uses floats internally)
- Large `sub rsp, 0x??` -- many locals or large stack-allocated structs

### 3. Scan for first usage of each argument register

Walk through the disassembly and find the **first** occurrence of each argument register. The key rule: if a register is **read before it is written**, it is a parameter.

**Integer argument registers (Windows x64 ABI):**

| Register | Arg # | Notes |
|----------|-------|-------|
| RCX/ECX/CX/CL | 1 | Also `this` pointer for member functions |
| RDX/EDX/DX/DL | 2 | |
| R8/R8D/R8W/R8B | 3 | |
| R9/R9D/R9W/R9B | 4 | |

**Float argument registers (shadow the same slots):**

| Register | Arg # | Notes |
|----------|-------|-------|
| XMM0 | 1 | Replaces RCX when arg1 is float/double |
| XMM1 | 2 | Replaces RDX when arg2 is float/double |
| XMM2 | 3 | Replaces R8 when arg3 is float/double |
| XMM3 | 4 | Replaces R9 when arg4 is float/double |

For each register, classify the first instruction that references it:

**Read (parameter confirmed):**
- `mov rax, [rcx]` -- reads from RCX (pointer dereference)
- `test rcx, rcx` -- tests RCX (null check)
- `cmp edx, 0x10` -- compares RDX (integer comparison)
- `mov [rsp+...], rcx` -- saves RCX (shadow space store)
- `movss xmm4, xmm0` -- copies XMM0 (float parameter save)

**Write (NOT a parameter -- register is being assigned):**
- `xor ecx, ecx` -- zeroes RCX
- `mov rcx, rax` -- overwrites RCX
- `lea rcx, [rip+...]` -- loads address into RCX (preparing a call)

**Stop scanning a register** once you find its first read or write. If the first usage is a **write**, that register slot is NOT a parameter.

### 4. Infer parameter types

For each confirmed parameter register, determine the type from how it is used:

**Pointer type** (most common for RCX):
- `mov rax, [rcx]` -- dereferences as pointer (vtable load = `this` pointer)
- `mov eax, [rcx+0x354]` -- struct field access at known offset
- `lea rax, [rcx+0x10]` -- address arithmetic on pointer
- `test rcx, rcx` followed by `je` -- null check (definitely a pointer)
- `call [rax+0x??]` after `mov rax, [rcx]` -- virtual function call (RCX is object pointer)

**Integer type:**
- `cmp edx, <immediate>` -- range check
- `and edx, 0xFF` -- masking to byte
- `shl edx, 2` -- bit shift (index calculation)
- `add edx, eax` -- arithmetic
- `mov [rax+...], edx` -- stored as 32-bit value

**Boolean type:**
- `test dl, dl` -- boolean test
- `movzx eax, dl` -- zero-extend byte
- `cmp cl, 0` / `cmp cl, 1` -- explicit boolean check

**Float type:**
- `movss [rsp+...], xmm0` -- saved as float
- `comiss xmm0, xmm1` -- float comparison
- `mulss xmm0, [rcx+...]` -- float multiply
- `addss xmm0, xmm1` -- float add
- `cvtsi2ss xmm0, eax` -- int-to-float conversion (function receives int, converts)

**String type (const char*):**
- RCX/RDX passed to a function that eventually calls `strlen`, `strcmp`, or prints it
- The register value is used with `movzx eax, byte [rcx]` -- reading characters

### 5. Detect `this` pointer (C++ member function)

A function is likely a member function if RCX shows these patterns:

- `mov rax, [rcx]` as the first or second instruction -- vtable load (definitive)
- Multiple field accesses like `[rcx+0x10]`, `[rcx+0x354]`, `[rcx+0x3F3]` -- reading struct fields
- `this` is passed through to callees: `mov rcx, rbx` where RBX was saved from the original RCX
- The function appears in a vtable (`rtti vtable` results include this address)

If RCX is a `this` pointer, the effective parameter count starts from RDX (arg2 becomes the "first visible" parameter).

### 6. Check for stack parameters (5th+ arguments)

If the function accesses `[rsp+0x28]` or higher offsets early in the function (before allocating locals), these are stack-passed arguments:

```asm
mov rax, [rsp+0x28]    ; 5th argument (after 0x20 shadow space)
mov rax, [rsp+0x30]    ; 6th argument
mov rax, [rsp+0x38]    ; 7th argument
```

**Important:** After `sub rsp, N`, the stack offsets shift. Stack parameters are at `[rsp+N+0x28]`, `[rsp+N+0x30]`, etc. Only check for stack params in instructions BEFORE the `sub rsp` or adjust offsets accordingly.

### 7. Detect return type

Scan the function's epilogue (instructions before `ret`):

- `xor eax, eax` then `ret` -- returns 0 (bool false or int 0)
- `mov eax, 1` then `ret` -- returns true or success
- `mov eax, [rbx+0x??]` then `ret` -- returns an int32 field (getter)
- `mov rax, rbx` then `ret` -- returns a pointer
- `lea rax, [rbx+0x??]` then `ret` -- returns pointer to sub-object
- `movss xmm0, [rbx+0x??]` then `ret` -- returns float
- No `eax`/`rax`/`xmm0` assignment before `ret` -- likely void return

Check multiple return paths (the function may have early returns with different patterns).

### 8. Cross-reference with call sites (optional, for validation)

Find callers to see what arguments they actually pass:

```
probe.exe "xref scan <address> <module> --type CALL --limit 5"
```

For each caller, disassemble the setup before the call:

```
probe.exe "disasm <call_site - 0x20> 12"
```

The argument setup at call sites confirms the parameter count and types:
- If callers always set RDX before calling, the function has at least 2 parameters
- If callers pass `lea rdx, [rip+...]` (string), the second parameter is `const char*`
- If callers pass `xmm1` instead of RDX, the second parameter is float

### 9. Present the function signature

Build a C-style function signature:

```
Function: sub_7FF612345678 (client.dll + 0x5678)
Size: ~0x1A0 bytes

Detected Signature:
  int __fastcall sub_7FF612345678(void* this, int damage, const char* source, bool notify)

Parameters:
  RCX  this      void*         vtable at [rcx], field access [rcx+0x354], [rcx+0x3F3]
  RDX  damage    int32         compared with immediate, used in arithmetic
  R8   source    const char*   lea r8, [rip+...] -> "player" at call sites
  R9   notify    bool          test r9b, r9b -> conditional branch

Return:
  EAX  int32     xor eax,eax before early ret; mov eax,[rbx+0x354] before normal ret

Stack frame: 0x40 bytes (sub rsp, 0x40)
Saved registers: RBX, RDI (non-volatile)
```

If using the GUI bridge, label the function with the detected signature:

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"store","store":"label","method":"setLabel","args":["0x<address>","int sub_XXXX(void* this, int arg2, char* arg3, bool arg4)"]}'
```

## If the function is very small (< 5 instructions)

Small functions have obvious signatures:

- **Getter**: `mov eax, [rcx+0x354]; ret` -- `int GetField(void* this)` -- 1 param (this)
- **Setter**: `mov [rcx+0x354], edx; ret` -- `void SetField(void* this, int value)` -- 2 params
- **Boolean getter**: `movzx eax, byte [rcx+0x103]; ret` -- `bool GetFlag(void* this)` -- 1 param
- **Thunk**: `jmp <other>` -- follow the jump and analyze the real function

## If register usage is ambiguous

Sometimes a register is both read and written in complex ways. Fallback strategies:

1. **Check .pdata** for unwind info -- it tells you the parameter count for SEH:
   ```
   probe.exe "call info <address>"
   ```

2. **Count shadow space stores** -- the number of `mov [rsp+0x??], reg` instructions in the prologue (where offset is 0x08, 0x10, 0x18, 0x20) directly indicates the parameter count.

3. **Check callers** -- the most reliable way to confirm parameter count is to see what callers actually pass.

## Tips

- Windows x64 ABI always reserves 32 bytes of shadow space (0x20) even for functions with fewer than 4 parameters
- Leaf functions (no calls) may skip the prologue entirely -- no `sub rsp`, no register saves
- Variadic functions (`printf`-style) store all 4 register args to shadow space unconditionally
- If XMM0 is used AND RCX is used for different purposes, the function has a mix of float and int parameters
- The `this` pointer convention means C++ methods have one fewer "visible" parameter than register analysis suggests
- Compiler optimizations may pass parameters in unexpected registers -- callers are the ground truth
