---
name: map-struct
description: Map out a data structure from a memory address by analyzing hex dumps, RTTI, and field types
---

# /probe:struct

Map out a data structure from a memory address.

## Arguments

- `address` (required): Base address of the struct instance (hex)
- `size` (optional): Approximate size in bytes (default: 256)
- `class_name` (optional): Known or suspected class name (helps with RTTI lookup)

## Steps

1. **Initial dump** -- Call `probe_dump` with the address and size to get a hex overview. Look for:
   - First 8 bytes: likely a vtable pointer (if it's a C++ object)
   - Pointer-like values: 8-byte aligned values in high address ranges (0x7FF..., 0x000001...)
   - Zero-filled regions: padding between fields or uninitialized data
   - Small integers: enum values, counts, flags
   - Float-like patterns: bytes that decode to reasonable float values

2. **RTTI identification** (if first 8 bytes look like a pointer):
   a. `probe_read_pointer {address}` -- read the vtable pointer
   b. `probe_read_pointer {vtable - 8}` -- read the RTTI COL pointer
   c. `probe_read_int {col + 0xC}` -- TypeDescriptor RVA
   d. Get module base from `probe_modules`
   e. `probe_read_string {module_base + rva + 0x10}` -- class name
   f. This confirms the object type and helps predict the layout

3. **Field-by-field analysis** -- Walk through the dump systematically:

   For each 8-byte aligned offset:
   a. **Check for pointer**: `probe_read_pointer {address + offset}`
      - If the value looks like an address, try to dereference it:
        - `probe_dump {ptr_value, 32}` -- see what it points to
        - `probe_read_string {ptr_value}` -- might be a string
        - Check if it's another object (vtable at ptr_value?)
   b. **Check for 32-bit int**: `probe_read_int {address + offset}`
      - Reasonable range for health, count, enum, flags?
   c. **Check for float**: `probe_read_float {address + offset}`
      - Reasonable range for coordinates, speed, time?
   d. **Check for boolean/byte**: `probe_read {address + offset, 1}`
      - 0 or 1 suggests a boolean flag

4. **Dynamic analysis** -- To understand what fields mean:
   a. Set hardware watchpoints on interesting fields: `probe_hwbp_set {addr + offset, "w", 4}`
   b. Perform actions that should change the field
   c. Check which fields changed and what code wrote to them
   d. The writing function often reveals the field's purpose

5. **Validate with multiple instances** -- If possible:
   a. Find another instance of the same struct
   b. Dump it and compare field positions
   c. Consistent patterns confirm field boundaries
   d. Different values at same offsets confirm they are separate data fields (not padding)

6. **Build the struct definition**:
   ```
   struct ClassName {          // Size: 0x100
       void* vtable;           // 0x000 - vtable pointer
       uint8_t _pad008[0x28];  // 0x008 - unknown
       void* parent;           // 0x030 - ptr to parent object
       int32_t health;         // 0x038 - current health
       int32_t max_health;     // 0x03C - maximum health
       float position_x;       // 0x040 - world X coordinate
       float position_y;       // 0x044 - world Y coordinate
       float position_z;       // 0x048 - world Z coordinate
       uint8_t _pad04C[0x4];   // 0x04C - padding
       uint64_t flags;         // 0x050 - state flags
       // ...
   };
   ```

7. **Report** the struct layout with:
   - Class name (from RTTI if available)
   - Total size
   - Each identified field: offset, type, name/purpose, sample value
   - Unknown regions marked as padding
   - Confidence level for each field identification

## Tips

- Vtable pointer is almost always at offset 0x0 for C++ objects
- Fields are typically aligned to their natural size (4-byte int at 4-byte boundary)
- Inheritance adds parent class fields at the beginning of the struct
- Padding often appears between fields of different sizes
- Arrays of same-typed values are easy to spot in hex dumps
- Strings in Source 2 are usually stored as pointers to CUtlString or const char*
- Bit fields and packed booleans are common in game engines

## Compound Type Detection Patterns

When analyzing hex dumps, look for these compound type signatures:

| Pattern | Type | Size | How to recognize |
|---------|------|------|-----------------|
| 3 consecutive floats at 4-aligned offset | `vec3` | 12 | Position/velocity — values in the 100s-1000s range (world coords) |
| 2 consecutive floats at 4-aligned offset | `vec2` | 8 | 2D coords — check the next 4 bytes aren't also a float (would be vec3) |
| 3 floats in degree range (0-360) | `qangle` | 12 | Angles — pitch/yaw/roll, typically -180 to 360 |
| 4 bytes where 4th byte (alpha) > 0x80 | `color32` | 4 | RGBA — at least one RGB channel > 10, alpha usually 0xFF |
| 4 consecutive 0.0-1.0 range floats | `colorf` | 16 | RGBA float color — common in shaders/materials |
| 12 consecutive floats (48 bytes) | `matrix3x4` | 48 | Transform matrix — rotation + translation |
| uint32 where `(val & 0x7FFF) < 16384` and upper bits non-zero | `handle` | 4 | Entity handle — CHandle in Source 2 |
| Float that matches game uptime | `gametime` | 4 | GameTime_t — monotonically increasing |
| Pointer to ASCII text | `string_ptr` | 8 | const char* — dereference and read string to verify |
| uint32 with specific bit patterns | `flags` | 4 | Bitmask — displayed as hex, individual bits are meaningful |

The GUI's **auto-detect** (Inspect button) uses these heuristics automatically. The inspector panel (right side) shows all possible interpretations at a selected offset, including compound types with color swatches for color32/colorf.
