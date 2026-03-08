---
name: find-callers
description: Find all functions that call a given function address — shows argument setup, string refs, and containing function names
argument-hint: <address> [module]
---

# /probe:find-callers

Find all functions that call a given function address. For each caller, disassemble the argument setup instructions before the call site to understand what parameters are being passed.

## Arguments

- `address` (required): Address of the target function (hex). Can be absolute or `module+offset`.
- `module` (optional): Module to search for callers in. If omitted, determined automatically from the address range.

## Steps

### 1. Verify connection and resolve address

```
probe.exe "ping"
```

If using the bridge:

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"ping"}'
```

If the address is `module+offset`, resolve it to absolute:

```
probe.exe "modules list"
```

Compare the target address against each module's base + size range to identify which module it belongs to. Record the module name and base address.

### 2. Confirm the target is a valid function

```
probe.exe "disasm <address> 5"
```

Look for a function prologue at the address:
- `push rbp` / `sub rsp, 0x??`
- `push rbx` / `sub rsp, 0x??`
- `mov [rsp+8], rbx` / `sub rsp, 0x??`
- `sub rsp, 0x??` (leaf-like)

If the first instruction is NOT a prologue, the user may have given a mid-function address. Search backwards for the start:

```
probe.exe "disasm <address - 0x80> 40"
```

Look for the prologue closest before the target address. Use the corrected function start for the xref scan.

### 3. Scan for callers

```
probe.exe "xref scan <address> <module> --type CALL --limit 20"
```

This scans the module for `CALL` and `JMP` instructions whose target resolves to the function address. Each result includes:
- `source` -- address of the calling instruction
- `function` -- enclosing function address (from .pdata)
- `instruction` -- the disassembled call/jmp instruction

If **zero callers** are found:
- The function may be called **indirectly** (via function pointer or vtable). Try `--type CALL,MOV` to catch indirect references like `mov rax, <addr>`.
- The function may be called from a **different module**. Try scanning other loaded modules:
  ```
  probe.exe "modules list"
  ```
  Then scan each relevant module.
- The function may be **inlined** by the compiler -- it has no direct call sites.
- The function may only be called via a **vtable**. Check if it appears in any RTTI vtable:
  ```
  probe.exe "rtti scan <module>"
  ```

### 4. Analyze argument setup at each call site

For each caller found, disassemble the 8 instructions leading up to the call to see the argument setup:

```
probe.exe "disasm <call_site_address - 0x20> 12"
```

Look for these argument-passing patterns (Windows x64 ABI):

- `mov rcx, ...` or `lea rcx, ...` -- first argument (or `this` pointer)
- `mov rdx, ...` or `lea rdx, ...` -- second argument
- `mov r8, ...` or `lea r8, ...` -- third argument
- `mov r9, ...` or `lea r9, ...` -- fourth argument
- `mov [rsp+0x20], ...` -- fifth argument (stack)

Pay special attention to:
- `lea rcx, [rip+0x????]` -- loading a string address as an argument. Resolve it:
  ```
  probe.exe "read_string <resolved_rip_target>"
  ```
- `mov rcx, <register>` where the register was loaded from a struct field -- reveals context about the caller.
- Constant values being passed (`mov edx, 1`, `xor r8d, r8d`) -- reveals parameter semantics.

### 5. Identify containing function names

For each caller, check if the enclosing function has any identifying information:

a. **Check for labels** (if GUI bridge is running):
```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"store","store":"label","method":"getLabel","args":["0x<enclosing_function>"]}'
```

b. **Check for string references** inside the caller function:
```
probe.exe "disasm func <enclosing_function>"
```
Scan the output for `lea` instructions with RIP-relative addressing and resolve them to strings.

c. **Check function store** (functions previously discovered):
```
probe.exe "functions at <enclosing_function>"
```

### 6. Present results

Format the output as a table:

```
Callers of sub_7FF612345678 (client.dll + 0x5678)
===================================================

#  Caller Function           Call Site              Args Before Call
-- ----------------------    --------------------   ----------------------------------------
1  sub_7FF612340000          client.dll + 0x0042    lea rcx, [rip+0x1234] -> "player_name"
   (client.dll + 0x0000)                            mov edx, [rbx+0x354]  (health field)
                                                    xor r8d, r8d          (arg3 = 0)

2  ProcessDamage             client.dll + 0x1234    mov rcx, rsi          (this pointer)
   (client.dll + 0x1200)                            mov edx, edi          (damage amount)
                                                    mov r8, [rsp+0x40]    (stack local)

3  sub_7FF612342800          client.dll + 0x2842    lea rcx, [rbp+0x30]   (local struct)
   (client.dll + 0x2800)                            lea rdx, [rip+0x5678] -> "entity"
   refs: "AI_TakeDamage"                            mov r8d, 1            (arg3 = 1)

Total: 3 direct callers found in client.dll
```

### 7. Navigate in GUI (if bridge available)

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"activity","status":"working","message":"Navigating to first caller..."},
    {"action":"navigate","tab":"disasm","address":"0x<first_call_site>"},
    {"action":"store","store":"label","method":"setLabel","args":["0x<target_function>","<function_name_or_addr>"]},
    {"action":"activity","status":"idle","message":"Done — found N callers"}
  ]
}'
```

Also label each caller if a name was identified:

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"store","store":"label","method":"setLabel","args":["0x<caller_addr>","caller_name"]}'
```

## Tips

- String references in the argument setup are the fastest way to understand what is being passed to the function
- If a caller passes `this` (RCX from a struct access pattern), trace it to understand the calling object's type
- Callers that pass constants (0, 1, NULL) as arguments reveal the function's parameter semantics
- If you find many callers (>10), the function is likely a utility function (logging, allocation, validation)
- If you find exactly one caller, the function may be a helper extracted by the compiler
- Tail calls (`jmp` instead of `call`) are also callers -- xref scan catches these with `--type CALL`
- If the function is virtual, callers will use `call [rax+offset]` (indirect) and won't appear in a direct xref scan
