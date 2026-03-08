---
name: probe-bridge
description: Interact with ARC Probe GUI via the Claude Bridge (HTTP :9996). Send probe commands, create structs, navigate tabs, set labels, and control the GUI programmatically. Use when the user wants to drive ARC Probe from Claude Code.
argument-hint: [action or probe command]
---

# ARC Probe — Claude Bridge Interaction

You have access to the ARC Probe GUI via an HTTP bridge on `127.0.0.1:9996`. This lets you send probe commands, mutate GUI state, and navigate the interface — all through `curl`.

## How to Send Commands

All requests are `POST` with `Content-Type: application/json`:

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{"action":"probe","command":"ping"}'
```

**IMPORTANT:** Always include `-H "Content-Type: application/json"` or the bridge will reject the request.

## Actions

### probe — Send TCP command to probe-shell.dll

Passthrough to the injected DLL's TCP server. Returns the probe's JSON response.

```bash
# Health check
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"ping"}'

# Read memory
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"read 0x7FFB0A8D0000 256"}'

# Pattern scan
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"pattern 48 8B 05 ?? ?? ?? ?? client.dll --resolve 3"}'

# Read pointer
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"read_ptr 0x7FFB0A8D0000+0x2E76FE8"}'
```

Response: `{"ok":true,"data":{"ok":true,"data":{...}}}`

The outer `ok` is the bridge status. The inner `ok`/`data` is from the probe itself.

### navigate — Switch GUI tabs

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"navigate","tab":"structs"}'

# Navigate to hex view at a specific address
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"navigate","tab":"hex","address":"0x7FFB0A8D0000"}'

# Navigate to disasm at address
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"navigate","tab":"disasm","address":"0x7FFB0A8D1000"}'
```

Tabs: `console`, `hex`, `structs`, `disasm`, `scanner`, `functions`

**Sub-tab navigation** (Functions tab):
```bash
# Navigate to a specific sub-tab within Functions
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"navigate","tab":"functions","subtab":"graph"}'
```

Functions sub-tabs: `functions`, `strings`, `classes`, `bookmarks`, `graph`

### store — Mutate Zustand stores directly

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"store","store":"struct","method":"createStruct","args":["MyStruct","0x1000",512]}'
```

See "Store Methods" section below for all available stores and methods.

### activity — Update Claude indicator in GUI

```bash
# Show working status
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"activity","status":"working","message":"Scanning entity list..."}'

# Set idle
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"activity","status":"idle","message":"Ready"}'
```

Statuses: `disconnected`, `idle`, `working`

### batch — Multiple actions in one request

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"activity","status":"working","message":"Building struct..."},
    {"action":"store","store":"struct","method":"createStruct","args":["Entity","0x1000",256]},
    {"action":"store","store":"struct","method":"addField","args":["Entity",0,"pointer","vtable"]},
    {"action":"store","store":"struct","method":"addField","args":["Entity",852,"int32","m_iHealth"]},
    {"action":"navigate","tab":"structs"},
    {"action":"activity","status":"idle","message":"Done"}
  ]
}'
```

Batch actions execute sequentially. Each returns its own result in the response array.

## Store Methods

### struct

| Method | Args | Description |
|--------|------|-------------|
| `createStruct` | `[name, baseAddress, totalSize?]` | Create a new struct (totalSize default 256) |
| `deleteStruct` | `[name]` | Delete a struct |
| `setActiveStruct` | `[name]` | Switch to viewing this struct |
| `renameStruct` | `[oldName, newName]` | Rename a struct |
| `setBaseAddress` | `[name, address]` | Change the base address |
| `setTotalSize` | `[name, size]` | Change total struct size |
| `addField` | `[structName, offset, type, name?]` | Add a field at offset |
| `removeField` | `[structName, fieldId]` | Remove field by ID |
| `updateField` | `[structName, fieldId, updates]` | Update field properties |
| `changeFieldType` | `[structName, fieldId, newType]` | Change a field's type |

Field types: `int8`, `int16`, `int32`, `int64`, `uint8`, `uint16`, `uint32`, `uint64`, `float`, `double`, `bool`, `pointer`, `char`, `wchar`, `vec2`, `vec3`, `qangle`, `color32`, `colorf`, `matrix3x4`, `flags`, `enum32`, `handle`, `gametime`, `string_ptr`

**Note:** `addField` offset is in **decimal** (not hex). Convert hex offsets: 0x330 = 816, 0x354 = 852, etc.

### hex

| Method | Args | Description |
|--------|------|-------------|
| `setAddress` | `[address]` | Navigate hex view to address |
| `setRegionSize` | `[size]` | Set region size |
| `pushAddress` | `[address]` | Push address to history |
| `addBookmark` | `[address, label?]` | Add bookmark |
| `removeBookmark` | `[address]` | Remove bookmark |

### disasm

| Method | Args | Description |
|--------|------|-------------|
| `navigate` | `[address]` | Navigate disasm to address |
| `setAddress` | `[address]` | Set address directly |

### label

| Method | Args | Description |
|--------|------|-------------|
| `setLabel` | `[address, label]` | Set address label |
| `removeLabel` | `[address]` | Remove label |
| `importLabels` | `[labelsObject]` | Import multiple labels |
| `clearLabels` | `[]` | Clear all labels |

### probe (GUI store, not the TCP probe)

| Method | Args | Description |
|--------|------|-------------|
| `setActiveTab` | `[tabName]` | Switch active tab |

### functions

| Method | Args | Description |
|--------|------|-------------|
| `discoverFunctions` | `[module]` | Discover functions in a module (exports, RTTI, pdata) |
| `setFunctionName` | `[address, name]` | Name a function |
| `setFunctionComment` | `[address, comment]` | Add comment to function |
| `clearFunctions` | `[]` | Clear all discovered functions |
| `importFunctions` | `[functionsObject]` | Import function data |

### strings

| Method | Args | Description |
|--------|------|-------------|
| `scanStrings` | `[module]` | Scan module for all printable strings |
| `findStrings` | `[query]` | Search for specific text |
| `selectString` | `[address]` | Select a string (loads its xrefs) |
| `loadXrefs` | `[address]` | Load code references to a string address |
| `clearStrings` | `[]` | Clear all scanned strings |

### rtti

| Method | Args | Description |
|--------|------|-------------|
| `scanModule` | `[module]` | Scan module for all RTTI classes |
| `findClass` | `[name, module?]` | Search for class by partial name |
| `selectClass` | `[name]` | Select a class (shows in detail pane) |
| `loadVtable` | `[className, module]` | Load vtable entries for class |
| `loadHierarchy` | `[className, module]` | Load inheritance chain |
| `clearClasses` | `[]` | Clear all RTTI data |

### bookmarks

| Method | Args | Description |
|--------|------|-------------|
| `addBookmark` | `[address, label?, color?, module?]` | Add a bookmark |
| `removeBookmark` | `[address]` | Remove a bookmark |
| `updateBookmark` | `[address, updates]` | Update bookmark properties |
| `clearBookmarks` | `[]` | Clear all bookmarks |

### xrefs

| Method | Args | Description |
|--------|------|-------------|
| `loadXrefs` | `[address]` | Load cross-references for a function |
| `setActiveAddress` | `[address]` | Set active address for xref display |
| `clearCache` | `[]` | Clear xref cache |

### analysis

The analysis store powers the **Explain** view — a visual, human-readable breakdown of what a function does.

| Method | Args | Description |
|--------|------|-------------|
| `analyzeFunction` | `[address]` | Disassemble and analyze a function (strings, calls, data access, steps) |
| `clear` | `[]` | Clear analysis cache |

The Explain view is available in two places:
1. **Functions tab** — click a function, the detail pane defaults to the "Explain" tab
2. **Disasm tab** — click the "Explain" button in the toolbar, or use the "Explain" right panel tab

The analysis extracts:
- **String references**: all strings the function loads (via RIP-relative LEA/MOV)
- **Function calls**: grouped by target with call counts
- **Step-by-step blocks**: logical grouping of instructions with plain-English labels
- **Data access**: struct field offsets read/written (e.g., `[rsi+0x1AF8]`)
- **Summary**: auto-generated one-line description from patterns

### callgraph

The callgraph store powers the **Graph** sub-tab — an interactive node canvas showing function call relationships.

| Method | Args | Description |
|--------|------|-------------|
| `buildGraph` | `[rootAddress, depth?]` | Build call graph from root (default depth 2) |
| `expandNode` | `[address]` | Expand one level deeper from a node |
| `collapseNode` | `[address]` | Collapse a node's children |
| `setSelectedNode` | `[address]` | Select a node in the graph |
| `setMaxDepth` | `[depth]` | Set max expansion depth (1-3) |
| `clear` | `[]` | Clear the graph |

```bash
# Build a call graph starting from a function, 2 levels deep
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"store","store":"callgraph","method":"buildGraph","args":["0x7FFB0C510000", 2]},
    {"action":"navigate","tab":"functions","subtab":"graph"}
  ]
}'
```

## New Probe TCP Commands

These are sent via `{"action":"probe","command":"..."}`:

| Command | Description |
|---------|-------------|
| `call <addr> [args] [--this <obj>] [--float N]` | Call function at address |
| `call info <addr>` | Analyze function prologue |
| `call history [--clear]` | Call result history |
| `session list` | List tracked memory writes |
| `session restore <id\|all>` | Restore original bytes |
| `session validate` | Check page validity of writes |
| `session info <id>` | Write record details |

## Probe TCP Commands Reference

These are sent via `{"action":"probe","command":"..."}`:

| Command | Description |
|---------|-------------|
| `ping` | Health check |
| `status` | PID, modules, process info |
| `modules list` | All loaded DLLs with base + size |
| `modules info <name>` | Details for specific module |
| `read <addr> <size>` | Read raw bytes (max 4096) |
| `read_ptr <addr>` | Read 8-byte pointer |
| `read_int <addr>` | Read int32 |
| `read_float <addr>` | Read float32 |
| `read_string <addr> [max]` | Read null-terminated string |
| `read_chain <addr> <off1> ...` | Follow pointer chain |
| `dump <addr> <size>` | Hex dump (max 256) |
| `write <addr> <hex>` | Write raw bytes |
| `write_int <addr> <value>` | Write int32 |
| `write_float <addr> <value>` | Write float32 |
| `write_ptr <addr> <value>` | Write 8-byte pointer |
| `pattern <bytes> [module] [--resolve N]` | Pattern scan |
| `sig <addr> [size] [module]` | Generate signature |
| `disasm <addr> [count]` | Disassemble instructions |
| `disasm func <addr>` | Disassemble full function |
| `rtti scan <module>` | Scan for RTTI type descriptors |
| `rtti hierarchy <class>` | Show inheritance tree |
| `strings search <module> <needle>` | Find string references |
| `pe headers <module>` | PE header info |
| `pe exports <module>` | Export table |
| `pe imports <module>` | Import table |
| `threads` | List all threads |
| `bp set <addr>` | Set INT3 breakpoint |
| `bp list` | List breakpoints |
| `watch <addr> <size> [interval]` | Watch memory changes |

Addresses support `+` for offsets: `0x7FFB0A8D0000+0x2E76FE8`

## Quick Start

Run these first five commands when starting any new investigation session:

```bash
# 1. Verify connection to probe
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"ping"}'

# 2. List loaded modules (find your target)
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"modules list"}'

# 3. Batch-populate the GUI: discover functions + scan strings + scan RTTI for the main module
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"activity","status":"working","message":"Populating GUI panels..."},
    {"action":"store","store":"functions","method":"discoverFunctions","args":["client.dll"]},
    {"action":"store","store":"strings","method":"scanStrings","args":["client.dll"]},
    {"action":"store","store":"rtti","method":"scanModule","args":["client.dll"]}
  ]
}'

# 4. Navigate to the functions tab to start exploring
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"navigate","tab":"functions"}'

# 5. Set activity to idle
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"activity","status":"idle","message":"Ready"}'
```

Replace `client.dll` with whatever module you are targeting. For smaller binaries, step 3 takes a few seconds. For large modules (60MB+), it can take 10-30 seconds -- the `working` activity indicator keeps the user informed.

After this, the Functions, Strings, and Classes panels are all populated and searchable in the GUI.

## Workflow

If the user provides an argument, treat it as a probe command and send it directly.

Otherwise, follow this decision tree:

1. **Check connectivity:** Send `ping` via probe action.
2. **Get module bases:** `modules list` -- identify the target module(s).
3. **Set activity status** to `working` when doing multi-step operations.
4. **Use batch** to combine multiple store mutations into one request (avoids many round-trips).
5. **Navigate** to the relevant tab after populating data.
6. **Set activity** back to `idle` when done.
7. **Present results** clearly -- summarize JSON into readable tables.

### Populating GUI Panels

Always populate the three discovery panels early. They feed into each other:

- **Functions** (`discoverFunctions`) -- exports, `.pdata` unwind entries, and RTTI vtable owners. Gives you a function list with addresses, names, and sizes.
- **Strings** (`scanStrings`) -- all printable strings in a module. Each string has an address you can xref.
- **Classes** (`scanModule`) -- RTTI type descriptors with class names. Each class links to a vtable you can expand.

Batch all three in one request (see Quick Start). You can target multiple modules by sending separate batch calls:

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"activity","status":"working","message":"Scanning engine2.dll..."},
    {"action":"store","store":"functions","method":"discoverFunctions","args":["engine2.dll"]},
    {"action":"store","store":"strings","method":"scanStrings","args":["engine2.dll"]},
    {"action":"store","store":"rtti","method":"scanModule","args":["engine2.dll"]},
    {"action":"activity","status":"idle","message":"Done"}
  ]
}'
```

### String Investigation Workflow

Goal: find a function by starting from a known string (error message, log text, class name literal).

```bash
# Step 1: Search for the string in the already-scanned strings
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"store","store":"strings","method":"findStrings","args":["error: invalid"]}'

# Step 2: Select the interesting string (loads its details in the GUI panel)
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"store","store":"strings","method":"selectString","args":["0x7FFB0C123456"]}'

# Step 3: Load xrefs -- find code that references this string
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"store","store":"strings","method":"loadXrefs","args":["0x7FFB0C123456"]}'

# Step 4: Disassemble at the xref address to see the referencing function
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"disasm func 0x7FFB0C200100"}'

# Step 5: Name the function and bookmark it
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"store","store":"functions","method":"setFunctionName","args":["0x7FFB0C200100","ValidateInput"]},
    {"action":"store","store":"bookmarks","method":"addBookmark","args":["0x7FFB0C200100","ValidateInput","blue","client.dll"]},
    {"action":"navigate","tab":"disasm","address":"0x7FFB0C200100"}
  ]
}'
```

You can also use the probe TCP command directly if you haven't scanned strings yet:

```bash
# Direct string search + xref via probe (bypasses GUI stores)
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"strings find error: invalid"}'

curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"strings xref 0x7FFB0C123456"}'
```

### Class Investigation Workflow

Goal: understand a C++ class -- its hierarchy, vtable layout, and virtual functions.

```bash
# Step 1: Search for the class in scanned RTTI data
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"store","store":"rtti","method":"findClass","args":["CPlayerController"]}'

# Step 2: Select the class (shows details in the GUI Classes panel)
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"store","store":"rtti","method":"selectClass","args":["CPlayerController"]}'

# Step 3: Load the inheritance hierarchy
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"store","store":"rtti","method":"loadHierarchy","args":["CPlayerController","client.dll"]}'

# Step 4: Load the vtable entries (each entry is a virtual function address)
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"store","store":"rtti","method":"loadVtable","args":["CPlayerController","client.dll"]}'

# Step 5: Disassemble interesting vtable entries and name them
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"probe","command":"disasm func 0x7FFB0C300200"},
    {"action":"store","store":"functions","method":"setFunctionName","args":["0x7FFB0C300200","CPlayerController::GetHealth"]},
    {"action":"store","store":"functions","method":"setFunctionComment","args":["0x7FFB0C300200","vtable[3], reads m_iHealth at rcx+0x354"]}
  ]
}'
```

To explore a class from an object pointer (no prior RTTI scan needed):

```bash
# Resolve RTTI from a live object
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"rtti resolve 0x055EA1C28FE8"}'

# Get the full hierarchy
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"rtti hierarchy CPlayerController"}'

# Get vtable entries
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"rtti vtable CPlayerController"}'
```

### Bookmarking Findings

Use bookmarks to tag important addresses with colors for visual categorization. Colors help you (and the user) distinguish different types of findings at a glance.

**Color conventions:**

| Color | Category | Example |
|-------|----------|---------|
| `blue` | Functions you have identified and named | `ValidateInput`, `UpdateHealth` |
| `red` | Dangerous / security-sensitive addresses | Anti-cheat checks, integrity validators |
| `green` | Data structures and globals | Entity list pointer, view matrix |
| `yellow` | Points of interest needing further investigation | Unresolved xrefs, suspicious branches |
| `purple` | Vtable addresses and class metadata | `CPlayerController vtable` |

```bash
# Bookmark a function
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"store","store":"bookmarks","method":"addBookmark","args":["0x7FFB0C200100","ValidateInput","blue","client.dll"]}'

# Bookmark a global variable
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"store","store":"bookmarks","method":"addBookmark","args":["0x7FFB0A8D0000+0x2E76FE8","g_EntityList","green","client.dll"]}'

# Update an existing bookmark's label or color
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"store","store":"bookmarks","method":"updateBookmark","args":["0x7FFB0C200100",{"label":"ValidateInput (confirmed)","color":"blue"}]}'

# Batch-bookmark multiple findings from one investigation
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"store","store":"bookmarks","method":"addBookmark","args":["0x7FFB0C300000","CPlayerController vtable","purple","client.dll"]},
    {"action":"store","store":"bookmarks","method":"addBookmark","args":["0x7FFB0C300200","CPlayerController::GetHealth","blue","client.dll"]},
    {"action":"store","store":"bookmarks","method":"addBookmark","args":["0x7FFB0C300240","CPlayerController::SetHealth","red","client.dll"]},
    {"action":"store","store":"bookmarks","method":"addBookmark","args":["0x7FFB0C400000","g_PlayerManager","green","client.dll"]}
  ]
}'
```

### Cross-Referencing

Load xrefs for any function to see who calls it (callers) and what it calls (callees).

```bash
# Load xrefs for a function address
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"store","store":"xrefs","method":"loadXrefs","args":["0x7FFB0C200100"]}'

# Set the active address so the GUI highlights the xref data
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"store","store":"xrefs","method":"setActiveAddress","args":["0x7FFB0C200100"]}'
```

The xref data appears in the Functions tab detail panel. Each caller/callee is clickable in the GUI, which navigates the disassembly view.

**Full xref exploration flow:**

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"activity","status":"working","message":"Loading xrefs for UpdateHealth..."},
    {"action":"store","store":"xrefs","method":"loadXrefs","args":["0x7FFB0C200100"]},
    {"action":"store","store":"xrefs","method":"setActiveAddress","args":["0x7FFB0C200100"]},
    {"action":"navigate","tab":"functions"},
    {"action":"activity","status":"idle","message":"Xrefs loaded"}
  ]
}'
```

Then examine each caller:

```bash
# Disassemble a caller to understand why it calls UpdateHealth
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"disasm func 0x7FFB0C210000"}'

# Name the caller if you identify it
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"store","store":"functions","method":"setFunctionName","args":["0x7FFB0C210000","ProcessDamageEvent"]}'
```

### Function Calling

Use the `call` command to invoke discovered functions directly. This is powerful for testing hypotheses about what a function does.

```bash
# Analyze a function's calling convention before calling it
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"call info 0x7FFB0C200100"}'

# Call a function with integer arguments (args go into RCX, RDX, R8, R9)
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"call 0x7FFB0C200100 42 100"}'

# Call a member function with a this pointer
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"call 0x7FFB0C200100 42 --this 0x055EA1C28FE8"}'

# Call a function that takes float arguments (specify which arg positions are floats)
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"call 0x7FFB0C200100 42 3.14 --float 2"}'

# Review call history
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"call history"}'
```

**Safety:** Calling arbitrary functions can crash the process. Always:
1. Disassemble the function first to understand what it does.
2. Use `call info` to check the prologue and expected arguments.
3. Start with simple getter functions (no side effects) before trying setters.
4. Check `call history` to review return values and detect failures.

### Write Tracking with Session Commands

The `session` commands track all memory writes so you can safely experiment and restore original values.

```bash
# List all tracked writes in this session
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"session list"}'

# Before experimenting, read the original value
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"read_int 0x055EA1C28FE8+0x354"}'

# Write a test value (session automatically tracks it)
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"write_int 0x055EA1C28FE8+0x354 999"}'

# Check session to see the tracked write with original bytes
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"session list"}'

# Restore a single write by its ID
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"session restore 1"}'

# Or restore ALL writes at once (undo everything)
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"session restore all"}'

# Validate that tracked addresses are still valid memory
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"session validate"}'

# Get details on a specific write record
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"session info 1"}'
```

**Best practice:** Always use `session list` after a series of writes so you can report to the user what was changed and offer to restore.

### Putting It All Together: Full Investigation Example

Here is a complete workflow combining multiple panels. Goal: find and understand a "damage calculation" function.

```bash
# 1. Ensure panels are populated (Quick Start step 3)

# 2. Search for damage-related strings
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"store","store":"strings","method":"findStrings","args":["damage"]}'

# 3. Pick a promising string, get its xrefs
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"store","store":"strings","method":"selectString","args":["0x7FFB0C500000"]},
    {"action":"store","store":"strings","method":"loadXrefs","args":["0x7FFB0C500000"]}
  ]
}'

# 4. Disassemble the referencing function
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"disasm func 0x7FFB0C510000"}'

# 5. The function accesses an object via RCX. Resolve its RTTI.
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"rtti resolve 0x055EA1C28FE8"}'
# Returns: CDamageCalculator

# 6. Explore the class hierarchy and vtable
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"store","store":"rtti","method":"selectClass","args":["CDamageCalculator"]},
    {"action":"store","store":"rtti","method":"loadHierarchy","args":["CDamageCalculator","client.dll"]},
    {"action":"store","store":"rtti","method":"loadVtable","args":["CDamageCalculator","client.dll"]}
  ]
}'

# 7. Load xrefs for the function to see its callers
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"store","store":"xrefs","method":"loadXrefs","args":["0x7FFB0C510000"]},
    {"action":"store","store":"xrefs","method":"setActiveAddress","args":["0x7FFB0C510000"]}
  ]
}'

# 8. Name the function, comment it, bookmark it
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"store","store":"functions","method":"setFunctionName","args":["0x7FFB0C510000","CDamageCalculator::CalculateDamage"]},
    {"action":"store","store":"functions","method":"setFunctionComment","args":["0x7FFB0C510000","Refs string \"damage\", called from ProcessHitEvent. RCX=this, EDX=raw_damage, R8=armor_type"]},
    {"action":"store","store":"bookmarks","method":"addBookmark","args":["0x7FFB0C510000","CalculateDamage","blue","client.dll"]},
    {"action":"navigate","tab":"functions"},
    {"action":"activity","status":"idle","message":"Investigation complete"}
  ]
}'
```

## Tips

- The bridge HTTP server runs inside the Tauri GUI backend. If ARC Probe GUI isn't running, the bridge is unavailable.
- The probe TCP server runs inside the injected DLL. If probe-shell.dll isn't injected, probe commands will fail (but navigate/store/activity still work).
- For large reads (>256 bytes), the probe response has `hex` but not `bytes` array. The GUI handles this.
- Pattern scans on large modules (60MB+ client.dll) take several seconds. Set activity to `working` first.
- Address arithmetic works in probe commands: `read_ptr 0x7FFB0A8D0000+0x330`
- Struct field offsets in `addField` are **decimal integers**, not hex strings.
- **Batch aggressively.** Every HTTP round-trip costs latency. Combine store mutations, bookmarks, labels, and navigation into a single batch whenever possible.
- **Name functions as you go.** Use `setFunctionName` and `setFunctionComment` immediately after identifying a function. These persist in the GUI session and make later exploration much easier.
- **Bookmark with colors consistently.** Pick a color scheme (see Bookmarking section) and stick with it so the user can visually parse the bookmark list.
- **Xrefs are bidirectional.** `loadXrefs` returns both callers (who calls this function) and callees (what this function calls). Use callees to understand dependencies, callers to understand usage.
- **Session commands are your safety net.** Before any write experiment, note the session state. After writes, always report what was changed and offer `session restore all` as an undo.
- **The GUI panels are interconnected.** Clicking a function in the Functions tab can navigate to disassembly. Clicking an xref jumps to the caller. Clicking a class in Classes shows its vtable. Use `navigate` to guide the user's view to the right panel at the right time.
- **For Source 2 games,** scan `client.dll` first (largest, most game logic), then `engine2.dll` and `schemasystem.dll` as needed.
