# ARC Probe - Agent Instructions

ARC Probe is a process memory inspection toolkit designed for LLM agents. It consists of a lightweight DLL injected into a target Windows process that exposes 65 commands over TCP. You read, write, disassemble, and search process memory through any of the three command delivery methods below.

## Prerequisites

- ARC Probe DLL must be injected into the target process
- The probe's TCP server must be listening (default: `127.0.0.1:9998`)

## How to Send Commands

There are three ways to send commands to the probe. Use whichever is available:

### 1. CLI (simplest, always available)

```bash
probe.exe "<command>"
```

Returns JSON to stdout. Exit code 0 = `ok:true`, 1 = error.

```bash
probe.exe "status"
probe.exe "read_int 0x055EA1C28FE8+0x354"
probe.exe "write_int 0x055EA1C28FE8+0x354 999"
```

### 2. TCP (direct socket, for scripting)

Connect to `127.0.0.1:9998`. Send `command\n`, receive `{"ok":true,"data":{...}}\n`.

```powershell
$tcp = New-Object System.Net.Sockets.TcpClient('127.0.0.1', 9998)
$s = $tcp.GetStream()
$w = New-Object System.IO.StreamWriter($s); $w.AutoFlush = $true
$r = New-Object System.IO.StreamReader($s)
$w.WriteLine('read_int 0x055EA1C28FE8+0x354')
$r.ReadLine()
$tcp.Close()
```

### 3. HTTP Bridge (when GUI is running)

```bash
curl -s -X POST http://localhost:9996 \
  -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"read_int 0x055EA1C28FE8+0x354"}'
```

The bridge also supports `navigate`, `store`, `activity`, and `batch` actions for GUI control. See the `probe-bridge` skill for details.

### MCP Tools (optional)

If the MCP server is connected, commands are available as `probe_*` tools (e.g., `probe_read_int`, `probe_write_float`). The MCP server wraps the same TCP commands.

## Tool Reference

### Process Tools

| Tool | Purpose | When to Use |
|------|---------|-------------|
| `probe_status` | Process info, PID, module bases | First thing after connecting -- verify the probe is alive |
| `probe_help` | List all TCP commands or example workflows | Discovering available commands |
| `probe_unload` | Eject the DLL | When done with the session |

### Memory Read Tools

| Tool | Purpose | When to Use |
|------|---------|-------------|
| `probe_read` | Raw bytes (hex) | Reading opcode bytes, raw data |
| `probe_read_pointer` | 8-byte pointer | Following pointer fields in structs |
| `probe_read_int` | 32-bit integer | Reading known int fields (health, team, etc.) |
| `probe_read_float` | 32-bit float | Reading float fields (position, angles) |
| `probe_read_string` | Null-terminated string | Reading entity names, class names |
| `probe_read_chain` | Follow pointer chain | Navigating nested structs: base -> field -> field |
| `probe_dump` | Hex dump with ASCII | Exploring unknown memory regions (max 256 bytes) |

### Memory Write Tools

| Tool | Purpose | When to Use |
|------|---------|-------------|
| `probe_write` | Write raw hex bytes | Patching instructions (NOP, JMP) |
| `probe_write_int` | Write 32-bit integer | Modifying integer fields |
| `probe_write_float` | Write 32-bit float | Modifying float fields |
| `probe_write_pointer` | Write 8-byte pointer | Modifying pointer fields |

**WARNING:** Writing to memory can crash the target process. Always:
1. Read the original bytes first and record them
2. Never write to code sections unless you are patching instructions
3. Prefer writing to data sections (stack, heap, globals)

### Search Tools

| Tool | Purpose | When to Use |
|------|---------|-------------|
| `probe_pattern_scan` | IDA-style byte pattern scan | Finding functions, global variables |
| `probe_find_value` | Search for a specific value (experimental) | Finding where a known value lives in memory |

### Analysis Tools

| Tool | Purpose | When to Use |
|------|---------|-------------|
| `probe_disassemble` | Disassemble N instructions | Reading code at an address |
| `probe_disassemble_function` | Disassemble until RET | Getting a complete function |
| `probe_modules` | List loaded modules | Finding module base addresses |
| `probe_module_info` | Detailed module info (sections, exports) | Analyzing a specific DLL |
| `probe_generate_signature` | Create a byte signature | Making a pattern that survives updates |
| `probe_test_signature` | Test a pattern for matches (experimental) | Verifying a signature is unique |

### RTTI Tools

| Tool | Purpose | When to Use |
|------|---------|-------------|
| `probe_rtti_resolve` | Resolve C++ class name from object pointer | Identifying what type an object is |
| `probe_rtti_scan` | Scan module for all RTTI classes | Discovering all virtual classes in a DLL |
| `probe_rtti_hierarchy` | Get inheritance chain for a class | Understanding class hierarchy |
| `probe_rtti_find` | Find RTTI by name substring | Searching for a class by partial name |
| `probe_rtti_vtable` | Get vtable address and entries | Mapping virtual functions for a class |

### String Scanner Tools

| Tool | Purpose | When to Use |
|------|---------|-------------|
| `probe_strings_scan` | Scan module for printable strings | Discovering strings in a DLL |
| `probe_strings_find` | Search for specific text in memory | Finding where a known string lives |
| `probe_strings_at` | Read string at a specific address | Reading a string you found a pointer to |
| `probe_strings_xref` | Find code references to a string | Finding functions that use a string |

### PE Header Tools

| Tool | Purpose | When to Use |
|------|---------|-------------|
| `probe_pe_header` | Full PE header info | Inspecting module metadata, timestamps |
| `probe_pe_sections` | List PE sections with flags | Understanding memory layout of a DLL |
| `probe_pe_exports` | List exported functions | Finding available API functions |
| `probe_pe_imports` | List imported DLLs and functions | Understanding module dependencies |

### Interface Tools (Source 2 / CreateInterface)

| Tool | Purpose | When to Use |
|------|---------|-------------|
| `probe_interfaces_list` | List all interfaces | Discovering available interfaces |
| `probe_interfaces_vtable` | Read interface vtable | Mapping virtual function indices |
| `probe_interfaces_dump` | Full interface analysis | Understanding an interface completely |

### Debugging Tools — Software Breakpoints

| Tool | Purpose | When to Use |
|------|---------|-------------|
| `probe_breakpoint_set` | Software breakpoint (INT3) | Catching when code executes |
| `probe_breakpoint_remove` | Remove software breakpoint | Cleanup |
| `probe_breakpoint_list` | List all breakpoints | Managing breakpoints |
| `probe_breakpoint_log` | Get register snapshots from hits | Analyzing state at breakpoint |
| `probe_breakpoint_enable` | Re-enable a disabled breakpoint | Toggling breakpoints |
| `probe_breakpoint_disable` | Disable without removing | Temporarily pausing a breakpoint |
| `probe_breakpoint_diag` | VEH handler diagnostics | Debugging breakpoint issues |

### Debugging Tools — Hardware Breakpoints

| Tool | Purpose | When to Use |
|------|---------|-------------|
| `probe_hwbp_set` | Hardware breakpoint (DR0-DR3) | Catching memory reads/writes (4 slots max) |
| `probe_hwbp_remove` | Remove hardware breakpoint | Cleanup |
| `probe_hwbp_list` | List hardware breakpoints | Checking DR slot usage |

### Thread Tools

| Tool | Purpose | When to Use |
|------|---------|-------------|
| `probe_threads_list` | List all threads in process | Enumerating threads, finding TIDs |
| `probe_threads_info` | Thread details (TEB, stack, priority) | Deep-diving a specific thread |
| `probe_threads_stack` | Walk call stack for a thread | Tracing call chains |

### Register Tools

| Tool | Purpose | When to Use |
|------|---------|-------------|
| `probe_registers` | Dump thread registers | Inspecting CPU state |
| `probe_stackwalk` | Walk call stack | Tracing call chains (legacy) |

### Memory Watch

| Tool | Purpose | When to Use |
|------|---------|-------------|
| `probe_watch` | Poll memory for changes | Detecting when a value changes |
| `probe_unwatch` | Stop watching | Cleanup |
| `probe_watchlist` | List active watches | Managing watches |

## Addresses

All addresses are hexadecimal strings. Accepted formats:
- `0x7FF612340000` (with prefix)
- `7FF612340000` (without prefix)

Module-relative addresses: First get the module base with `probe_modules`, then add the RVA (relative virtual address) to get the absolute address.

```
absolute_address = module_base + RVA
```

## Pattern Scan Syntax

Patterns use space-separated hex bytes with `??` wildcards:

```
48 8B 05 ?? ?? ?? ??    -- MOV RAX, [RIP+??]
E8 ?? ?? ?? ??          -- CALL rel32
48 89 5C 24 ??          -- MOV [RSP+?], RBX
```

Wildcards (`??`) match any byte. Use them for:
- RIP-relative offsets (change between builds)
- Stack offsets (vary by compiler)
- Register encodings when you only care about the opcode

## RIP-Relative Address Resolution

Many x86-64 instructions use RIP-relative addressing. To resolve the target:

```
target = instruction_address + instruction_length + int32_at(instruction_address + rip_offset)
```

For a typical `MOV RAX, [RIP+disp32]` (`48 8B 05 XX XX XX XX`):
- instruction_length = 7
- rip_offset = 3 (where the 4-byte displacement starts)
- target = addr + 7 + int32_at(addr + 3)

For a `CALL rel32` (`E8 XX XX XX XX`):
- instruction_length = 5
- rip_offset = 1
- target = addr + 5 + int32_at(addr + 1)

## Common Workflows

### 1. Find a Function by String Reference

Goal: Locate a function that references a known string (e.g., an error message).

```
1. probe_find_value with type "string" -- search for the string
2. Note the string address
3. probe_pattern_scan -- scan for LEA instructions referencing that address
   Pattern: "48 8D ?? ?? ?? ?? ??" near the string's RVA
4. probe_disassemble -- disassemble at the LEA to see the surrounding function
5. Scroll up to find the function prologue (look for push rbp / sub rsp patterns)
```

### 2. Map Out a Struct from a Base Address

Goal: Determine the layout of an unknown data structure.

```
1. probe_dump {address, 256}      -- get a hex overview
2. Identify pointer-like values   -- 8-byte aligned values in the 0x7FF... range
3. For each pointer candidate:
   a. probe_read_pointer          -- verify it's a valid pointer
   b. probe_dump                  -- see what it points to
   c. probe_read_string           -- check if it's a string pointer
4. Identify numeric fields        -- look for reasonable int/float values
5. probe_read_int / probe_read_float to confirm
6. For vtable pointers (usually at offset 0x0):
   a. probe_read_pointer {address} -- read vtable ptr
   b. probe_dump {vtable, 128}     -- dump vtable entries
   c. probe_disassemble on entries -- see what the virtual functions do
```

### 3. Find What Writes to an Address

Goal: Discover what code modifies a memory location.

```
1. probe_hwbp_set {address, access: "w", size: "4"}
   -- set a write watchpoint
2. Wait for the value to change (trigger the action in the process)
3. probe_breakpoint_log {id}
   -- get the register state including RIP
4. probe_disassemble {rip_address}
   -- see the instruction that wrote to the address
5. probe_disassemble_function
   -- get the full function for context
6. probe_hwbp_remove {id}
   -- clean up
```

### 4. Trace a Call Chain

Goal: Understand the call hierarchy leading to a function.

```
1. probe_breakpoint_set {function_address}
2. Trigger the function (perform the action)
3. probe_breakpoint_log {id}
   -- RSP points to the return address stack
4. probe_read_pointer {rsp}
   -- read return address (caller)
5. probe_disassemble {caller - 0x20, 10}
   -- see the CALL instruction in context
6. Repeat: read next return address at RSP+8, RSP+16, etc.
   Or use probe_stackwalk for automated stack walking.
```

### 5. Identify a Class by RTTI

Goal: Determine the C++ class name from an object pointer.

```
1. probe_read_pointer {object}
   -- read vtable pointer (first 8 bytes of object)
2. probe_read_pointer {vtable - 8}
   -- read RTTI Complete Object Locator pointer (vtable[-1])
3. probe_read_int {col + 0xC}
   -- read TypeDescriptor RVA from COL
4. Add RVA to module base to get TypeDescriptor address
5. probe_read_string {type_desc + 0x10}
   -- read the mangled class name
   -- Format: ".?AVClassName@@" -> strip ".?AV" and "@@"
```

### 6. Discover Interfaces (Source 2 processes)

Goal: Find and map Source 2 engine interfaces.

```
1. probe_interfaces_list           -- list all interfaces in all modules
2. probe_interfaces_vtable {name}  -- read the vtable
3. probe_disassemble on each entry -- understand what each virtual function does
4. probe_interfaces_dump {name}    -- automated analysis with safe probing
```

## Interpreting Disassembly

Key patterns to recognize:

**Function prologue:**
```asm
push rbp           ; or push rbx, push rdi, etc.
sub rsp, 0x??      ; allocate stack frame
```

**Function epilogue:**
```asm
add rsp, 0x??      ; deallocate stack frame
pop rbp
ret
```

**Virtual function call:**
```asm
mov rax, [rcx]        ; load vtable from this pointer
call [rax + 0x??]     ; call vtable entry at index ??/8
```

**Global variable access:**
```asm
mov rax, [rip + 0x????]  ; RIP-relative load of global
```

**Structure field access:**
```asm
mov eax, [rcx + 0x354]   ; read field at offset 0x354 from object in RCX
```

## Pointer Chains

A pointer chain navigates nested structures. For example, to read a player's health:

```
EntityList -> Entity[index] -> m_iHealth
```

Using `probe_read_chain`:
- address: entity list base
- offsets: [index * stride, health_offset]

Each offset is added to the dereferenced pointer from the previous step. The chain stops and returns the final value.

## Safety Guidelines

1. **Never write to executable code** unless you are intentionally patching instructions and have recorded the original bytes.
2. **Limit read sizes** to 4096 bytes per call -- the probe enforces this limit.
3. **Remove breakpoints** when done -- leftover INT3 bytes will crash the process.
4. **Don't rapid-fire commands** -- allow ~100ms between calls to avoid overwhelming the TCP server.
5. **Verify addresses** before writing -- read first to confirm the address is valid.
6. **Record original values** before any write operation so you can restore them.
7. **Use `probe_status`** to verify the probe is healthy before starting work.
