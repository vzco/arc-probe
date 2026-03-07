---
name: trace-writes
description: Find what code reads or writes to a specific memory address using hardware breakpoints
---

# /probe:trace

Find what code reads or writes to a specific memory address.

## Arguments

- `address` (required): Memory address to monitor (hex)
- `access` (optional): "w" for writes only (default), "rw" for reads and writes
- `size` (optional): Watch size in bytes: 1, 2, 4, or 8 (default: 4)

## Steps

1. **Set a hardware watchpoint**:
   ```
   probe_hwbp_set address=<addr> access=<w|rw> size=<size>
   ```
   You get back a breakpoint ID. Hardware breakpoints use debug registers DR0-DR3 (4 slots max).

2. **Wait for the target action** -- Tell the user what to do in the application to trigger the write. For example: "Take damage in-game" or "Click the button" or "Send a message".

3. **Read the breakpoint log**:
   ```
   probe_breakpoint_log id=<bp_id>
   ```
   Each hit gives you a full register snapshot: RIP (the instruction that accessed the address), RAX-R15, RSP, RFLAGS.

4. **Disassemble the writer**:
   ```
   probe_disassemble address=<rip> count=10
   ```
   Look at the instruction at RIP — it's the exact instruction that wrote (or read) the address.

5. **Get the full function**:
   ```
   probe_disassemble_function address=<function_start>
   ```
   Walk backwards from RIP to find the function prologue (look for `push rbp` / `sub rsp` / `push rbx` patterns). Or subtract 0x40-0x100 from RIP and disassemble — the prologue is usually within that range.

6. **Identify the caller**:
   - RSP from the breakpoint log points to the return address on the stack
   - `probe_read_pointer address=<rsp_value>` reads the return address
   - Disassemble there to see who called this function

7. **Generate a signature** for the function so it can be found again:
   ```
   probe_generate_signature address=<function_start>
   ```

8. **Clean up**:
   ```
   probe_hwbp_remove id=<bp_id>
   ```
   Always remove hardware breakpoints when done — there are only 4 slots.

## If the breakpoint doesn't fire

- The value might be written by a **different thread** — hardware BPs work across all threads, so this is unlikely. More likely the value is written **once during initialization** and you missed it.
- The value might be **computed, not stored** — the address might be read from somewhere else and cached. Try watching the source address instead.
- The write might happen via **DMA or kernel mode** — hardware breakpoints only catch usermode access. If you suspect kernel writes, this approach won't work.
- Try **polling instead**: `probe_watch address=<addr> size=4 interval_ms=100` to catch when the value changes, then narrow down the timing.

## Common patterns you'll see

**Direct field write:**
```asm
mov [rcx+0x354], eax    ; rcx = object pointer, 0x354 = field offset, eax = new value
```
This tells you: the function receives the object in RCX, and offset 0x354 is the field.

**Indirect write via pointer chain:**
```asm
mov rax, [rcx+0x30]     ; load sub-object pointer
mov [rax+0x10], edx     ; write to sub-object field
```
This tells you: the struct has a pointer at +0x30 to a sub-struct, and the actual field is at +0x10 in the sub-struct.

**Atomic/interlocked write:**
```asm
lock xchg [rcx+0x40], eax   ; thread-safe write
```
This means the field is accessed from multiple threads — important to know for reading it safely.
