---
name: identify-class
description: Identify a C++ class from an object pointer using RTTI and map its vtable
---

# /probe:class

Identify a C++ class from an object pointer using RTTI, then map its vtable.

## Arguments

- `address` (required): Address of the object instance (hex)
- `module` (optional): Module name to search within

## Steps

1. **Check if it's a C++ object** — read the first 8 bytes:
   ```
   probe_read_pointer address=<addr>
   ```
   If the value looks like a code address (in the 0x7FF... range), it's likely a vtable pointer. If it's NULL or a small value, this isn't a C++ object (or it hasn't been constructed yet).

2. **Resolve the class name via RTTI**:
   ```
   probe_rtti_resolve address=<addr>
   ```
   This walks: object -> vtable -> vtable[-1] (COL) -> TypeDescriptor -> mangled name.
   Returns the demangled class name.

3. **If `rtti resolve` fails**, do it manually:
   a. Read vtable pointer: `probe_read_pointer address=<addr>` → vtable_addr
   b. Read COL pointer (vtable - 8): `probe_read_pointer address=<vtable_addr - 8>` → col_addr
   c. Read TypeDescriptor RVA: `probe_read_int address=<col_addr + 0xC>` → td_rva
   d. Get module base from `probe_modules`
   e. Read mangled name: `probe_read_string address=<module_base + td_rva + 0x10>`
   f. Demangle: strip `.?AV` prefix and `@@` suffix → class name

4. **Get the inheritance chain**:
   ```
   probe_rtti_hierarchy class=<class_name> module=<module>
   ```
   Shows parent classes. Critical for understanding which fields come from which base class.

5. **Map the vtable**:
   ```
   probe_rtti_vtable class=<class_name> module=<module> limit=20
   ```
   This dumps virtual function entries with first-instruction disassembly previews. Look for:
   - **Tiny functions** (1-3 instructions): getters/setters, often return `this+offset` or set a field
   - **Large functions**: actual logic — these are worth deeper disassembly
   - **Thunks** (`jmp` to another function): inherited methods that just redirect

6. **Explore interesting vtable entries**:
   ```
   probe_disassemble_function address=<vtable_entry>
   ```
   For small getter functions, you can directly read what offset they access:
   ```asm
   mov eax, [rcx+0x354]   ; rcx = this, returns field at offset 0x354
   ret
   ```
   This tells you: vtable index N is a getter for field at offset 0x354.

7. **Report**:
   ```
   Class: C_BaseEntity (inherits: CEntityInstance)
   Vtable: 0x7FFB23456789 (42 entries)

   vtable[0]: 0x7FFB21234000 — destructor
   vtable[1]: 0x7FFB21234100 — getter: returns [this+0x354] (int32)
   vtable[5]: 0x7FFB21234500 — large function (~0x200 bytes), references "health"
   ...

   Known fields from vtable analysis:
     +0x354  int32    (getter at vtable[1])
     +0x35C  int32    (getter at vtable[3])
   ```

## If RTTI is stripped

Some binaries strip RTTI (`/GR-` compiler flag). Signs:
- vtable[-1] doesn't point to a valid COL (the read returns garbage)
- `rtti scan` returns very few or zero types for the module
- No `.?AV` strings in the module's .rdata section

Fallbacks when RTTI is stripped:
- **Check PE exports** — the class might be exported by name
- **Search for strings** — constructor/destructor often reference the class name in asserts or logs
- **Compare vtable sizes** — dump the vtable until you hit a non-code pointer, count entries
- **Look for debug symbols** — PDB files, if available, have full type info
- **Pattern match** — if you know the class from another build that has RTTI, match the vtable function patterns
