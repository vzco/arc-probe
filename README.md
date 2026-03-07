# arc probe

if cheat engine and claude had a baby.

arc probe is a **real-time process memory inspector built for AI agents**. inject a DLL into any Windows x64 process, then read memory, disassemble x86-64, scan patterns, set breakpoints, resolve RTTI — all through **structured JSON over TCP**.

the GUI exists to show the human what the AI is doing. a human can drive it. claude can drive it. or both can work together.

- **65 commands** over TCP — memory read/write, disassembly, RTTI, pattern scan, breakpoints, hooks, threads
- **agent-first design** — Claude Bridge API lets AI control the GUI programmatically
- **struct editor** with live memory values and recursive pointer drill-down
- **works with any x64 process** — no game-specific code required

```bash
# inject into a running process
probe-inject.exe --pid 1234

# read an integer from memory
probe.exe "read_int client.dll+0x354"

# find a C++ class by RTTI
probe.exe "rtti find CitadelPlayerPawn client.dll"

# pattern scan with wildcard bytes and auto-resolve
probe.exe "pattern 48 8B 0D ?? ?? ?? ?? 48 85 C9 client.dll --resolve 3"
```

> **status:** early access. download binaries from [Releases](https://github.com/vzco/arc-probe/releases), install the Claude Code plugin, and start inspecting.

---

## what it looks like

### struct editor — live memory values

define structs from schema dumps or manually, then watch values update in real-time. pointer fields expand inline to show nested structures. the AI builds these programmatically through the Claude Bridge.

![struct editor with live values](docs/images/struct-live-values.png)

### hex viewer

navigate any address in the process. PE headers, entity memory, vtables — click any byte to see every type interpretation in the data inspector. bookmarks and labels for quick navigation.

![hex viewer](docs/images/hex-viewer.png)

### disassembler

color-coded x86-64 disassembly with resolved addresses and labels. trace function calls, identify globals via RIP-relative addressing, understand code flow.

![disassembler](docs/images/disasm-view.png)

### console — structured JSON

every command returns structured JSON. the console renders it with syntax highlighting and collapsible trees. type commands directly or let the AI drive.

![console](docs/images/console.png)

### struct hierarchy — full inheritance chain

the AI built 12 struct definitions covering a 14-level C++ inheritance chain — 218 fields total — all from schema dumps + live memory verification. pointer drill-down lets you expand nested structs inline.

![struct hierarchy](docs/images/struct-hierarchy.png)

---

## how it works

three binaries:

```
probe-shell.dll          probe-inject.exe          probe.exe (CLI)
(inside target process)  (manual-map injector)     (sends commands)
        |                        |                        |
        +-- TCP server ---------|------------------------+
        |   127.0.0.1:9998      |
        +-- Command dispatcher (65 commands)
        +-- Memory engine (SEH-protected reads/writes)
        +-- Zydis x86-64 disassembler
        +-- VEH breakpoint engine (software + hardware)
        +-- RTTI scanner, PE parser
        +-- String scanner, thread inspector
        +-- MinHook function hooking
```

**protocol:** send `command\n` over TCP, receive `{"ok":true,"data":{...}}\n`. all memory access is SEH-wrapped — bad addresses return errors, never crash.

**the GUI** is a Tauri v2 app (Rust backend + React frontend) that connects to the same TCP server. the **Claude Bridge** (`localhost:9996`) accepts JSON POST requests to drive the GUI — create structs, set labels, navigate tabs, read memory — all programmatically.

**the injector** uses manual PE mapping. the DLL doesn't appear in the target's module list.

---

## what the AI can do

arc probe is designed so an AI agent can perform the entire reverse engineering workflow autonomously:

1. **connect** to the target process and enumerate modules
2. **discover all functions** in a module — merging .pdata boundaries, PE exports, and RTTI vtable names
3. **scan cross-references** — find every CALL, JMP, LEA, and MOV that references a given address
4. **discover classes** via RTTI scanning and inheritance resolution
5. **find globals** through pattern scanning with RIP-relative resolution
6. **build struct definitions** from schema dumps, verified against live memory
7. **wire pointer drill-down** between structs for recursive exploration
8. **set breakpoints** to trace what code reads/writes specific addresses
9. **annotate everything** with labels, bookmarks, function names, and comments in the GUI

the human sees every step in the GUI as it happens. both can work simultaneously — the AI automates the tedious memory mapping while you focus on understanding the results.

```bash
# the AI sends this to build a struct via the Claude Bridge
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action": "batch",
  "actions": [
    {"action": "store", "store": "struct", "method": "createStruct",
     "args": ["C_BaseEntity", "0x439D9F49400", 1536]},
    {"action": "store", "store": "struct", "method": "addField",
     "args": ["C_BaseEntity", 852, "int32", "m_iHealth"]},
    {"action": "store", "store": "struct", "method": "addField",
     "args": ["C_BaseEntity", 816, "pointer", "m_pGameSceneNode"]},
    {"action": "navigate", "tab": "structs"}
  ]
}'
```

---

## command reference

### memory

| command | description |
|---------|-------------|
| `read <addr> <size>` | raw bytes (hex), max 4096 |
| `read_ptr <addr>` | 8-byte pointer |
| `read_int <addr>` | 32-bit integer |
| `read_float <addr>` | 32-bit float |
| `read_string <addr>` | null-terminated string |
| `read_chain <addr> <off1> ...` | follow pointer chain |
| `dump <addr> <size>` | hex dump with ASCII |
| `write <addr> <hex>` | write raw bytes |
| `write_int <addr> <val>` | write 32-bit integer |

### search & disassembly

| command | description |
|---------|-------------|
| `pattern <hex> [module]` | IDA-style byte pattern with `??` wildcards |
| `disasm <addr> [count]` | disassemble N instructions |
| `disasm_function <addr>` | disassemble until RET |
| `generate_sig <addr>` | create a byte signature for a function |

### RTTI & modules

| command | description |
|---------|-------------|
| `rtti find <name>` | search for class by partial name |
| `rtti hierarchy <class>` | get inheritance chain |
| `rtti vtable <class>` | vtable address and entries |
| `modules list` | loaded modules with bases |
| `modules info <name>` | detailed module info |
| `pe exports <module>` | exported functions |

### breakpoints

| command | description |
|---------|-------------|
| `bp set <addr>` | software breakpoint (INT3) |
| `hwbp set <addr> <access> <size>` | hardware breakpoint (DR0-DR3) |
| `bp log <id>` | register snapshots from hits |
| `threads stack <tid>` | walk call stack |

[full command reference (65 commands) &rarr;](docs/commands.md)

---

## installation

### 1. download binaries

grab the latest release from [Releases](https://github.com/vzco/arc-probe/releases):

| file | purpose |
|------|---------|
| `probe.exe` | CLI client — sends commands, prints JSON |
| `probe-inject.exe` | manual-map DLL injector |
| `probe-shell.dll` | injected DLL (TCP server + all tools) |

extract to a directory and add it to your PATH, or use full paths.

**requirements:**
- Windows 10/11 x64
- administrator privileges (for cross-process memory access)

### 2. install the claude code plugin

```bash
/plugin marketplace add vzco/arc-probe
/plugin install arc-probe
```

this gives you 14 reverse engineering skills:

| skill | description |
|-------|-------------|
| `/arc-probe:inject` | inject into a process and verify connection |
| `/arc-probe:analyze-module` | deep module analysis — exports, RTTI, strings |
| `/arc-probe:map-struct` | map a data structure from a memory address |
| `/arc-probe:find-function` | locate a function by behavior or strings |
| `/arc-probe:identify-class` | identify a C++ class via RTTI + vtable |
| `/arc-probe:trace-writes` | find what code writes to an address |
| `/arc-probe:find-string-xref` | trace from a string to referencing code |
| `/arc-probe:follow-pointers` | navigate nested pointer chains |
| `/arc-probe:make-signature` | generate byte signatures that survive updates |
| `/arc-probe:resolve-rip` | resolve RIP-relative addresses |
| `/arc-probe:re-handbook` | full RE techniques reference |
| `/arc-probe:probe-bridge` | drive the GUI via Claude Bridge HTTP API |
| `/arc-probe:probe-gui-struct` | build structs in the GUI from live memory |
| `/arc-probe:probe-entity-scan` | scan game entities (Source 2 / Deadlock) |

### 3. inject and go

```bash
# inject into your target process
probe-inject.exe notepad.exe

# verify the connection
probe.exe ping

# start exploring
probe.exe status
probe.exe "rtti scan Notepad.exe"
probe.exe "dump 0x7FF701AB0000 64"
```

or let Claude do it: `/arc-probe:inject`

---

## tutorials

### part 1: getting started

the full walkthrough — from connecting to a live process, through RTTI discovery, pattern scanning, and building 12 annotated struct definitions with 218 fields and recursive pointer drill-down — all driven by Claude as an AI agent.

**[read part 1 &rarr;](docs/tutorial.md)**

### part 2: advanced static analysis

function discovery, cross-reference scanning, RTTI deep dives, vtable disassembly, Source 2 interface enumeration, and the complete agent-driven analysis workflow. 16 screenshots from a live Deadlock session.

**[read part 2 &rarr;](docs/advanced-tutorial.md)**

---

## components

| component | description |
|-----------|-------------|
| `probe-shell.dll` (2.7 MB) | injected DLL: TCP server, 65 commands, Zydis x86-64 disassembler, VEH breakpoints, RTTI scanner, PE parser, string scanner, MinHook |
| `probe-inject.exe` (710 KB) | manual-map injector (relocations, imports, TLS, SEH/.pdata, header erasure) |
| `probe.exe` (181 KB) | CLI client: send command, print JSON, exit. REPL mode with no args |
| GUI (Tauri v2) | desktop app — hex viewer, disassembler, struct editor, scanner, CFG viewer |
| Claude Code plugin | 14 RE skills + agent instructions for autonomous reverse engineering |

### vendor dependencies

| library | version | license | purpose |
|---------|---------|---------|---------|
| [Zydis](https://github.com/zyantific/zydis) | 4.1.1 | MIT | x86-64 instruction decoder and formatter |
| [MinHook](https://github.com/TsudaKageworst/minhook) | latest | BSD 2-Clause | x86/x64 function hooking |

---

## license

ARC Probe is source-available under a commercial license. Free for personal use, education, security research, and CTF competitions. Commercial use requires a paid license.

See [LICENSE](LICENSE) for full terms.
