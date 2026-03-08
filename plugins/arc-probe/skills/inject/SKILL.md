---
name: inject
description: Inject ARC Probe into a running process and verify the connection
---

# /probe:inject

Inject ARC Probe into a running process and verify the connection.

## Arguments

- `target` (required): Process name (e.g., "notepad.exe") or PID (e.g., "12345")

## Steps

1. Verify the target process is running:
   ```bash
   tasklist /FI "IMAGENAME eq <process_name>" /NH
   ```
   Or for PID:
   ```bash
   tasklist /FI "PID eq <pid>" /NH
   ```

2. Run the injector with the target:
   ```bash
   probe-inject.exe <target>
   ```
   The injector is located in the ARC Probe build output directory. If the path is not in PATH, use the full path.

3. Wait 2 seconds for the DLL to initialize and start the TCP server.

4. Verify the probe is connected:
   ```bash
   probe.exe "status"
   ```
   Or via MCP: `probe_status`. This should return process info including PID, module bases, and driver status.

5. If `probe_status` fails:
   - Check that the target process is still running
   - Check that port 9998 is not in use by another instance: `netstat -ano | findstr :9998`
   - The DLL may have failed to initialize -- check the probe log file in `%TEMP%`

6. Report the initial status to the user, including:
   - Target process name and PID
   - Number of loaded modules
   - Available memory regions
   - Probe version

## Error Handling

- If the process is not found, suggest the user start it first
- If injection fails with access denied, remind the user to run as Administrator
- If the TCP connection fails, the DLL may not have loaded -- suggest checking Event Viewer or probe logs
- If port 9998 is already in use, suggest setting PROBE_PORT to a different value

## What to do next

After successful injection, you can send commands via:

1. **CLI**: `probe.exe "<command>"` — returns JSON
2. **TCP**: connect to `127.0.0.1:9998`, send `command\n`
3. **HTTP Bridge**: `POST http://localhost:9996` (requires GUI running)

Good first commands:
```bash
probe.exe "status"           # Process info, PID, modules
probe.exe "modules list"     # All loaded DLLs with base addresses
probe.exe "ping"             # Health check
```

Then explore with:
- `dump <addr> <size>` — hex dump unknown memory
- `read_int <addr>` / `read_float <addr>` / `read_ptr <addr>` — read typed values
- `write_int <addr> <value>` / `write_float <addr> <value>` — modify values
- `disasm <addr> [count]` — disassemble code
- `rtti scan <module>` — discover C++ classes
- `pattern <bytes> [module]` — search for byte patterns
