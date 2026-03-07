---
name: find-function
description: Locate a function by its behavior, string references, RTTI, or hardware breakpoints
---

# /probe:find

Locate a function in the target process by its behavior or string references.

## Arguments

- `description` (required): Natural language description of the function to find (e.g., "the function that handles player damage", "where the chat message is processed")
- `module` (optional): Module to search in (narrows the search)

## Steps

1. **Identify search strategy** based on the description:
   - If the description mentions a **string or message**: search for that string first
   - If it mentions a **class or method name**: scan for RTTI or known symbol patterns
   - If it mentions **behavior**: identify what data the function would access and work backwards

2. **String-based search** (most common path):
   a. Think about what strings the function might reference (error messages, log output, format strings, UI text)
   b. Call `probe_pattern_scan` or `probe_find_value (experimental)` with type "string" to find the string in the module's data section
   c. Note the string's address
   d. Search for LEA instructions that reference the string address:
      - Calculate the RIP-relative offset from likely code locations
      - Pattern: `48 8D ?? ?? ?? ?? ??` (LEA reg, [rip+disp32]) or `4C 8D ?? ?? ?? ?? ??`
   e. For each reference found, disassemble the surrounding code with `probe_disassemble`
   f. Walk backwards to find the function prologue (push rbp / sub rsp pattern)

3. **RTTI-based search** (for known class methods):
   a. Pattern scan for the class name string `.?AV<ClassName>@@`
   b. Find the TypeDescriptor, then the Complete Object Locator, then the vtable
   c. Dump the vtable to enumerate virtual functions
   d. Disassemble each vtable entry to find the target method

4. **Behavior-based search** (for data-touching functions):
   a. Identify what memory address the function reads or writes
   b. Set a hardware breakpoint on that address: `probe_hwbp_set` with type "w" or "rw"
   c. Trigger the behavior in the application
   d. Read the breakpoint registers to get the RIP of the writing instruction
   e. Disassemble the function containing that instruction

5. **Validate the result**:
   a. Disassemble the full function with `probe_disassemble_function`
   b. Check that it references the expected strings, calls expected APIs, or touches expected data
   c. Generate a signature with `probe_generate_signature` for future reference
   d. Test the signature with `probe_test_signature` to confirm it's unique

6. **Report**:
   ```
   Function: <description>
   Address: 0x7FF612345678
   Module: target.dll + 0x5678
   Size: ~0x1A0 bytes
   Signature: 48 89 5C 24 08 57 48 83 EC 20 48 8B D9 E8 ?? ?? ?? ??
   References: "error message string" at +0x42
   Calls: SubFunction at 0x7FF612346000
   ```

## Tips

- Most functions reference at least one string -- always try string search first
- If the function has no strings, look for unique constant values (magic numbers, enum values)
- Virtual functions are best found via RTTI -> vtable -> disassemble
- Hardware breakpoints are the most reliable way to find functions that modify specific data
- Always generate and test a signature so the function can be found again after updates
