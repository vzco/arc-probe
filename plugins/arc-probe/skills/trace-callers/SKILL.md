---
name: trace-callers
description: Build a reverse call tree by following callers up multiple levels — shows the full chain of functions leading to a target
argument-hint: <address> [depth] [module]
---

# /probe:trace-callers

Build a reverse call tree by following the call chain upward from a target function. Finds direct callers, then their callers, up to N levels deep. Shows argument setup at each call site and builds a visual tree of the call hierarchy.

## Arguments

- `address` (required): Address of the target function (hex). Can be absolute or `module+offset`.
- `depth` (optional): Number of levels to trace upward (default: 2, max recommended: 3).
- `module` (optional): Module to search in. If omitted, determined automatically.

## Steps

### 1. Verify connection and resolve address

```
probe.exe "ping"
probe.exe "modules list"
```

Identify which module contains the target address. Record the module name and base address.

Confirm the address is a function start:

```
probe.exe "disasm <address> 5"
```

Look for a prologue (`push rbp`, `sub rsp`, etc.). If mid-function, search backwards for the start.

### 2. Find direct callers (Level 0 -> Level 1)

```
probe.exe "xref scan <address> <module> --type CALL --limit 15"
```

Record each caller:
- `call_site` -- address of the CALL instruction
- `enclosing_function` -- start of the function containing the call (from .pdata)
- The disassembled instruction

For each caller, grab a quick summary -- disassemble 3-4 instructions before the call to see argument setup:

```
probe.exe "disasm <call_site - 0x18> 8"
```

Note any string references in the argument setup (resolve `lea reg, [rip+...]` targets).

If **zero callers** found:
- Try `--type CALL,MOV` for indirect references
- Try scanning other modules
- The function may be virtual (called via vtable) -- check `rtti scan`
- Report the dead end and stop

### 3. Find callers of callers (Level 1 -> Level 2)

For each unique enclosing function from Level 1, repeat the xref scan:

```
probe.exe "xref scan <level1_function> <module> --type CALL --limit 10"
```

Limit to 10 callers per function to avoid combinatorial explosion. If a function has more than 10 callers, it is likely a utility function -- note it as "widely called" and don't trace further up from it.

Again, for each caller found, grab argument setup:

```
probe.exe "disasm <call_site - 0x18> 8"
```

### 4. Continue deeper (Level 2 -> Level 3, if requested)

If `depth` is 3, repeat for each Level 2 function. Apply the same limits:
- Max 10 callers per function
- Skip functions already seen (avoid cycles)
- Skip functions flagged as "widely called"

**Cycle detection:** Keep a set of all function addresses already visited. If an xref scan returns a function already in the set, mark it as `(recursive/cycle)` and don't trace further.

**Practical limits:** Beyond depth 3, the tree becomes unwieldy. Recommend stopping at depth 2-3 and going deeper only for specific branches of interest.

### 5. Identify function names at each level

For each function in the tree, try to identify it:

a. **Check for string references** (fast heuristic):
```
probe.exe "disasm func <function_address>"
```
Scan for `lea` instructions with RIP-relative operands. Resolve strings:
```
probe.exe "read_string <resolved_address>"
```

b. **Check the function store:**
```
probe.exe "functions at <function_address>"
```

c. **Check for labels** (if GUI bridge is running):
```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"store","store":"label","method":"getLabel","args":["0x<function_address>"]}'
```

d. **Check RTTI** if the function is at a vtable entry:
```
probe.exe "rtti find <partial_name>"
```

If no name is found, use the address as the label: `sub_7FF612345678`.

### 6. Build the reverse call tree

Assemble the collected data into a tree structure. Each node shows:
- Function name or address
- Module + offset
- Key argument info from the call site (what was passed)

```
target: sub_7FF612345678 (client.dll + 0x5678)
  "ProcessDamage" — refs: "damage_type"
===========================================================

sub_7FF612345678 (client.dll + 0x5678)
|
+-- [caller] sub_7FF612340000 (client.dll + 0x0000)
|   |  call site: +0x42, args: mov rcx,rsi / mov edx,[rbx+0x354]
|   |  refs: "GameRules::ProcessDamage"
|   |
|   +-- [caller] sub_7FF612338000 (client.dll - 0xD678)
|   |   |  call site: +0x1A, args: lea rcx,[rip+...] -> "game_rules"
|   |   |  refs: "OnTick"
|   |   |
|   |   +-- [caller] sub_7FF612330000 (widely called, 15+ callers)
|   |       (tick dispatcher — not traced further)
|   |
|   +-- [caller] sub_7FF61234A000 (client.dll + 0x4422)
|       |  call site: +0x8C, args: mov rcx,rdi / xor edx,edx
|       |  refs: "CombatSystem::Update"
|       |
|       +-- (no callers found — likely a virtual function)
|
+-- [caller] sub_7FF612342800 (client.dll + 0x2800)
|   |  call site: +0x42, args: mov rcx,[rbp+0x30] / mov edx,1
|   |  refs: "AI_TakeDamage"
|   |
|   +-- [caller] sub_7FF612350000 (client.dll + 0xA422)
|       |  call site: +0x64, args: mov rcx,r14 / lea rdx,[rip+...] -> "npc"
|       |  refs: "AI::Think"
|       |
|       +-- [caller] sub_7FF612360000 (client.dll + 0x1A422)
|           call site: +0x30, args: mov rcx,[rsp+0x40]
|           (no string refs)
|
+-- [caller] sub_7FF612346000 (client.dll + 0x0422)
    |  call site: +0x22, args: xor ecx,ecx / xor edx,edx
    |  (no string refs — possibly a test/debug caller)
    |
    +-- (no callers found)

Summary:
  Level 0 (target): 1 function
  Level 1 (direct callers): 3 functions
  Level 2 (indirect callers): 4 functions
  Total unique functions: 8
  Widely-called functions skipped: 1
```

### 7. Navigate in GUI (if bridge available)

Label all discovered functions and navigate to the target:

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"activity","status":"working","message":"Labeling call tree..."},
    {"action":"store","store":"label","method":"setLabel","args":["0x<target>","target: <name>"]},
    {"action":"store","store":"label","method":"setLabel","args":["0x<caller1>","caller L1: <name>"]},
    {"action":"store","store":"label","method":"setLabel","args":["0x<caller2>","caller L1: <name>"]},
    {"action":"store","store":"label","method":"setLabel","args":["0x<caller1_1>","caller L2: <name>"]},
    {"action":"navigate","tab":"disasm","address":"0x<target>"},
    {"action":"activity","status":"idle","message":"Done — traced N callers across M levels"}
  ]
}'
```

### 8. Report

Present the tree (as shown in step 6) followed by a summary:

```
Reverse Call Tree for sub_7FF612345678
=======================================

[tree as above]

Key Findings:
- Main call path: OnTick -> GameRules::ProcessDamage -> target
- AI path: AI::Think -> AI_TakeDamage -> target
- The target function is called from 3 direct callers
- Deepest chain: 3 levels (OnTick -> ProcessDamage -> target)
- "sub_7FF612330000" is a central dispatcher (15+ callers), likely a tick/update loop
```

## Handling large call trees

If a function has many callers at Level 1 (>8), prioritize:

1. **Functions with string references** -- these are easiest to name and understand
2. **Functions with few callers themselves** -- these are closer to the "real" entry points
3. **Functions in the same module** -- cross-module calls can be traced later if needed

Skip or summarize "utility" functions that are called from many places (allocators, logging, validation helpers). These create noise in the tree without adding understanding.

## Tips

- The reverse call tree reveals the "execution context" -- it answers "how does execution reach this function?"
- String references at higher levels in the tree often name the entire feature or subsystem
- If the top of the tree converges to one or two functions, those are the system entry points (tick handlers, event dispatchers, message pumps)
- Virtual function calls create "dead ends" in the tree -- the function has callers but they use `call [rax+offset]` which xref scan can't resolve
- Functions with zero callers at the top of the tree are likely virtual functions, callbacks, or thread entry points
- Depth 2 is usually enough to understand the context. Go to depth 3 only if the Level 2 functions are still anonymous
