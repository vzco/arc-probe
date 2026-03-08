---
name: map-module
description: Comprehensive module analysis — discover all functions, scan strings, find key functions by string refs, build a labeled function map
argument-hint: <module>
---

# /probe:map-module

Comprehensive analysis of a loaded module. Discovers all functions (exports + RTTI + .pdata), scans for strings, traces string references to find key functions, and builds a complete function map with labels in the GUI.

## Arguments

- `module` (required): Module name (e.g., "client.dll", "engine2.dll", "target.exe")

## Steps

### 1. Verify the module is loaded

```
probe.exe "modules list"
```

Find the target module in the list. Record:
- **Base address**
- **Size**
- **Full path** (if available)

If the module is not found, report available modules and stop. Common mistakes:
- Case sensitivity — try both `Client.dll` and `client.dll`
- Missing extension — try `<name>.dll` or `<name>.exe`
- The module might not be loaded yet (e.g., game still on menu screen)

### 2. Get PE structure overview

```
probe.exe "pe sections <module>"
```

Record the sections — this tells you where code vs data lives:
- `.text` — executable code (functions live here)
- `.rdata` — read-only data (strings, vtables, RTTI metadata)
- `.data` — writable globals
- `.pdata` — exception/unwind data (function boundaries)

Note the `.text` section size — this is the total code area to analyze.

### 3. Discover all functions

```
probe.exe "functions discover <module> --limit 5000"
```

This combines three discovery sources:
- **Exports** — named functions in the PE export table
- **RTTI** — virtual functions found via RTTI vtable scanning
- **.pdata** — all functions with unwind info (most comprehensive)

The response includes:
- `count` — number of functions returned
- `total` — total functions discovered (may exceed limit)
- Each function: `address`, `rva`, `source` (export/rtti/pdata), `name` (if known), `size` (from pdata)

Record the total count and the named functions (exports and RTTI). Unnamed .pdata entries are unlabeled — they'll be identified in later steps.

If `total` exceeds the limit, note this. For large modules (e.g., client.dll with 100K+ functions), focus analysis on named functions and string-referenced functions rather than trying to catalog everything.

### 4. Analyze the export table

```
probe.exe "pe exports <module>"
```

Exports are the module's public API. For each exported function:
- Record the name and address
- Group by naming pattern (e.g., `CreateInterface`, `Init*`, `Shutdown*`, `Get*`)

Key exports to look for:
- `CreateInterface` — Source 2 interface factory (critical entry point)
- `DllMain` / `DllEntryPoint` — initialization entry
- Functions with `Init`, `Create`, `Register` in the name — initialization functions
- Functions with `Get`, `Find`, `Lookup` — accessor functions

### 5. Scan for RTTI classes

```
probe.exe "rtti scan <module>"
```

This reveals all C++ classes with virtual functions in the module. For each class:
- Record the class name and vtable address
- Note the inheritance if visible

Group classes by namespace or prefix to understand the module's class hierarchy. Common patterns:
- `C_*` or `CBase*` — game entity classes
- `I*` — interface classes
- `*Manager` / `*System` — singleton managers
- `*Component` — component-based architecture

For the most important classes (3-5 key classes), get the full hierarchy:

```
probe.exe "rtti hierarchy <class_name> <module>"
```

### 6. Scan for strings

```
probe.exe "strings scan <module>"
```

This scans the module's .rdata section for printable strings. The results can be large. Focus on:
- **Error messages** — `"Failed..."`, `"Error..."`, `"Invalid..."` — reveal error handling functions
- **Log/debug strings** — `"[%s]..."`, `"DEBUG:..."` — reveal logging infrastructure
- **Format strings** — `"%s"`, `"%d"`, `"%f"` — reveal data processing functions
- **Class/function names** — `"CClassName::FuncName"` — direct function identification
- **UI text** — buttons, labels, tooltips — reveal UI handler functions
- **File paths** — `".cfg"`, `".json"`, `".mdl"` — reveal file loading functions

### 7. Trace key strings to functions

For the 5-10 most interesting strings found in step 6, find the functions that reference them:

```
probe.exe "strings find <text> <module>"
```

Note the string address, then find code references:

```
probe.exe "strings xref <string_address> <module>"
```

Each xref gives you a code address (LEA instruction) and possibly the enclosing function. For each:

```
probe.exe "disasm <xref_address - 0x20> 15"
```

Identify the enclosing function and what it does. This is the highest-value analysis step — string references directly reveal function purpose.

### 8. Analyze the import table

```
probe.exe "pe imports <module>"
```

Imports reveal what OS and library APIs the module uses. Group by DLL:
- **kernel32.dll** — file I/O, memory, threading, process management
- **user32.dll** — window management, input handling
- **ws2_32.dll** / **winhttp.dll** — networking
- **d3d11.dll** / **dxgi.dll** — graphics rendering
- **ntdll.dll** — low-level OS functions

This tells you the module's capabilities (does it do networking? graphics? file I/O?).

### 9. Build the function map and label in GUI

Combine all findings into a function map. For each identified function, create a GUI label:

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"activity","status":"working","message":"Labeling <module> functions..."},
    {"action":"store","store":"label","method":"setLabel","args":["0x<addr1>","ExportedFunc1"]},
    {"action":"store","store":"label","method":"setLabel","args":["0x<addr2>","ExportedFunc2"]},
    {"action":"store","store":"label","method":"setLabel","args":["0x<addr3>","ClassName::vf0 (destructor)"]},
    {"action":"store","store":"label","method":"setLabel","args":["0x<addr4>","handles: \"error message text\""]},
    {"action":"activity","status":"idle","message":"Labeled N functions in <module>"}
  ]
}'
```

Use batch to label many functions at once. Limit batch size to ~50 labels per request.

### 10. Report

Present the full module map:

```
Module Analysis: client.dll
===========================

Base:    0x7FFB0A8D0000
Size:    0x3E00000 (62 MB)
Path:    C:\Program Files\Steam\steamapps\common\Game\bin\win64\client.dll

Sections:
  .text    0x001000  0x2800000  (40 MB code)
  .rdata   0x2801000 0x0F00000  (15 MB read-only data)
  .data    0x3701000 0x0100000  (1 MB writable data)
  .pdata   0x3801000 0x0300000  (3 MB exception data)

Functions: 98,432 total (via .pdata)
  Exports: 47 named
  RTTI:    2,841 virtual functions across 412 classes

Key Exports:
  0x7FFB0A8D1000  CreateInterface
  0x7FFB0A8D2000  DllMain
  ...

Key Classes (RTTI):
  C_BaseEntity (142 vtable entries)
    <- CEntityInstance <- CBaseEntity
  C_BasePlayerPawn (89 vtable entries)
    <- C_BaseEntity <- ...
  CGameEntitySystem (31 vtable entries)
  ...

Key Functions (from string references):
  0x7FFB0B1A4500  "CBaseEntity::TakeDamage" handler
  0x7FFB0B2E7800  "Failed to initialize entity %s" error path
  0x7FFB0B3C1200  "Connecting to server %s:%d" network connect
  ...

Imports: 12 DLLs
  kernel32.dll (84 functions) — file I/O, memory
  user32.dll (23 functions) — window management
  ws2_32.dll (18 functions) — networking
  ...
```

## For very large modules (>50 MB)

Large modules like Source 2's `client.dll` contain tens of thousands of functions. Adjust the workflow:

1. **Skip full function discovery** — use `--exports --rtti` without `--pdata` to avoid listing 100K+ unnamed functions
2. **Focus string search** — search for specific strings rather than scanning all strings
3. **Target specific classes** — use `rtti find <partial_name>` to search for classes of interest
4. **Limit xref scans** — xref scanning a 40MB .text section takes several seconds per scan

## For small modules (<1 MB)

Small modules are fully analyzable:

1. **Discover everything** — `functions discover <module>` with no limit
2. **Scan all strings** — `strings scan <module>` produces a manageable list
3. **Trace every string** — xref every interesting string
4. **Disassemble key functions** — disassemble the top 10-20 functions fully
5. **Map all RTTI classes** — get hierarchy and vtable for every class

## Tips

- Start with exports and RTTI — these give you named anchor points
- String references are the bridge between "unnamed .pdata function" and "understood function"
- The import table reveals module capabilities at a glance
- For game modules, focus on classes with `Entity`, `Player`, `World`, `Game` in the name
- Always label findings in the GUI so they persist across analysis sessions
- Run this skill first, then use `/probe:analyze-function` to deep-dive into specific functions of interest
