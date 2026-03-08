---
name: investigate-module
description: Master orchestration — full end-to-end investigation of an unknown module using all ARC Probe capabilities together (strings, RTTI, functions, xrefs, structs, bookmarks, GUI bridge)
argument-hint: <module>
---

# /probe:investigate-module

Full end-to-end investigation of an unknown module. This is the "master orchestration" skill — it ties together reconnaissance, string scanning, RTTI class discovery, cross-reference analysis, structure mapping, and GUI visualization into a single coherent workflow.

Use this when you're dropped into a process and need to understand what a module does from scratch. By the end, you'll have named functions, mapped classes, bookmarked key addresses, and populated the GUI with all findings.

**Key principle:** Use probe commands to GET data, use store commands to VISUALIZE it in the GUI. The bridge returns `{"ok":true,"data":null}` for store mutations (fire-and-forget). For actual data, use `{"action":"probe","command":"..."}` which returns the probe's JSON response.

## Arguments

- `module` (required): Module name (e.g., "client.dll", "engine2.dll", "target.exe")

## Phase 1: Reconnaissance

**Goal:** Understand the module's footprint — size, sections, API surface, dependencies.

### 1.1 Find and verify the module

```
probe.exe "modules list"
```

Find the target module. Record:
- **Base address**
- **Size**
- **Full path**

If the module is not found:
- Try case variations (`Client.dll` vs `client.dll`)
- Try with/without extension
- The module may not be loaded yet (game on menu screen, DLL lazy-loaded)
- Report available modules and stop

### 1.2 Get PE structure overview

```
probe.exe "modules info <module>"
probe.exe "pe sections <module>"
```

Record the sections — this tells you where code vs data lives:
- `.text` — executable code (functions live here)
- `.rdata` — read-only data (strings, vtables, RTTI metadata)
- `.data` — writable globals
- `.pdata` — exception/unwind data (function boundaries)

Note the `.text` section size — this is the total code area. The ratio of `.text` to `.rdata` tells you how code-heavy vs data-heavy the module is.

### 1.3 Analyze the export table

```
probe.exe "pe exports <module>"
```

Exports are the module's public API. Group by naming pattern:
- `CreateInterface` — Source 2 interface factory (critical entry point)
- `DllMain` / `DllEntryPoint` — initialization
- `Init*`, `Create*`, `Register*` — initialization functions
- `Get*`, `Find*`, `Lookup*` — accessor functions
- `Process*`, `Handle*`, `On*` — event handlers
- `Shutdown*`, `Destroy*`, `Release*` — cleanup

If exports are present, bookmark the most important ones immediately:

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"activity","status":"working","message":"Phase 1: Reconnaissance..."},
    {"action":"store","store":"bookmarks","method":"addBookmark","args":["0x<export1_addr>","export: CreateInterface","blue","<module>"]},
    {"action":"store","store":"bookmarks","method":"addBookmark","args":["0x<export2_addr>","export: DllMain","blue","<module>"]}
  ]
}'
```

### 1.4 Analyze the import table

```
probe.exe "pe imports <module>"
```

Imports reveal what the module CAN DO. Group by DLL:

| Import DLL | Reveals |
|---|---|
| `kernel32.dll` | File I/O, memory, threading, process management |
| `user32.dll` | Window management, input handling, message pumps |
| `ws2_32.dll` / `winhttp.dll` | Networking (sockets, HTTP) |
| `d3d11.dll` / `dxgi.dll` | Graphics rendering |
| `ntdll.dll` | Low-level OS functions, syscalls |
| `advapi32.dll` | Registry, security, services |
| `crypt32.dll` / `bcrypt.dll` | Cryptography |
| `ole32.dll` / `oleaut32.dll` | COM, OLE automation |

This gives you a capability profile of the module before diving into code.

### 1.5 Discover all functions and populate GUI

```
probe.exe "functions discover <module> --limit 5000"
```

Then populate the GUI Functions tab:

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"store","store":"functions","method":"discoverFunctions","args":["<module>"]},
    {"action":"navigate","tab":"functions"}
  ]
}'
```

Record:
- Total function count (from `.pdata`)
- Named function count (exports + RTTI)
- Unnamed function count (these will be identified in later phases)

**Decision point:** Module size determines the depth of subsequent phases.

| Module Size | Approach |
|---|---|
| Small (<1 MB) | Full analysis — scan everything, trace every string, map every class |
| Medium (1-20 MB) | Targeted analysis — scan strings/RTTI, trace top 10-20 strings |
| Large (>20 MB) | Focused analysis — search for specific strings/classes, skip full scans |


## Phase 2: String Discovery

**Goal:** Find all printable strings and identify which ones reveal function purpose.

### 2.1 Scan for strings

```
probe.exe "strings scan <module>"
```

Populate the GUI Strings sub-tab:

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"activity","status":"working","message":"Phase 2: Scanning strings..."},
    {"action":"store","store":"strings","method":"scanStrings","args":["<module>"]}
  ]
}'
```

### 2.2 Categorize interesting strings

Scan the results for high-value patterns:

| Pattern | Why It Matters | Example |
|---|---|---|
| `"ClassName::MethodName"` | Direct function identification (assert/debug strings) | `"CBaseEntity::TakeDamage"` |
| `"Failed..."`, `"Error..."`, `"Invalid..."` | Error handling functions | `"Failed to initialize %s"` |
| `"[TAG] ..."`, `"DEBUG: ..."` | Logging infrastructure | `"[NET] Connection lost"` |
| `"%s"`, `"%d"`, `"%f"` | Format strings reveal parameter types | `"Player %s (id=%d) spawned"` |
| `".cfg"`, `".json"`, `".xml"` | File loading/parsing functions | `"configs/game.json"` |
| `"http://"`, `"ws://"` | Network endpoints | `"http://api.example.com"` |
| UI text (buttons, labels) | UI handler functions | `"Start Game"`, `"Settings"` |
| SQL queries | Database access | `"SELECT * FROM players"` |
| Registry paths | Configuration storage | `"SOFTWARE\\Company\\App"` |

### 2.3 Trace key strings to functions

For the 5-10 most interesting strings, find what code references them:

```
probe.exe "strings find <interesting_text> <module>"
```

Note the string address, then find code references:

```
probe.exe "strings xref <string_address> <module>"
```

Each xref gives you a code address (LEA instruction) inside a function. Disassemble around it:

```
probe.exe "disasm <xref_address - 0x20> 15"
```

Identify the enclosing function (look for prologue above the xref). The string context reveals the function's purpose:

- `lea rcx, "CBaseEntity::TakeDamage"` then `call AssertFailed` --> this IS `CBaseEntity::TakeDamage`
- `lea rcx, "Failed to connect to %s"` then `call LogError` --> this is a network connection function
- `lea rcx, "HP: %d / %d"` then `call DrawText` --> this is a health bar renderer

### 2.4 Bookmark interesting strings

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"store","store":"bookmarks","method":"addBookmark","args":["0x<string1_addr>","str: \"CBaseEntity::TakeDamage\"","green","<module>"]},
    {"action":"store","store":"bookmarks","method":"addBookmark","args":["0x<string2_addr>","str: \"Failed to connect\"","green","<module>"]},
    {"action":"store","store":"label","method":"setLabel","args":["0x<func1_addr>","CBaseEntity::TakeDamage (from assert string)"]},
    {"action":"store","store":"label","method":"setLabel","args":["0x<func2_addr>","NetworkConnect (refs: \"Failed to connect\")"]},
    {"action":"store","store":"functions","method":"setFunctionName","args":["0x<func1_addr>","CBaseEntity::TakeDamage"]},
    {"action":"store","store":"functions","method":"setFunctionName","args":["0x<func2_addr>","NetworkConnect"]}
  ]
}'
```

**Color convention for bookmarks:**
- `"blue"` — functions, exports, entry points
- `"green"` — data, strings, constants
- `"orange"` — user-discovered / interesting / needs review
- `"red"` — dangerous / write targets / hooks


## Phase 3: RTTI Class Discovery

**Goal:** Map all C++ classes with virtual functions, understand inheritance hierarchies, and identify key classes.

### 3.1 Scan for RTTI classes

```
probe.exe "rtti scan <module>"
```

Populate the GUI Classes sub-tab:

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"activity","status":"working","message":"Phase 3: Scanning RTTI classes..."},
    {"action":"store","store":"rtti","method":"scanModule","args":["<module>"]}
  ]
}'
```

### 3.2 Categorize classes by role

Group the discovered classes by naming patterns:

| Pattern | Role | Priority |
|---|---|---|
| `*Manager`, `*System` | Singleton managers — central control points | High |
| `*Controller`, `*Handler` | Event/input processing | High |
| `*Factory`, `*Builder` | Object creation | Medium |
| `*Component` | Component-based architecture pieces | Medium |
| `C_*`, `CBase*` | Game entity classes | High (for games) |
| `I*` (prefix) | Interface/abstract base classes | Medium |
| `*Renderer`, `*View` | Visual/UI classes | Medium |
| `*Cache`, `*Pool` | Resource management | Low |
| `*Listener`, `*Observer` | Event observation | Low |

### 3.3 Explore key class hierarchies

For the 3-5 most important classes (Managers, Controllers, main entity types):

```
probe.exe "rtti hierarchy <class_name> <module>"
```

Load the hierarchy into the GUI:

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"store","store":"rtti","method":"loadHierarchy","args":["<class_name>","<module>"]}'
```

### 3.4 Map vtables for key classes

For each important class:

```
probe.exe "rtti vtable <class_name> <module>"
```

Load the vtable into the GUI:

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"store","store":"rtti","method":"loadVtable","args":["<class_name>","<module>"]}'
```

### 3.5 Analyze vtable entries

For each vtable, categorize entries by disassembling the first few instructions:

```
probe.exe "disasm <vtable_entry_addr> 8"
```

Recognize these patterns:

- **Destructor** (index 0-1): complex function with `call operator_delete`
- **Getter**: `mov eax, [rcx+0x??]; ret` — reveals a struct field at that offset
- **Setter**: `mov [rcx+0x??], edx; ret` — reveals a writable field
- **Boolean getter**: `movzx eax, byte ptr [rcx+0x??]; ret`
- **Type check**: `xor eax, eax; ret` (returns false) or `mov eax, 1; ret` (returns true)
- **Thunk**: `jmp <other_address>` — follow the jump
- **Pure virtual**: `int3` or `call __purecall` — abstract, must be overridden

Name functions based on vtable position + class name:

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"store","store":"bookmarks","method":"addBookmark","args":["0x<vtable_addr>","vtable: <ClassName>","blue","<module>"]},
    {"action":"store","store":"functions","method":"setFunctionName","args":["0x<vf0_addr>","<ClassName>::~<ClassName>"]},
    {"action":"store","store":"functions","method":"setFunctionName","args":["0x<vf3_addr>","<ClassName>::GetHealth (getter +0x354)"]},
    {"action":"store","store":"functions","method":"setFunctionName","args":["0x<vf5_addr>","<ClassName>::IsAlive (bool +0x103)"]},
    {"action":"store","store":"label","method":"setLabel","args":["0x<vtable_addr>","vtable: <ClassName>"]}
  ]
}'
```

### 3.6 Build a field map from getters/setters

Compile all getter/setter findings into a field map per class:

```
Known fields for ClassName:
  +0x000  pointer  vtable
  +0x103  bool     IsAlive (vf[5])
  +0x330  pointer  GameSceneNode* (vf[18])
  +0x350  int32    MaxHealth (vf[22])
  +0x354  int32    Health (vf[23])
  +0x3F3  uint8    TeamNum (vf[31])
```

This field map feeds directly into Phase 5 (Structure Mapping).


## Phase 4: Cross-Reference Analysis

**Goal:** Build a call graph understanding by tracing connections between discovered functions.

### 4.1 Pick key functions to analyze

From Phases 2-3, you should have a list of named functions. Prioritize:
1. Functions found via string references (Phase 2) — you know their purpose
2. Large vtable entries from important classes (Phase 3) — core logic
3. Exported functions (Phase 1) — the module's public API

### 4.2 Full disassembly of each key function

```
probe.exe "disasm func <function_addr>"
```

For each function, note:
- **Size** (instruction count, byte length)
- **Complexity** (branches, loops, nested calls)
- **String references** (LEA instructions pointing to .rdata)
- **CALL instructions** (callees)
- **Field accesses** (`[rcx+offset]` patterns)

### 4.3 Find callers (who calls this function)

```
probe.exe "xref scan <function_addr> <module> --type CALL --limit 20"
```

Load xrefs into the GUI:

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"store","store":"xrefs","method":"loadXrefs","args":["0x<function_addr>"]}'
```

If zero callers found:
- The function may be called **indirectly** via function pointer or vtable
- Try `--type CALL,MOV` to catch indirect references
- Try scanning other modules
- The function may only be called from JIT-compiled code

### 4.4 Find callees (what this function calls)

```
probe.exe "xref scan <function_addr> <module> --callees --limit 30"
```

For interesting callees, quick-identify them:

```
probe.exe "disasm <callee_addr> 5"
```

Look for getters, known API functions, or functions with string references.

### 4.5 Build the call graph

Connect the findings into a graph:

```
GameWorld::SpawnEntity
  -> CEntitySystem::InitEntity (refs: "Failed to initialize entity")
    -> EntityValidator::Check
    -> CBaseEntity::SetupFields
      -> FieldParser::ParseJSON (refs: "configs/entities.json")
  -> CBaseEntity::Activate
    -> CBaseEntity::TakeDamage (refs: assert string)
```

### 4.6 Label call graph in GUI

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"activity","status":"working","message":"Phase 4: Labeling call graph..."},
    {"action":"store","store":"functions","method":"setFunctionName","args":["0x<addr1>","GameWorld::SpawnEntity"]},
    {"action":"store","store":"functions","method":"setFunctionComment","args":["0x<addr1>","Caller of InitEntity. 3 callees."]},
    {"action":"store","store":"functions","method":"setFunctionName","args":["0x<addr2>","CEntitySystem::InitEntity"]},
    {"action":"store","store":"functions","method":"setFunctionComment","args":["0x<addr2>","Refs: \"Failed to initialize entity\". Called by SpawnEntity."]},
    {"action":"store","store":"label","method":"setLabel","args":["0x<addr1>","GameWorld::SpawnEntity"]},
    {"action":"store","store":"label","method":"setLabel","args":["0x<addr2>","CEntitySystem::InitEntity"]},
    {"action":"activity","status":"idle","message":"Phase 4 complete"}
  ]
}'
```

Limit batch size to ~50 entries per request.


## Phase 5: Structure Mapping

**Goal:** For classes with interesting data, map their struct layout and create interactive structs in the GUI.

### 5.1 Find live instances

To create a struct in the GUI, you need a base address — a live instance of the class. Several approaches:

**From a known pointer:** If earlier phases revealed a global pointer to a manager/system object:

```
probe.exe "read_ptr <global_pointer_addr>"
```

**From vtable pattern scan:** Search for the vtable pointer (first 8 bytes of every instance):

```
probe.exe "pattern <vtable_addr_as_LE_bytes>"
```

Convert the vtable address to little-endian bytes. For `0x7FFB1A2B3C40`:
```
probe.exe "pattern 40 3C 2B 1A FB 7F 00 00"
```

**From known entity system:** If the target is a game, use entity list traversal.

**From breakpoints:** Set a breakpoint on a constructor or getter, trigger it, read RCX:

```
probe.exe "bp set <constructor_addr>"
# ... trigger the object creation ...
probe.exe "bp log 0"
# RCX from the log = this pointer = object instance
probe.exe "bp remove 0"
```

### 5.2 Hex dump the instance

```
probe.exe "dump <instance_addr> 256"
```

Look for:
- **Pointer-like values** — 8-byte aligned, in the `0x7FF...` range (code) or `0x000...` high range (heap)
- **Small integers** — health, team, state values
- **Float values** — positions, angles, scales
- **Zero regions** — padding or uninitialized fields
- **String pointers** — follow them with `read_string`

### 5.3 Validate fields with typed reads

For each candidate field:

```
probe.exe "read_ptr <instance + pointer_offset>"
probe.exe "read_int <instance + int_offset>"
probe.exe "read_float <instance + float_offset>"
probe.exe "read_string <instance + string_ptr_offset>"
```

Cross-reference with the field map from Phase 3.6 — the getter/setter offsets should match.

### 5.4 Create struct in GUI

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"activity","status":"working","message":"Phase 5: Building struct <ClassName>..."},
    {"action":"store","store":"struct","method":"createStruct","args":["<ClassName>","0x<instance_addr>",1024]},
    {"action":"store","store":"struct","method":"addField","args":["<ClassName>",0,"pointer","vtable"]},
    {"action":"store","store":"struct","method":"addField","args":["<ClassName>",16,"pointer","m_pUnknown (+0x10)"]},
    {"action":"store","store":"struct","method":"addField","args":["<ClassName>",259,"bool","m_bIsAlive (+0x103)"]},
    {"action":"store","store":"struct","method":"addField","args":["<ClassName>",816,"pointer","m_pGameSceneNode (+0x330)"]},
    {"action":"store","store":"struct","method":"addField","args":["<ClassName>",848,"int32","m_iMaxHealth (+0x350)"]},
    {"action":"store","store":"struct","method":"addField","args":["<ClassName>",852,"int32","m_iHealth (+0x354)"]},
    {"action":"store","store":"struct","method":"addField","args":["<ClassName>",860,"int32","m_lifeState (+0x35C)"]},
    {"action":"store","store":"struct","method":"addField","args":["<ClassName>",1011,"uint8","m_iTeamNum (+0x3F3)"]},
    {"action":"navigate","tab":"structs"},
    {"action":"activity","status":"idle","message":"Struct <ClassName> created with N fields"}
  ]
}'
```

**Important:** `addField` offsets are in **decimal**, not hex. Convert: `0x103 = 259`, `0x330 = 816`, `0x354 = 852`, etc.

### 5.5 Follow pointer fields

For each pointer field in the struct, read where it points and recursively explore:

```
probe.exe "read_ptr <instance + pointer_offset>"
probe.exe "dump <pointed_addr> 128"
probe.exe "rtti resolve <pointed_addr>"
```

If RTTI resolves, you know the pointed-to class. Create a child struct if it's interesting.

### 5.6 Label addresses

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"store","store":"label","method":"setLabel","args":["0x<instance_addr>","instance: <ClassName>"]},
    {"action":"store","store":"label","method":"setLabel","args":["0x<child_obj_addr>","instance: GameSceneNode (owned by <ClassName>)"]},
    {"action":"store","store":"bookmarks","method":"addBookmark","args":["0x<instance_addr>","live instance: <ClassName>","orange","<module>"]}
  ]
}'
```


## Phase 6: Documentation and Summary

**Goal:** Bookmark all key findings, name all discovered functions, and produce a comprehensive report.

### 6.1 Final bookmarking pass

Ensure all key addresses are bookmarked with consistent color coding:

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"activity","status":"working","message":"Phase 6: Documenting findings..."},
    {"action":"store","store":"bookmarks","method":"addBookmark","args":["0x<export1>","export: CreateInterface","blue","<module>"]},
    {"action":"store","store":"bookmarks","method":"addBookmark","args":["0x<vtable1>","vtable: CEntitySystem","blue","<module>"]},
    {"action":"store","store":"bookmarks","method":"addBookmark","args":["0x<func1>","fn: CEntitySystem::InitEntity","blue","<module>"]},
    {"action":"store","store":"bookmarks","method":"addBookmark","args":["0x<string1>","str: \"Failed to initialize\"","green","<module>"]},
    {"action":"store","store":"bookmarks","method":"addBookmark","args":["0x<instance1>","obj: CEntitySystem instance","orange","<module>"]},
    {"action":"store","store":"bookmarks","method":"addBookmark","args":["0x<hook_target>","CAUTION: hooked function","red","<module>"]},
    {"action":"activity","status":"idle","message":"Investigation complete"}
  ]
}'
```

### 6.2 Generate signatures for key functions

For each important discovered function:

```
probe.exe "sig <function_addr> 32 <module>"
```

Test uniqueness:

```
probe.exe "pattern <signature> <module>"
```

If not unique (>1 match), try longer:

```
probe.exe "sig <function_addr> 64 <module>"
```

Record signatures so functions can be re-found after binary updates.

### 6.3 Comment functions with context

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"store","store":"functions","method":"setFunctionComment","args":["0x<addr1>","Initializes entity system. Refs: \"Failed to initialize\". Called by GameWorld::SpawnEntity. Sig: 48 89 5C 24 08 57 48 83 EC 20..."]},
    {"action":"store","store":"functions","method":"setFunctionComment","args":["0x<addr2>","Health getter. Returns [rcx+0x354]. vtable[23] of C_BaseEntity."]}
  ]
}'
```

### 6.4 Final report

Present a comprehensive structured summary:

```
Module Investigation: client.dll
=================================

Base:     0x7FFB0A8D0000
Size:     0x3E00000 (62 MB)
Path:     C:\Program Files\Steam\steamapps\common\Game\bin\win64\client.dll

Sections:
  .text    0x001000  0x2800000  (40 MB code)
  .rdata   0x2801000 0x0F00000  (15 MB read-only data)
  .data    0x3701000 0x0100000  (1 MB writable data)
  .pdata   0x3801000 0x0300000  (3 MB exception data)

Capability Profile (from imports):
  - Networking (ws2_32.dll) — socket-based communication
  - Graphics (d3d11.dll) — rendering
  - File I/O (kernel32.dll) — file loading
  - Threading (kernel32.dll) — multi-threaded

Functions: 98,432 total (.pdata)
  47 exports (public API)
  2,841 RTTI virtual functions across 412 classes
  37 functions named via string references
  ~95,000 unnamed functions

Key Exports:
  0x7FFB0A8D1000  CreateInterface
  0x7FFB0A8D2000  DllMain

Top Classes (RTTI):
  CEntitySystem (31 vtable entries)
    - manages entity lifecycle
    - vtable: 0x7FFB0C8D4000
  C_BaseEntity (42 vtable entries)
    - base class for all game entities
    - inheritance: C_BaseEntity <- CEntityInstance
    - vtable: 0x7FFB0C8D5000
    - field map: +0x330 pGameSceneNode, +0x354 health, +0x3F3 team
  C_BasePlayerPawn (89 vtable entries)
    - player character
    - inheritance: <- C_BaseEntity <- CEntityInstance
    - vtable: 0x7FFB0C8D6000

Key Functions (from string + xref analysis):
  0x7FFB0B1A4500  CBaseEntity::TakeDamage — refs assert string, sig: 48 89 5C...
  0x7FFB0B2E7800  CEntitySystem::InitEntity — refs "Failed to initialize", sig: 40 53 48...
  0x7FFB0B3C1200  NetworkConnect — refs "Connecting to %s:%d", sig: 48 89 74...

Structs Created:
  C_BaseEntity at 0x055EA1C28000 — 8 fields mapped (+0x0 vtable, +0x330 scene node, +0x354 health, ...)
  CEntitySystem at 0x055EA2D10000 — 5 fields mapped

Signatures (for binary update resilience):
  CBaseEntity::TakeDamage    48 89 5C 24 08 57 48 83 EC 20 48 8B D9 E8 ?? ?? ?? ??
  CEntitySystem::InitEntity  40 53 48 83 EC 20 48 8B D9 48 8B 49 08 ?? ?? ?? ??
  NetworkConnect             48 89 74 24 10 57 48 83 EC 30 48 8B FA ?? ?? ?? ??

Open Questions:
  - Function at 0x7FFB0B4A0000 (large, ~0x800 bytes, no string refs) — needs hwbp tracing
  - Unknown class "CSecretManager" — no instances found, constructor not identified
  - Import of ws2_32!connect but no visible connection string — may use encrypted config
```


## Adapting to Module Size

### Small modules (<1 MB)

Fully analyzable. Do everything:
- `functions discover` with no limit
- `strings scan` produces a manageable list
- Trace EVERY interesting string to its function
- Map EVERY RTTI class hierarchy and vtable
- Disassemble the top 20 functions fully
- Create structs for all discovered classes

### Medium modules (1-20 MB)

Targeted analysis:
- `functions discover --limit 5000`
- `strings scan` then focus on the most interesting 20-30 strings
- RTTI scan all classes but only deep-dive the top 5-10
- Xref analysis on the top 10 functions
- Create structs only for classes where you found live instances

### Large modules (>20 MB)

Focused analysis — don't try to catalog everything:
- Skip full `functions discover` with pdata (use `--exports --rtti` only)
- Use `strings find <specific_text>` instead of `strings scan` (full scan takes too long)
- Use `rtti find <partial_name>` to search for specific classes
- Limit xref scans — scanning a 40MB .text section takes several seconds per scan
- Focus on one subsystem at a time


## Decision Trees

### "I found a string but no xrefs"

```
strings xref returned 0 results?
  |
  +-- Try broader xref: xref scan <addr> <module> --type LEA,MOV --limit 20
  |     |
  |     +-- Found results? --> Analyze those code locations
  |     +-- Still nothing?
  |           |
  |           +-- Is the string in .data (writable)?
  |           |     |
  |           |     +-- Yes: Runtime-constructed string. Set hwbp on the string address
  |           |     |     to catch what writes it: hwbp set <addr> r 8
  |           |     +-- No (.rdata): The reference may be via a pointer table.
  |           |           Pattern scan for the address bytes in .rdata
  |           |
  |           +-- The string may be in a different module. Try without module filter.
```

### "I found an RTTI class but no instances"

```
rtti vtable found, but pattern scan for vtable bytes returns 0?
  |
  +-- Object may be heap-allocated (pattern scan only covers loaded modules)
  |     --> Set breakpoint on the constructor (find via vtable[0] destructor cross-refs)
  |     --> Trigger object creation, read RCX from bp log
  |
  +-- Class may be abstract (pure virtual entries in vtable)
  |     --> Look for derived classes: rtti find <basename>
  |     --> Search for instances of derived classes instead
  |
  +-- Class may not be instantiated yet
        --> Try again after triggering more application behavior
```

### "I found a function but don't know what it does"

```
No string references? No RTTI context?
  |
  +-- Check callers: xref scan <addr> <module> --type CALL
  |     |
  |     +-- Found callers? --> Analyze the callers (they may have strings)
  |     +-- No callers? --> Likely called indirectly (vtable or function pointer)
  |           --> Check if the address appears in any vtable: rtti scan <module>
  |
  +-- Check callees: xref scan <addr> <module> --callees
  |     |
  |     +-- Callees have strings? --> The function's purpose relates to its callees
  |     +-- Callees are all system API? --> Function is a wrapper/utility
  |
  +-- Set a breakpoint and observe: bp set <addr>
  |     --> Trigger various actions in the application
  |     --> bp log to see when it fires and with what arguments
  |
  +-- Hardware breakpoint on data it accesses:
        --> Find a field offset from the disassembly ([rcx+0x??])
        --> hwbp set <known_instance + offset> w 4
        --> See what triggers the write
```


## Example: Full Batch for Phase 1+2+3 Setup

This single batch populates the GUI with all three discovery scans at once:

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"activity","status":"working","message":"Investigating <module>: discovering functions, strings, classes..."},
    {"action":"store","store":"functions","method":"discoverFunctions","args":["<module>"]},
    {"action":"store","store":"strings","method":"scanStrings","args":["<module>"]},
    {"action":"store","store":"rtti","method":"scanModule","args":["<module>"]},
    {"action":"navigate","tab":"functions"},
    {"action":"activity","status":"idle","message":"Discovery complete — review Functions tab"}
  ]
}'
```

After this batch completes, the GUI has all three tabs populated and ready for interactive exploration. The LLM can then proceed with targeted analysis in Phases 2-5 based on what was discovered.


## Tips

- **Start broad, narrow down.** Phase 1 gives you the big picture. Each subsequent phase drills deeper into specific areas of interest.
- **Strings are your best friend.** In an unknown binary, string references are the most reliable and fastest way to understand function purpose. Always prioritize string analysis.
- **RTTI class names are the second-best friend.** They reveal the module's object model and class hierarchy without any disassembly.
- **Exports + imports = capability profile.** Before reading a single instruction, you can understand what a module does from its API surface and dependencies.
- **Cross-reference analysis connects the dots.** Individual functions are points; xrefs are the lines that connect them into a graph. The graph reveals the system architecture.
- **Label as you go.** Every address you understand should be labeled immediately. Don't wait until the end — you'll forget what you discovered.
- **Use the GUI as your notebook.** Structs, bookmarks, labels, and function names all persist in the GUI session. Populate them continuously.
- **Generate signatures for everything important.** Binary updates break addresses but not signatures. A good signature survives months of patches.
- **Don't get lost in the weeds.** Large modules have tens of thousands of functions. You don't need to understand them all. Focus on the subsystem relevant to the user's goal.
- **Use batch commands aggressively.** One batch with 30 label commands is better than 30 individual curl calls. Limit to ~50 actions per batch.
- **After this skill, use targeted skills for deep dives.** Once the module map is built, use `/probe:analyze-function` for specific functions, `/probe:discover-class` for specific classes, `/probe:map-struct` for specific data structures, and `/probe:trace-writes` for dynamic analysis.
