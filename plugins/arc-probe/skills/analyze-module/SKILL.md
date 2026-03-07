---
name: analyze-module
description: Deep analysis of a loaded module — exports, RTTI classes, strings, key functions
---

# /probe:analyze

Deep analysis of a loaded module in the target process.

## Arguments

- `module` (required): Module name (e.g., "target.dll", "engine.dll")

## Steps

1. **Get module info** -- Call `probe_modules` to list all loaded modules. Find the target module and note its base address and size. If the module is not found, report available modules and stop.

2. **Module overview** -- Report:
   - Base address
   - Size
   - Full path (if available)

3. **Export table** -- Call `probe_pattern_scan` with the module name to locate the PE export directory. Alternatively, use `probe_dump` at the module base to read the PE headers:
   - Read DOS header at base (MZ signature at offset 0)
   - Read PE signature offset at base + 0x3C
   - Navigate to the Optional Header to find the Export Directory RVA
   - Dump the export directory to enumerate exported functions

4. **RTTI scan** -- Scan for RTTI type descriptors to identify C++ classes:
   - Pattern scan for `.?AV` (RTTI type info string prefix) within the module
   - For each match, read the class name string
   - Group by namespace if possible
   - This reveals the class hierarchy implemented in the module

5. **String references** -- Scan for interesting strings:
   - Pattern scan for common string patterns (error messages, debug prints, format strings)
   - Use `probe_find_value` for specific strings if looking for something particular
   - Strings near code often indicate function purpose

6. **Key function identification** -- For each interesting string or export:
   - Find cross-references (LEA instructions pointing to the string)
   - Disassemble the referencing function
   - Note the function address and purpose

7. **Report** -- Present a structured analysis:
   ```
   Module: target.dll
   Base: 0x7FF612340000
   Size: 0x1A0000

   Exports: (list)
   Classes (RTTI): (list with vtable addresses)
   Key Functions: (list with addresses and descriptions)
   Notable Strings: (list with addresses)
   ```

## Tips

- Start broad (modules, exports, RTTI) then narrow down to specific functions
- RTTI class names are the fastest way to understand what a module does
- Export names often follow naming conventions that reveal the module's API
- String references are the most reliable way to find specific functionality
