---
name: resolve-rip
description: Resolve a RIP-relative address from a disassembled x86-64 instruction
---

# /probe:rip

Resolve a RIP-relative address from a disassembled instruction.

## When to use

You see a disassembled instruction like:
```
mov rax, [rip+0x1A2B3C]     ; loads a global variable
lea rcx, [rip+0x5678]        ; loads the address of a string or struct
call 0x7FF612345678           ; calls a function (already resolved by Zydis)
```

The `[rip+disp32]` addressing means the target address depends on where the instruction is. You need to resolve it to an absolute address.

## The formula

```
target = instruction_address + instruction_length + displacement
```

Where:
- `instruction_address` = the address shown in the disassembly
- `instruction_length` = number of bytes in the instruction (sum up the hex bytes)
- `displacement` = the signed 32-bit value encoded in the instruction

## Common instruction patterns

### MOV reg, [RIP+disp32] — Load a global pointer
```
48 8B 05 XX XX XX XX     ; MOV RAX, [RIP+disp32]
```
- instruction_length = 7
- displacement starts at byte 3 (offset +3)
- `target = addr + 7 + int32_at(addr + 3)`

### LEA reg, [RIP+disp32] — Load address of string/data
```
48 8D 0D XX XX XX XX     ; LEA RCX, [RIP+disp32]
48 8D 15 XX XX XX XX     ; LEA RDX, [RIP+disp32]
4C 8D 05 XX XX XX XX     ; LEA R8,  [RIP+disp32]
```
- instruction_length = 7
- displacement starts at byte 3
- `target = addr + 7 + int32_at(addr + 3)`

### CALL rel32 — Relative function call
```
E8 XX XX XX XX           ; CALL rel32
```
- instruction_length = 5
- displacement starts at byte 1
- `target = addr + 5 + int32_at(addr + 1)`

### JMP rel32 — Relative jump
```
E9 XX XX XX XX           ; JMP rel32
```
- Same as CALL: `target = addr + 5 + int32_at(addr + 1)`

### Conditional jump rel32
```
0F 84 XX XX XX XX        ; JE rel32
0F 85 XX XX XX XX        ; JNE rel32
```
- instruction_length = 6
- displacement starts at byte 2
- `target = addr + 6 + int32_at(addr + 2)`

## Steps

1. **Disassemble the instruction**:
   ```
   probe_disassemble address=<addr> count=1
   ```
   Note the instruction bytes and text.

2. **Read the raw displacement** (4 bytes, little-endian signed):
   ```
   probe_read address=<addr + rip_offset> size=4
   ```
   Where `rip_offset` is where the 4-byte displacement starts (3 for MOV/LEA, 1 for CALL/JMP, 2 for Jcc).

3. **Calculate the target**:
   ```
   target = instruction_address + instruction_length + signed_displacement
   ```

4. **Verify the target**:
   ```
   probe_read_pointer address=<target>    ; if it's a global pointer
   probe_read_string address=<target>     ; if it's a string reference
   probe_disassemble address=<target>     ; if it's a function call
   ```

## Shortcut: use --resolve

The `probe_pattern_scan` command has a `--resolve` flag that does this automatically:
```
probe_pattern_scan pattern="48 8B 05 ?? ?? ?? ??" module="client.dll" resolve=3
```
This scans for the pattern and resolves the RIP-relative address at offset 3, returning the absolute target.

## Signed displacement reminder

The displacement is a **signed** 32-bit integer. Negative values mean the target is BEFORE the instruction:
- `FF FF FF E0` = -32 in signed int32 → target is 32 bytes before (instruction_address + instruction_length - 32)
- `00 00 10 00` = 4096 → target is 4096 bytes after

Always treat the 4 bytes as signed when calculating.
