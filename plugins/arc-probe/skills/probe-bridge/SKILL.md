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

Tabs: `console`, `hex`, `structs`, `disasm`, `scanner`

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

Field types: `int8`, `int16`, `int32`, `int64`, `uint8`, `uint16`, `uint32`, `uint64`, `float`, `double`, `bool`, `pointer`, `char`, `wchar`

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

## Workflow

If the user provides an argument, treat it as a probe command and send it.

Otherwise:

1. **Check connectivity:** Send `ping` via probe action
2. **Get module bases:** `modules list` — find client.dll, engine2.dll etc.
3. **Set activity status** to `working` when doing multi-step operations
4. **Use batch** for creating structs with multiple fields (one request instead of many)
5. **Navigate** to the relevant tab after populating data
6. **Set activity** back to `idle` when done
7. **Present results** clearly — summarize JSON into readable tables

## Tips

- The bridge HTTP server runs inside the Tauri GUI backend. If ARC Probe GUI isn't running, the bridge is unavailable.
- The probe TCP server runs inside the injected DLL. If probe-shell.dll isn't injected, probe commands will fail (but navigate/store/activity still work).
- For large reads (>256 bytes), the probe response has `hex` but not `bytes` array. The GUI handles this.
- Pattern scans on large modules (60MB+ client.dll) take several seconds. Set activity to `working` first.
- Address arithmetic works in probe commands: `read_ptr 0x7FFB0A8D0000+0x330`
- Struct field offsets in `addField` are **decimal integers**, not hex strings.
