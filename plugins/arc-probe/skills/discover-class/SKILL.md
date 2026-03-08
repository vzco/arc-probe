---
name: discover-class
description: Discover a C++ class by name via RTTI — map its vtable, disassemble key virtual functions, explore inheritance hierarchy, and label everything
argument-hint: <class_name> [module]
---

# /probe:discover-class

Given a class name (or partial name), use RTTI to find the class in the target process, map its vtable, disassemble key virtual functions, explore the inheritance hierarchy, find related classes, and build a complete class map with labels in the GUI.

## Arguments

- `class_name` (required): Class name or partial name to search for (e.g., "BaseEntity", "PlayerPawn", "Render")
- `module` (optional): Module to search in (e.g., "client.dll"). If omitted, searches all modules.

## Steps

### 1. Search for the class by name

```
probe.exe "rtti find <class_name> <module>"
```

This searches RTTI type descriptors for class names containing the search term. Returns:
- `name` — the demangled class name
- `vtable` — vtable address
- `module` — which module contains it

If **multiple matches** are found, present them all and let the user choose. Common pattern: searching for "Entity" might match `CEntityInstance`, `C_BaseEntity`, `CEntitySystem`, etc.

If **no matches** are found:
- Try a **shorter substring**: `"Entity"` instead of `"CBaseEntity"`
- Try without namespace: `"BaseEntity"` instead of `"Game::CBaseEntity"`
- The class might not have RTTI (compiled with `/GR-`). See "If RTTI is stripped" below.
- The module might not be loaded yet.

### 2. Get the full inheritance chain

```
probe.exe "rtti hierarchy <class_name> <module>"
```

This walks the RTTI Class Hierarchy Descriptor to find all base classes. Example output:
```
C_CitadelPlayerPawn
  <- C_BasePlayerPawn
    <- C_BaseModelEntity
      <- C_BaseEntity
        <- CEntityInstance
```

Record the full chain. Each base class contributes fields and virtual functions to the derived class. Fields from base classes appear at lower offsets.

### 3. Map the vtable

```
probe.exe "rtti vtable <class_name> <module>"
```

This reads the vtable entries — each entry is a pointer to a virtual function. The response includes:
- `vtable_address` — base of the vtable in memory
- `entries` — array of function pointers with disassembly previews

For each vtable entry, you get:
- `index` — vtable slot number (0, 1, 2, ...)
- `address` — function pointer value
- `preview` — first disassembled instruction

### 4. Categorize vtable entries

Scan through the vtable entries and categorize each function:

**Tiny functions (1-5 instructions) — analyze inline:**

```
probe.exe "disasm <vtable_entry_address> 8"
```

Recognize these patterns:

- **Destructor** (usually index 0 or 1):
  ```asm
  push rbx
  sub rsp, 0x20
  ...
  call operator_delete    ; or just deallocates and returns
  ```

- **Getter** (returns a field):
  ```asm
  mov eax, [rcx+0x354]   ; int32 getter at offset 0x354
  ret
  ```
  or
  ```asm
  movss xmm0, [rcx+0x40] ; float getter at offset 0x40
  ret
  ```

- **Boolean getter**:
  ```asm
  movzx eax, byte ptr [rcx+0x103]  ; bool at offset 0x103
  ret
  ```

- **Setter**:
  ```asm
  mov [rcx+0x354], edx   ; sets field at 0x354 from second arg
  ret
  ```

- **Type check / IsA**:
  ```asm
  xor eax, eax           ; return false
  ret
  ```
  or
  ```asm
  mov eax, 1             ; return true
  ret
  ```

- **Thunk** (redirects to another function):
  ```asm
  jmp <other_address>
  ```

- **Pure virtual stub** (should never be called):
  ```asm
  int3                   ; or call __purecall
  ```

**Large functions (>20 instructions) — deeper analysis for the most interesting ones:**

```
probe.exe "disasm func <vtable_entry_address>"
```

Look for string references, system calls, and field access patterns to understand the function's purpose.

### 5. Build the field map from vtable analysis

From the getters and setters found in step 4, compile a field map:

```
Known fields for C_BaseEntity:
  +0x103  bool     (getter at vf[12])
  +0x330  pointer  (getter at vf[18], likely GameSceneNode*)
  +0x350  int32    (getter at vf[22], max health?)
  +0x354  int32    (getter at vf[23], current health?)
  +0x35C  int32    (getter at vf[25], life state?)
  +0x3F3  uint8    (getter at vf[31], team number?)
```

### 6. Find related classes

Search for classes that share namespace or naming patterns:

```
probe.exe "rtti find <namespace_or_prefix> <module>"
```

For example, if the target class is `C_BaseEntity`, search for:
- `C_Base` — all base classes
- `CEntity` — entity system classes
- `CGame` — game system classes

For each related class, check if it inherits from the target class:

```
probe.exe "rtti hierarchy <related_class> <module>"
```

Build a tree of all related classes:

```
C_BaseEntity
  ├── C_BaseModelEntity
  │   ├── C_BasePlayerPawn
  │   │   └── C_CitadelPlayerPawn
  │   └── C_BaseFlex
  ├── C_BaseToggle
  └── C_BaseTrigger
```

### 7. Cross-reference the vtable (find object instances)

To find live instances of the class in memory, search for the vtable pointer. Every instance of the class has the vtable address as its first 8 bytes:

```
probe.exe "pattern <vtable_bytes_as_pattern> <module>"
```

Convert the vtable address to little-endian bytes. For example, vtable at `0x7FFB1A2B3C40`:
```
probe.exe "pattern 40 3C 2B 1A FB 7F 00 00"
```

Each match is a pointer to an instance of the class (or a subclass). This is useful for finding objects without knowing where they're stored.

**Note:** This scans the module's memory, which might not include heap allocations. Heap-allocated objects won't be found this way.

### 8. Explore a specific virtual function in depth

For the most interesting vtable entries identified in step 4, do a full function analysis:

```
probe.exe "disasm func <function_address>"
```

Look for:
- **String references** — `lea rcx, [rip+...]` followed by reading the string
- **Other vtable calls** — `mov rax, [rcx]; call [rax+0x??]` reveals interactions with other virtual classes
- **Field access patterns** — `mov eax, [rcx+offset]` reveals struct layout
- **System API calls** — imported functions reveal capabilities

Find what calls this virtual function:

```
probe.exe "xref scan <function_address> <module> --type CALL --limit 10"
```

### 9. Label everything in the GUI

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"activity","status":"working","message":"Labeling <class_name> class map..."},
    {"action":"store","store":"label","method":"setLabel","args":["0x<vtable_addr>","vtable: <class_name>"]},
    {"action":"store","store":"label","method":"setLabel","args":["0x<vf0_addr>","<class_name>::~<class_name> (destructor)"]},
    {"action":"store","store":"label","method":"setLabel","args":["0x<vf1_addr>","<class_name>::GetHealth (getter +0x354)"]},
    {"action":"store","store":"label","method":"setLabel","args":["0x<vf5_addr>","<class_name>::vf5 (large, refs \"error\")"]},
    {"action":"navigate","tab":"disasm","address":"0x<vtable_addr>"},
    {"action":"activity","status":"idle","message":"Done — mapped <class_name> with N vtable entries"}
  ]
}'
```

If an object instance was found, also create a struct in the GUI:

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"store","store":"struct","method":"createStruct","args":["<class_name>","0x<instance_addr>",1024]},
    {"action":"store","store":"struct","method":"addField","args":["<class_name>",0,"pointer","vtable"]},
    {"action":"store","store":"struct","method":"addField","args":["<class_name>",259,"bool","m_bFlag (+0x103)"]},
    {"action":"store","store":"struct","method":"addField","args":["<class_name>",816,"pointer","m_pGameSceneNode (+0x330)"]},
    {"action":"store","store":"struct","method":"addField","args":["<class_name>",852,"int32","m_iHealth (+0x354)"]},
    {"action":"navigate","tab":"structs"}
  ]
}'
```

Note: `addField` offsets are in **decimal** (0x103 = 259, 0x330 = 816, 0x354 = 852).

### 10. Report

```
Class Discovery: C_BaseEntity
==============================

Module: client.dll
Vtable: 0x7FFB1A2B3C40 (client.dll + 0x1E93C40, .rdata)

Inheritance:
  C_BaseEntity <- CEntityInstance <- IHandleEntity

Vtable (42 entries):
  [0]  0x7FFB0B234000  ~C_BaseEntity (destructor)
  [1]  0x7FFB0B234100  GetRefEHandle — returns [rcx+0x10]
  [2]  0x7FFB0B234200  IsAlive — movzx eax, byte [rcx+0x35C]; test/cmp
  [3]  0x7FFB0B234300  GetHealth — mov eax, [rcx+0x354]; ret
  [4]  0x7FFB0B234400  GetMaxHealth — mov eax, [rcx+0x350]; ret
  [5]  0x7FFB0B234500  TakeDamage — large (0x1A0 bytes), refs "damage"
  ...
  [18] 0x7FFB0B236000  GetGameSceneNode — mov rax, [rcx+0x330]; ret
  ...
  [31] 0x7FFB0B237000  GetTeamNumber — movzx eax, byte [rcx+0x3F3]; ret
  ...

Field Map (from vtable getters):
  +0x010  pointer  GetRefEHandle return (entity handle)
  +0x103  bool     IsAlive flag
  +0x330  pointer  GameSceneNode*
  +0x350  int32    MaxHealth
  +0x354  int32    Health
  +0x35C  int32    LifeState
  +0x3F3  uint8    TeamNum

Related Classes (same hierarchy):
  C_BaseModelEntity (58 vtable entries) <- C_BaseEntity
  C_BasePlayerPawn (89 vtable entries) <- C_BaseModelEntity
  C_CitadelPlayerPawn (112 vtable entries) <- C_BasePlayerPawn

Live Instances Found: 3
  0x055EA1C28000 (heap)
  0x055EA1C29000 (heap)
  0x055EA1C30000 (heap)
```

## If RTTI is stripped

Some binaries are compiled with `/GR-` which strips RTTI metadata. Signs:
- `rtti find` returns zero results for any search
- `rtti scan <module>` returns few or no types
- No `.?AV` strings in the module's .rdata

Fallback approaches:

1. **PE exports** — the class might be exported by name. Check `pe exports <module>` for class-related symbols.

2. **String references** — constructor/destructor functions often reference the class name in assert or log messages. Search for the class name as a string:
   ```
   probe.exe "strings find <class_name> <module>"
   ```

3. **Debug symbols** — if a PDB file is available (same directory as the binary, or symbol server), it contains full type information.

4. **Manual vtable discovery** — scan .rdata for pointer arrays where every entry points into .text. These are likely vtables. Disassemble entries to identify the class.

5. **Cross-reference from known code** — if you know a function that creates instances of the class, breakpoint it and read RCX after construction to get the vtable pointer, then work backwards to the class layout.

## Tips

- vtable index 0 is almost always the destructor (or a destructor wrapper)
- Getters and setters in the vtable directly reveal the struct field layout
- Inheritance means the base class's vtable entries come first — shared getters across related classes access the same offsets
- The `this` pointer is always in RCX for member functions (Microsoft x64 ABI)
- Virtual function calls pattern: `mov rax, [rcx]; call [rax+index*8]` — the index tells you which vtable slot is being called
- If a vtable entry is a thunk (`jmp <addr>`), follow the jump to find the real implementation
- Pure virtual functions (`__purecall`) indicate abstract classes — the derived class must override them
