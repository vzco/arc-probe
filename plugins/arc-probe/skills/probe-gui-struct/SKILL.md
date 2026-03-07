---
name: probe-gui-struct
description: Create, populate, and inspect structs in the ARC Probe GUI from live process memory. Reads memory via the probe, builds struct definitions in the GUI, and navigates to view them. Use when the user wants to map out a data structure visually.
argument-hint: <address> [class_name] [size]
---

# ARC Probe — GUI Struct Builder

Build struct definitions in the ARC Probe GUI backed by live memory reads from an injected process.

## Prerequisites

- ARC Probe GUI running (`pnpm tauri dev` in `arc-probe/gui/`)
- probe-shell.dll injected into target process
- Bridge on `127.0.0.1:9996`, probe on TCP (usually 9998)

## Arguments

- `address` (required): Base address of the struct instance (hex, e.g., `0x2E6C0389400`)
- `class_name` (optional): Name for the struct (default: attempts RTTI identification)
- `size` (optional): Struct size in bytes (default: 256, use 1024 for large game objects)

## Steps

### 1. Verify connectivity

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"ping"}'
```

### 2. Identify the object (RTTI)

If the address is a C++ object, the first 8 bytes are usually a vtable pointer:

```bash
# Read vtable pointer
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"read_ptr <address>"}'

# Read RTTI Complete Object Locator (vtable - 8)
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"read_ptr <vtable>-8"}'

# Read TypeDescriptor RVA (COL + 0xC)
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"read_int <col>+0xC"}'

# Read mangled class name (module_base + rva + 0x10)
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"read_string <module_base>+<rva>+0x10"}'
```

Use the class name (strip `.?AV` prefix and `@@` suffix) as the struct name.

### 3. Read memory to identify fields

```bash
# Hex dump first 256 bytes
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"dump <address> 256"}'
```

Look for:
- **Pointers** (8 bytes, high address ranges like `0x7FF...` or `0x0000_02...`)
- **Integers** (4 bytes, reasonable values for health/count/enum)
- **Floats** (4 bytes, reasonable for coordinates/time)
- **Booleans** (1 byte, 0 or 1)
- **Zero runs** (padding between fields)

### 4. Create the struct in the GUI

Use a batch to create the struct and add all fields at once. Field offsets must be **decimal**.

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"activity","status":"working","message":"Creating <ClassName>..."},
    {"action":"store","store":"struct","method":"createStruct","args":["<ClassName>","<address>",<size>]},
    {"action":"store","store":"struct","method":"addField","args":["<ClassName>",0,"pointer","vtable"]},
    {"action":"store","store":"struct","method":"addField","args":["<ClassName>",816,"pointer","m_pGameSceneNode"]},
    {"action":"store","store":"struct","method":"addField","args":["<ClassName>",852,"int32","m_iHealth"]},
    {"action":"store","store":"struct","method":"setActiveStruct","args":["<ClassName>"]},
    {"action":"navigate","tab":"structs"},
    {"action":"activity","status":"idle","message":"<ClassName> ready"}
  ]
}'
```

### 5. Add labels for key addresses

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"store","store":"label","method":"setLabel","args":["<address>","<ClassName> instance"]}'
```

### 6. Verify values

Read specific fields to confirm the struct is correct:

```bash
# Read health
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"read_int <address>+0x354"}'

# Read team
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"read <address>+0x3F3 1"}'
```

Compare with what the GUI shows. Enable "Live" mode in the toolbar for auto-refresh.

## Common Field Types and Offsets

### Hex to Decimal Conversion (for addField)

| Hex | Decimal | Common Field |
|-----|---------|-------------|
| 0x000 | 0 | vtable |
| 0x330 | 816 | m_pGameSceneNode |
| 0x350 | 848 | m_iMaxHealth |
| 0x354 | 852 | m_iHealth |
| 0x35C | 860 | m_lifeState |
| 0x3C0 | 960 | m_flSimulationTime |
| 0x3F3 | 1011 | m_iTeamNum |

### Field Type Reference

| Type | Size | Use for |
|------|------|---------|
| `pointer` | 8 | Pointers, vtables, scene nodes |
| `int32` | 4 | Health, team, counts, enums |
| `uint32` | 4 | Handles, flags, unsigned values |
| `float` | 4 | Coordinates, time, speed |
| `bool` | 1 | Flags (0/1) |
| `uint8` | 1 | Small enums, team number |
| `int64` | 8 | Steam IDs, large counters |
| `uint64` | 8 | Bitmasks, handles |

## Tips

- Use `totalSize` of 1024 for game entity structs (C_BaseEntity is ~4000+ bytes but 1024 covers the most useful fields)
- The GUI's "Compact" button hides undefined rows, showing only your labeled fields
- Enable "Live" toggle (top-right) for auto-refreshing values every 200ms
- After creating a struct, you can click any undefined row in the GUI to add fields manually
- Double-click a field name in the GUI to rename it
- Use the inspector panel (right side) to see all type interpretations at a selected offset
- If values show `?`, the probe isn't connected or the address is invalid
