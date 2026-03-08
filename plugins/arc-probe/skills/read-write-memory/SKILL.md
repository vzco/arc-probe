---
name: read-write-memory
description: Read and write process memory — all data types, safety rules, and concrete examples via CLI, TCP, or bridge
---

# /probe:rw

Read and write process memory. Covers every data type, all three command delivery methods, safety rules, and copy-paste examples.

## How to Send Commands

ARC Probe commands can be sent three ways. Use whichever is available:

### 1. CLI (simplest)

```bash
probe.exe <command>
```

Returns JSON to stdout. Exit code 0 = success, 1 = error.

```bash
probe.exe "read_int 0x7FFB0A8D0000+0x354"
# {"ok":true,"data":{"value":100}}

probe.exe "write_int 0x055EA1C28FE8+0x50 1"
# {"ok":true,"data":{"bytes_written":4}}
```

If `probe.exe` is not in PATH, use the full path to the build directory.

### 2. TCP (direct)

Connect to `127.0.0.1:9998` and send commands as newline-terminated strings:

```powershell
$tcp = New-Object System.Net.Sockets.TcpClient('127.0.0.1', 9998)
$s = $tcp.GetStream()
$w = New-Object System.IO.StreamWriter($s); $w.AutoFlush = $true
$r = New-Object System.IO.StreamReader($s)
$w.WriteLine('read_int 0x7FFB0A8D0000+0x354')
$r.ReadLine()
$tcp.Close()
```

### 3. HTTP Bridge (when GUI is running)

```bash
curl -s -X POST http://localhost:9996 \
  -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"read_int 0x7FFB0A8D0000+0x354"}'
```

The bridge wraps the response: `{"ok":true,"data":{"ok":true,"data":{...}}}`. Outer `ok` = bridge status, inner = probe response.

## Address Formats

All addresses are hexadecimal. Accepted formats:

| Format | Example |
|--------|---------|
| With 0x prefix | `0x7FF612340000` |
| Without prefix | `7FF612340000` |
| Module-relative | `client.dll+0x2E76FE8` |
| Offset arithmetic | `0x055EA1C28FE8+0x50` |

To get module bases: `modules list`

---

## Reading Memory

### read — Raw bytes (hex string)

```
read <addr> <size>
```

Returns up to 4096 bytes as a hex string. Use for opcode bytes, raw binary data, or when you need exact byte values.

```bash
probe.exe "read 0x7FFB0A8D0000 16"
# {"ok":true,"data":{"hex":"4D5A9000030000000400...","size":16}}
```

### read_ptr — 8-byte pointer

```
read_ptr <addr>
```

Reads an 8-byte (64-bit) pointer value. Use for following pointer fields in structs, reading vtable pointers, or any 8-byte address.

```bash
probe.exe "read_ptr 0x055EA1C28FE8"
# {"ok":true,"data":{"value":"0x7FFB22EA1000"}}
```

### read_int — 32-bit integer

```
read_int <addr>
```

Reads a signed 32-bit integer. Use for health, team numbers, enum values, counts, flags.

```bash
probe.exe "read_int 0x055EA1C28FE8+0x354"
# {"ok":true,"data":{"value":100}}
```

### read_float — 32-bit float

```
read_float <addr>
```

Reads a 32-bit float. Use for positions, angles, speed, cooldown timers.

```bash
probe.exe "read_float 0x055EA1C28FE8+0x40"
# {"ok":true,"data":{"value":1234.5}}
```

### read_string — Null-terminated string

```
read_string <addr> [max_length]
```

Reads a null-terminated ASCII string. Default max 256 bytes. Use for entity names, class names, debug text.

```bash
probe.exe "read_string 0x7FFB22EA4098"
# {"ok":true,"data":{"value":"player_controller","length":17}}
```

### read_chain — Follow pointer chain

```
read_chain <addr> <off1> <off2> ...
```

Each offset: add to current address, dereference (read 8-byte pointer), move to result. Last value is the final read. Use for navigating nested structs.

```bash
# EntityList → first chunk → first entity pointer
probe.exe "read_chain client.dll+0x3862C28 0x10 0x0"
# {"ok":true,"data":{"value":"0x055EA1C28FE8"}}
```

### dump — Hex dump with ASCII

```
dump <addr> <size>
```

Max 256 bytes. Returns formatted hex dump with ASCII sidebar. Use for visual inspection of unknown memory.

```bash
probe.exe "dump 0x055EA1C28FE8 64"
```

---

## Writing Memory

**CRITICAL SAFETY RULES:**

1. **Always read before writing.** Record the original value so you can restore it.
2. **Never write to code sections** unless intentionally patching instructions.
3. **Verify the address is valid** — read it first. If the read fails, do not write.
4. **Prefer data sections** (heap, globals, stack) over code sections.
5. **Small writes only** — write the minimum bytes needed.

### write — Raw hex bytes

```
write <addr> <hex_bytes>
```

Writes raw bytes. Use for patching instructions (NOP = `90`, INT3 = `CC`), writing arbitrary byte sequences.

```bash
# Read original bytes first!
probe.exe "read 0x7FFB21234000 5"
# {"ok":true,"data":{"hex":"E8AB123400"}}

# NOP out a 5-byte CALL instruction
probe.exe "write 0x7FFB21234000 9090909090"
# {"ok":true,"data":{"bytes_written":5}}

# RESTORE original bytes when done
probe.exe "write 0x7FFB21234000 E8AB123400"
```

### write_int — 32-bit integer

```
write_int <addr> <value>
```

Writes a signed 32-bit integer. Use for health, flags, counters, enum values.

```bash
# Read current value
probe.exe "read_int 0x055EA1C28FE8+0x354"
# {"ok":true,"data":{"value":100}}

# Set health to 999
probe.exe "write_int 0x055EA1C28FE8+0x354 999"
# {"ok":true,"data":{"bytes_written":4}}

# Enable a bool (stored as int32)
probe.exe "write_int 0x055EA1C29038 1"
# {"ok":true,"data":{"bytes_written":4}}
```

### write_float — 32-bit float

```
write_float <addr> <value>
```

Writes a 32-bit float. Use for positions, speed multipliers, timers.

```bash
# Read current position
probe.exe "read_float 0x055EA1C28FE8+0x40"
# {"ok":true,"data":{"value":1234.5}}

# Modify position
probe.exe "write_float 0x055EA1C28FE8+0x40 5000.0"
```

### write_ptr — 8-byte pointer

```
write_ptr <addr> <value>
```

Writes an 8-byte pointer value. Use for redirecting pointers, modifying vtable entries (dangerous).

```bash
# Read current pointer
probe.exe "read_ptr 0x055EA1C28FE8+0x330"
# {"ok":true,"data":{"value":"0x055EB2340000"}}

# Overwrite pointer (be very careful!)
probe.exe "write_ptr 0x055EA1C28FE8+0x330 0x055EB2350000"
```

---

## Common Recipes

### Toggle a boolean field

```bash
# Read current value
probe.exe "read_int 0x055EA1C29038"
# 0 = off

# Enable it
probe.exe "write_int 0x055EA1C29038 1"

# Disable it
probe.exe "write_int 0x055EA1C29038 0"
```

### Set health to max

```bash
# Read max health
probe.exe "read_int 0x055EA1C28FE8+0x350"
# {"ok":true,"data":{"value":625}}

# Read current health
probe.exe "read_int 0x055EA1C28FE8+0x354"
# {"ok":true,"data":{"value":100}}

# Set current = max
probe.exe "write_int 0x055EA1C28FE8+0x354 625"
```

### NOP a function call (skip it)

```bash
# Read the CALL instruction bytes (5 bytes for E8 calls)
probe.exe "read 0x7FFB21234000 5"
# Save: E8 AB 12 34 00

# NOP it out
probe.exe "write 0x7FFB21234000 9090909090"

# Restore later
probe.exe "write 0x7FFB21234000 E8AB123400"
```

### Modify a float via bridge batch

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"activity","status":"working","message":"Writing memory..."},
    {"action":"probe","command":"read_float 0x055EA1C28FE8+0x40"},
    {"action":"probe","command":"write_float 0x055EA1C28FE8+0x40 5000.0"},
    {"action":"probe","command":"read_float 0x055EA1C28FE8+0x40"},
    {"action":"activity","status":"idle","message":"Done"}
  ]
}'
```

---

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| `"error":"Invalid address"` | Address format wrong | Use hex with or without 0x prefix |
| `"error":"Read failed"` | Address not readable | Verify with `dump` first; entity may be dead or freed |
| `"error":"Write failed"` | Address not writable | Code sections are read-only; data/heap is writable |
| `"error":"Connection refused"` | Probe not running | Inject DLL first, or check port 9998 |
| `"error":"Size exceeds maximum"` | Read >4096 bytes | Split into multiple reads |

## Tips

- `modules list` gives you base addresses — add offsets to get absolute addresses
- Entity addresses change every match — use pointer chains, not hardcoded addresses
- Writing to networked values (health, position) may be overwritten by the server on the next tick
- Hardware breakpoints (`hwbp set <addr> w 4`) let you find what code writes to an address — useful for finding where to patch
- `watch <addr> <size>` polls for changes — use this to verify your writes are sticking
- Bool fields in Source 2 are sometimes stored as `int32` (4 bytes) and sometimes as `uint8` (1 byte) — check the schema dump
