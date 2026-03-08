---
name: analyze-vtable
description: Fully analyze a C++ virtual function table — detect params, find string refs, measure sizes, and label all entries
argument-hint: <class_name_or_vtable_addr> [module]
---

# /probe:analyze-vtable

Fully analyze a C++ virtual function table. For each virtual function entry, detect parameters, check for string references, measure the function size, and present a formatted vtable map. Works from either a class name (via RTTI) or a direct vtable address.

## Arguments

- `class_name_or_vtable_addr` (required): Either a class name / partial name to search via RTTI, or a hex address of the vtable directly.
- `module` (optional): Module to search in (e.g., "client.dll"). If omitted, searches all modules.

## Steps

### 1. Resolve the vtable address

**If the argument is a class name:**

```
probe.exe "rtti find <class_name> <module>"
```

This returns matching classes with their vtable addresses. If multiple matches:
- Present all matches and let the user choose
- Common ambiguity: searching "Entity" matches `CEntityInstance`, `C_BaseEntity`, `CEntitySystem`, etc.

If no matches:
- Try a shorter substring: "Entity" instead of "CBaseEntity"
- Try without prefix: "BaseEntity" instead of "C_BaseEntity"
- The module may not have RTTI compiled in (`/GR-`). See "If RTTI is unavailable" below.

Once you have the class, get the vtable:

```
probe.exe "rtti vtable <class_name> <module>"
```

Record:
- `vtable_address` -- base of the vtable in memory
- `entries` -- array of function pointers
- `class_name` -- resolved full class name

**If the argument is a hex address:**

Verify it looks like a vtable by reading the first few entries:

```
probe.exe "dump <address> 64"
```

Each 8-byte value should be a pointer into a `.text` section (executable code). Verify one entry:

```
probe.exe "disasm <first_entry> 3"
```

If it disassembles to valid instructions, it is a vtable. If not, the address may be wrong.

Get the class name from RTTI (vtable - 8 points to the Complete Object Locator):

```
probe.exe "rtti resolve <any_object_with_this_vtable>"
```

Or manually:
```
probe.exe "read_ptr <vtable_address - 8>"
```
Then follow the COL -> TypeDescriptor -> class name chain (see the `identify-class` skill).

### 2. Get the inheritance chain

```
probe.exe "rtti hierarchy <class_name> <module>"
```

Record the full inheritance chain. This tells you:
- How many vtable entries are inherited from base classes
- Which entries are overrides vs. new virtuals
- The total depth of the class hierarchy

### 3. Read all vtable entries

The `rtti vtable` command returns entries with previews. If you have a raw vtable address, read entries manually:

```
probe.exe "read_ptr <vtable_address>"
probe.exe "read_ptr <vtable_address + 0x8>"
probe.exe "read_ptr <vtable_address + 0x10>"
...
```

Continue reading until you hit a NULL pointer or a value that is not a valid code address. Vtable entries are contiguous 8-byte pointers.

**Detecting vtable end:** Read entries until:
- A NULL (0x0000000000000000) entry
- An entry that doesn't point into any module's `.text` section
- You've read 200+ entries (practical limit; most vtables are under 150)

### 4. Analyze each vtable entry

For each entry, perform a quick categorization by disassembling the first 8-12 instructions:

```
probe.exe "disasm <entry_address> 12"
```

**Categorize by pattern:**

**A. Destructor (usually index 0 or 1):**
```asm
push rbx
sub rsp, 0x20
mov rbx, rcx
; ... cleanup code ...
call operator_delete   ; or just returns after cleanup
```
Naming: `~ClassName()` or `ClassName::~ClassName`

**B. Getter (1-3 instructions):**
```asm
mov eax, [rcx+0x354]        ; int32 getter
ret
```
```asm
movss xmm0, [rcx+0x40]      ; float getter
ret
```
```asm
movzx eax, byte [rcx+0x103] ; bool getter
ret
```
```asm
mov rax, [rcx+0x330]         ; pointer getter
ret
```
Naming: `Get<Field>()` -- note the offset and inferred type.

**C. Setter (2-3 instructions):**
```asm
mov [rcx+0x354], edx         ; int32 setter
ret
```
Naming: `Set<Field>(value)` -- note the offset.

**D. Boolean return (type check / IsA):**
```asm
xor eax, eax                 ; return false
ret
```
```asm
mov eax, 1                   ; return true
ret
```
Naming: `IsA<Type>()` or `CanDoSomething()`

**E. Thunk (1 instruction):**
```asm
jmp <other_address>
```
Follow the jump to find the real function. Note: `<other_address> = thunk target`.

**F. Pure virtual (should never be called):**
```asm
call __purecall
```
or
```asm
int3
```
Naming: `(pure virtual)`

**G. Medium/Large function (>10 instructions):**

For these, do a deeper pass:

a. **Measure size:**
```
probe.exe "disasm func <entry_address>"
```
Count instructions and compute size = last instruction address - start + last instruction length.

b. **Detect parameters** (abbreviated version of detect-params skill):
Scan for first reads of RCX (already `this`), RDX, R8, R9, XMM0-XMM3. Count how many are parameters.

c. **Find string references:**
Scan the disassembly for `lea reg, [rip+displacement]` instructions. Resolve the target:
```
target = lea_address + instruction_length + displacement
```
```
probe.exe "read_string <target>"
```
If readable, this is a string reference that hints at the function's purpose.

d. **Note key patterns:**
- Calls to other vtable functions: `mov rax, [rcx]; call [rax+offset]`
- Calls to known APIs: `call <import_address>`
- Conditional branches that check fields: `cmp [rcx+0x??], 0` / `je`

### 5. Check for vtable overrides across the hierarchy

If the class inherits from a base class, many vtable entries will be the same as the base:

```
probe.exe "rtti vtable <base_class_name> <module>"
```

Compare entries at the same index. If a derived class has a different function pointer at index N, that entry is **overridden**.

Mark each entry as:
- `inherited` -- same pointer as base class
- `overridden` -- different pointer, same slot
- `new` -- index beyond the base class vtable size

### 6. Present the formatted vtable

```
Vtable for CTraceManager (client.dll + 0x1E93C40)
===================================================
Class: CTraceManager <- CBaseObject <- CEntityInstance
Module: client.dll
Entries: 34

 Idx  Address              Size   Params  Type        Name / Notes
 ---  -------------------  -----  ------  ----------  ---------------------------------
 [0]  0x7FF812340000       0x80   1       destructor  ~CTraceManager
 [1]  0x7FF812340100       0x04   1       getter      GetRefEHandle — returns [rcx+0x10]
 [2]  0x7FF812340200       0x340  3       override    ProcessTrace — refs: "trace_origin", "trace_dir"
 [3]  0x7FF812340300       0x08   1       getter      IsTraceEnabled — bool [rcx+0x48]
 [4]  0x7FF812340400       0x0A   2       setter      SetTraceEnabled — [rcx+0x48] = dl
 [5]  0x7FF812340500       0x1A0  4       new         ComputeRaycast — refs: "ray_hit", "surface_normal"
 [6]  0x7FF812340600       0x04   0       bool        AlwaysTrue — mov eax, 1; ret
 [7]  0x7FF812340700       -      -       thunk       -> 0x7FF812348000
 [8]  0x7FF812340800       -      -       pure        (pure virtual)
 [9]  0x7FF812340900       0x60   2       inherited   BaseObject::GetBounds — refs: "bounds"
 ...
 [33] 0x7FF812343000       0x120  3       new         FlushResults — refs: "trace_results"

Field Map (from getters/setters):
  +0x010  pointer  GetRefEHandle (vf[1])
  +0x048  bool     IsTraceEnabled (vf[3]), SetTraceEnabled (vf[4])
  +0x0F0  int32    GetTraceCount (vf[12])
  +0x100  float    GetMaxRange (vf[15])

String References Found:
  vf[2]:  "trace_origin", "trace_dir"
  vf[5]:  "ray_hit", "surface_normal"
  vf[9]:  "bounds"
  vf[33]: "trace_results"
```

### 7. Label in the GUI (if bridge available)

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"activity","status":"working","message":"Labeling CTraceManager vtable..."},
    {"action":"store","store":"label","method":"setLabel","args":["0x<vtable_addr>","vtable: CTraceManager"]},
    {"action":"store","store":"label","method":"setLabel","args":["0x<vf0>","CTraceManager::~CTraceManager"]},
    {"action":"store","store":"label","method":"setLabel","args":["0x<vf1>","CTraceManager::GetRefEHandle"]},
    {"action":"store","store":"label","method":"setLabel","args":["0x<vf2>","CTraceManager::ProcessTrace"]},
    {"action":"store","store":"label","method":"setLabel","args":["0x<vf3>","CTraceManager::IsTraceEnabled"]},
    {"action":"store","store":"label","method":"setLabel","args":["0x<vf5>","CTraceManager::ComputeRaycast"]},
    {"action":"navigate","tab":"disasm","address":"0x<vtable_addr>"},
    {"action":"activity","status":"idle","message":"Done — mapped CTraceManager with 34 vtable entries"}
  ]
}'
```

### 8. Report

Present the formatted vtable (step 6), followed by a summary:

```
Summary:
  Total entries: 34
  Destructors: 1
  Getters: 8 (reveals 8 struct fields)
  Setters: 3
  Boolean returns: 4 (type checks)
  Large functions: 6 (key logic)
  Thunks: 2
  Pure virtual: 1
  Inherited (unchanged from base): 9
  Overridden: 6
  New (added by this class): 19

Key functions to investigate further:
  vf[2]  ProcessTrace (0x340 bytes, 3 params, multiple string refs)
  vf[5]  ComputeRaycast (0x1A0 bytes, 4 params, core logic)
  vf[33] FlushResults (0x120 bytes, 3 params)
```

## If RTTI is unavailable

If the target binary has no RTTI (`rtti find` returns nothing):

1. **Start from a known vtable address** -- if you have an object pointer, read the first 8 bytes to get the vtable.

2. **Manual vtable discovery** -- scan `.rdata` for pointer arrays where every entry points into `.text`:
   ```
   probe.exe "pe sections <module>"
   ```
   Find `.rdata` base and size. Dump regions and look for consecutive code pointers.

3. **Work backwards from a known virtual call** -- if you see `mov rax, [rcx]; call [rax+0x18]`, the vtable is at `[rcx]` and the function is at slot 3 (0x18 / 8).

4. **No class name** -- without RTTI, name the vtable by its address: `vtable_7FF812345678`. Name functions by their behavior discovered during analysis.

## Tips

- Vtable index 0 is almost always the destructor (scalar deleting destructor)
- Index 1 is often a second destructor variant (base class destructor) or a simple getter
- Getters and setters cluster together and directly reveal the struct layout
- Base class entries come first -- if `CBaseEntity` has 42 entries, then `C_BasePlayerPawn` entries start at index 42
- Pure virtual entries indicate abstract classes -- the derived class must override them
- If two classes share the same vtable entry (same function pointer), the function is inherited
- Thunks (`jmp <addr>`) are compiler-generated wrappers -- always follow the jump
- Very large vtables (>100 entries) are common in game engines -- be prepared to process many entries
- Focus analysis time on the large, complex functions -- getters and setters are fully characterized by their single instruction
