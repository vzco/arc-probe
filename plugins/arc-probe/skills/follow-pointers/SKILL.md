---
name: follow-pointers
description: Follow a pointer chain to navigate nested data structures step by step
---

# /probe:follow

Follow a pointer chain to navigate nested data structures.

## Arguments

- `address` (required): Starting address (hex)
- `chain` (required): Comma-separated offsets describing the chain (e.g., "0x30,0x10,0x354")

## How pointer chains work

Games and complex software store data in nested structures:

```
GlobalPtr -> EntityList -> Entity[i] -> GameSceneNode -> Position (Vec3)
```

Each arrow is a pointer dereference. To read the position, you:
1. Read the global pointer to get the EntityList address
2. Add an offset and read the pointer to get Entity[i]
3. Add an offset and read the pointer to get GameSceneNode
4. Add an offset and read the float values (no dereference — final value)

## Steps

1. **Use `probe_read_chain`** for a single call:
   ```
   probe_read_chain address=<start> offsets=[0x30, 0x10, 0x354]
   ```
   This follows each offset: dereference at start+0x30, dereference at result+0x10, read value at result+0x354.

2. **If the chain fails**, break it down step by step to find where it breaks:
   ```
   probe_read_pointer address=<start>              ; read vtable/first ptr at base
   probe_dump address=<start> size=128             ; overview of the base struct
   probe_read_pointer address=<start+0x30>         ; follow first offset
   probe_dump address=<result> size=128            ; overview of second struct
   probe_read_pointer address=<result+0x10>        ; follow second offset
   probe_read_int address=<result+0x354>           ; read final value
   ```

3. **Validate each link** — at each step, check:
   - Is the pointer value in a valid range? (typically 0x7FF... for DLL memory, 0x00000001... for heap)
   - Does the pointed-to memory look like a struct? (vtable pointer at offset 0, reasonable values)
   - Is the pointer NULL (0x0)? That means the object doesn't exist right now

## When a pointer is NULL or invalid

- **NULL pointer (0x0)**: The object isn't allocated yet. Common for optional fields. Wait for the right game state (e.g., entity must be alive, player must be in-game).
- **0xDEADBEEF / 0xFEEEFEEE / 0xCDCDCDCD**: Debug fill patterns — the memory was freed or never initialized.
- **0xFFFFFFFF or 0x7FFF**: Often a "null handle" in entity systems — the handle is invalid.
- **Small values (< 0x10000)**: Probably not a pointer — this offset might be an integer field, not a pointer field.

## Common pointer chain patterns

**Entity via index:**
```
EntityList + 0x10 -> ChunkArray
ChunkArray + (chunk_index * 8) -> Chunk
Chunk + (entry_index * stride) -> EntityIdentity
EntityIdentity + 0x0 -> Entity pointer
```

**Object hierarchy:**
```
Entity + 0x330 -> GameSceneNode
GameSceneNode + 0xC8 -> AbsOrigin (Vec3, read 12 bytes as 3 floats)
GameSceneNode + 0x103 -> bDormant (1 byte, 0 = active)
```

**Controller -> Pawn resolution:**
```
Controller + 0x6BC -> m_hPawn (CHandle, 4 bytes)
handle & 0x7FFF = entity_index
Resolve via entity list to get pawn pointer
Pawn + 0x354 -> m_iHealth
```

## Tips

- If you know the class name, use `rtti find <classname> <module>` to find it, then `rtti vtable` to get the vtable layout
- `probe_dump` at the start of a struct shows the memory layout — pointers and integers are visually distinct in hex
- When exploring an unknown struct, read 256 bytes and look for pointer-like values (8 bytes, high addresses), then dereference each one to see what it points to
- Entity handles (CHandle) are NOT pointers — they're 4-byte indices that must be resolved through the entity list
