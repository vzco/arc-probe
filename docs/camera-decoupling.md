# Camera Decoupling Research — Deadlock (Source 2)

> **Status:** POC validated — camera desync proven, CreateMove identified
> **Engine:** Source 2 (Deadlock, build ~2026-03)
> **Tools:** ARC Probe (injected DLL), Claude Bridge, source2gen SDK

---

## Table of Contents

- [Objective](#objective)
- [Background](#background)
- [Key Fields Discovered](#key-fields-discovered)
  - [Pawn Angle Fields](#pawn-angle-fields)
  - [Movement Services](#movement-services)
  - [Camera Services](#camera-services)
  - [Scene Node (Model Orientation)](#scene-node-model-orientation)
- [Live Readings](#live-readings)
- [Critical Finding: Model Follows Camera, Not Aim](#critical-finding-model-follows-camera-not-aim)
- [Camera Architecture (RTTI + String Analysis)](#camera-architecture-rtti--string-analysis)
  - [Camera Classes](#camera-classes)
  - [String References](#string-references)
- [Global Pointer Resolution](#global-pointer-resolution)
- [Approach Analysis](#approach-analysis)
  - [Option A: Write m_angClientCamera in a Loop](#option-a-write-m_angclientcamera-in-a-loop)
  - [Option B: Hook CCitadel_ThirdPersonCamera Vtable](#option-b-hook-ccitadel_thirdpersoncamera-vtable)
  - [Option C: SetClientCameraAngles Mechanism](#option-c-setclientcameraangles-mechanism)
  - [Option D: Hook CreateMove](#option-d-hook-createmove)
- [Observations and Gotchas](#observations-and-gotchas)
- [Next Steps](#next-steps)
- [Session 3 Findings (2026-03-08)](#session-3-findings-2026-03-08)
  - [Hardware Breakpoint Crash Fix](#hardware-breakpoint-crash-fix)
  - [Camera Object Confirmed](#camera-object-confirmed)
  - [Three Camera Yaw Writers](#three-camera-yaw-writers-clientdll)
  - [Pipeline Discovery](#pipeline-discovery)
  - [NOP Experiment Results](#nop-experiment-results)
  - [Code Cave Experiment](#code-cave-experiment-partial-success)
  - [Feedback Problem](#feedback-problem)
  - [Next Steps (Session 3)](#next-steps-1)
  - [Useful Addresses](#useful-addresses-this-session-clientdll-base-0x7ff87b9f0000)
- [Tools Used](#tools-used)

---

## Objective

Decouple the player's look/aim direction from the camera direction in Deadlock. The goal is to make the player model rotate independently from the camera — for example, the model spins 360 degrees one way while the camera goes the other way. This is a form of **view angle desync** where what the player sees through the camera differs from where the player model is facing.

In practical terms:

- The **camera** (what the player sees on screen) follows one set of angles.
- The **aim direction** (where shots land, server-authoritative) follows another.
- The **model orientation** (what other players see) follows yet another.

Controlling these three systems independently is the core challenge.

## Background

Source 2 separates "what direction the player is aiming" from "where the camera is looking" into distinct fields on the player pawn. This is a departure from Source 1, where `m_angRotation` on the player entity served double duty. In Deadlock specifically, the third-person camera adds another layer: the camera orbits the player model and has its own angle state (`m_angClientCamera`) that is distinct from the server-authoritative aim angles (`v_angle`).

Understanding which system controls what — and which one the model orientation follows — is the key to achieving desync.

---

## Key Fields Discovered

All offsets are from the source2gen SDK dump and verified against a live Deadlock process. Base is `client.dll`.

### Pawn Angle Fields

On `C_CitadelPlayerPawn` / `C_BasePlayerPawn`:

| Field | Offset | Type | Purpose |
|-------|--------|------|---------|
| `v_angle` | pawn + `0xF98` | QAngle (3 floats) | Player's aim direction. Server-authoritative, written every frame by the engine. |
| `v_anglePrevious` | pawn + `0xFA4` | QAngle (3 floats) | Previous frame's view angles. Used for interpolation/delta calculation. |
| `m_angEyeAngles` | pawn + `0x11B0` | QAngle (3 floats) | Eye angles broadcast to remote players. What enemies see as your aim direction. |
| `m_angClientCamera` | pawn + `0x1240` | QAngle (3 floats) | Client camera angles. Separate from aim — this controls the camera orbit. |
| `m_angLockedEyeAngles` | pawn + `0x14B0` | QAngle (3 floats) | Locked eye angles used during cutscenes or scripted events. |
| `m_pCameraServices` | pawn + `0xF18` | ptr | Pointer to `CPlayer_CameraServices` instance. |
| `m_pMovementServices` | pawn + `0xF20` | ptr | Pointer to `CPlayer_MovementServices` instance. |

### Movement Services

On `CCitadelPlayer_MovementServices` (accessed via pawn + `0xF20` dereference):

| Field | Offset | Type | Purpose |
|-------|--------|------|---------|
| `m_bHasFreeCursor` | +`0x2BE` | bool | Free cursor mode. The shop UI sets this to `true` — cursor moves independently from camera. |
| `m_flTurnSpringSpeed` | +`0x2C0` | float | How fast the model turns to match the camera direction. |
| `m_flInputDirectionCommitment` | +`0x2C4` | float | Input direction commitment factor. |
| `m_vecOldViewAngles` | +`0x228` | QAngle (3 floats) | Previous view angles stored by movement services. |

### Camera Services

On `CPlayer_CameraServices` (accessed via pawn + `0xF18` dereference):

| Field | Offset | Type | Purpose |
|-------|--------|------|---------|
| `m_vecPunchAngle` | +`0x48` | QAngle (3 floats) | Recoil punch angles. Added to the view during weapon fire. |
| `m_hViewEntity` | +`0x1B4` | CHandle (uint32) | Entity handle for the entity the camera follows. Normally the local player pawn. |
| `m_angDemoViewAngles` | +`0x310` | QAngle (3 floats) | View angles during demo playback or spectating. |

### Scene Node (Model Orientation)

On `CGameSceneNode` (accessed via pawn + `0x330` dereference):

| Field | Offset | Type | Purpose |
|-------|--------|------|---------|
| `m_angRotation` | +`0xB8` | QAngle (3 floats) | Model rotation in local space. |
| `m_angAbsRotation` | +`0xD4` | QAngle (3 floats) | Model rotation in world/absolute space. |
| `m_vecAbsOrigin` | +`0xC8` | Vector (3 floats) | World position of the entity. |

---

## Live Readings

All three angle systems were read simultaneously from a running Deadlock match using the ARC Probe debug server. The local player was standing still and looking in a consistent direction.

| Field | Pitch | Yaw | Roll |
|-------|-------|-----|------|
| `v_angle` (aim) | 36.49 | 126.34 | 0.0 |
| `m_angClientCamera` | 36.39 | 128.67 | 0.0 |
| scene node `m_angRotation` | 0.0 | 128.78 | — |
| scene node `m_angAbsRotation` | 0.0 | 128.78 | — |

---

## Critical Finding: Model Follows Camera, Not Aim

The live readings above reveal the fundamental relationship:

```
Model yaw:          128.78°  ← tracks this
m_angClientCamera:  128.67°  ← closest match
v_angle (aim):      126.34°  ← does NOT match model
```

The model orientation (`m_angAbsRotation` at 128.78 degrees yaw) tracks `m_angClientCamera` (128.67 degrees), **not** `v_angle` (126.34 degrees). The ~2 degree offset between `v_angle` and `m_angClientCamera` is significant — it proves these are genuinely independent systems.

This means:

1. **`v_angle`** is the server-authoritative aim direction. Where shots actually land.
2. **`m_angClientCamera`** controls the visual camera orbit AND the model's facing direction.
3. **`m_angRotation` / `m_angAbsRotation`** on the scene node reflects the model's visual orientation, which follows the camera.

To achieve desync, you need to control `m_angClientCamera` independently from `v_angle`. The model will follow `m_angClientCamera`, while shots still go where `v_angle` points.

Additionally, the scene node pitch is always 0.0 degrees regardless of where the player looks up or down. Deadlock models do not visually pitch — only yaw rotation is reflected in the model. Pitch is expressed only through animations (upper body lean).

---

## Camera Architecture (RTTI + String Analysis)

### Camera Classes

RTTI scanning of `client.dll` revealed the following camera-related class hierarchy:

| Class | Vtable Entries | Purpose |
|-------|---------------|---------|
| `CCitadel_ThirdPersonCamera` | 50 | Main third-person camera implementation. This is the active camera during gameplay. |
| `CCitadelCameraController` | 7 | Camera controller singleton registered as a game system. Manages camera transitions. |
| `CCitadelUserMsg_SetClientCameraAngles` | — | Protobuf network message used to force client camera angles from the server. |
| `CCitadel_Modifier_TurnCameraToTarget` | — | Game modifier that forces the camera to rotate toward a target (used by abilities). |
| `CCitadel_Modifier_NonPlayerCamera` | — | Modifier for non-player camera modes (death cam, spectator). |
| `CPlayer_CameraServices` | — | Base camera services component on the player pawn. |
| `CCitadelPlayer_CameraServices` | — | Deadlock-specific extension of camera services. |

Other relevant classes found:

| Class | Purpose |
|-------|---------|
| `ICamera` | Camera interface base class. |
| `CSimpleCamera` | Simple camera implementation (likely used for debug/editor). |
| `CDataDrivenCamera` | Data-driven camera with configurable parameters. |
| `CAimCameraUpdateNode` | Animation graph node that updates the aim camera. |
| `CAimCameraOperation` | Animation graph operation for aim camera logic. |

The `CCitadel_ThirdPersonCamera` with 50 vtable entries is the primary target for hooking. It likely contains the equivalent of `CalcView` from Source 1 — the function that computes the final camera position and angles each frame.

### String References

Key string references found in `client.dll` via the probe's string scanner:

| String | Address | Significance |
|--------|---------|--------------|
| `"CreateMoveSubTick"` | `client.dll + ...` (0x7FF882C0C015) | CreateMove pipeline — the standard Source engine hook point for input manipulation. |
| `"CreateMoveFirst"` | `client.dll + ...` (0x7FF882C0C057) | Early CreateMove call, before input is processed. |
| `"m_angClientCamera"` | 21 xrefs in `client.dll` | The client camera angle field is actively used across 21 different code paths. High usage suggests it is a core field, not vestigial. |

The 21 cross-references to `"m_angClientCamera"` indicate this field is read and written by many systems — rendering, input, networking, animation. Any override must account for the engine potentially snapping it back.

---

## Global Pointer Resolution

These global pointers were resolved via pattern scanning against a live Deadlock process. **These offsets change with every game update** — re-scan after patches.

| Name | Pattern | Resolved Address | Notes |
|------|---------|-----------------|-------|
| `dwGameEntitySystem` | `48 8B 1D ?? ?? ?? ?? 48 89 1D ?? ?? ?? ?? 4C 63 B3` | `0x7FF883A15EB8` | Entity system instance pointer. Base for entity list traversal. |
| `dwPrediction` | `48 8D 05 ?? ?? ?? ?? C3 CC CC CC CC CC CC CC CC 40 53 56 41 54` | `0x7FF883030DA0` | Prediction object. Contains local player reference. |
| `dwLocalPlayerPawn` | Derived: `dwPrediction + 0xD8` | `0x7FF883030E78` | Local player pawn pointer. Read via `read_chain`. |

Entity list traversal from `dwGameEntitySystem`:

```
entity_system = read_ptr(dwGameEntitySystem)
chunk_ptr     = read_ptr(entity_system + 0x10)   // first chunk
identity      = read_ptr(chunk_ptr) + (index * 0x70)  // CEntityIdentity
entity        = read_ptr(identity + 0x0)          // actual entity pointer
```

---

## Approach Analysis

Four approaches have been identified for achieving camera decoupling. They are listed from simplest to most robust.

### Option A: Write m_angClientCamera in a Loop

**Complexity:** Low | **Robustness:** Unknown | **Risk:** May be overwritten every frame

Continuously override `m_angClientCamera.yaw` with a custom value (spinning, offset, or fixed) while leaving `v_angle` under normal player control.

```
// Pseudocode
while (active) {
    pawn = read_ptr(dwLocalPlayerPawn)
    current_v_angle = read_float3(pawn + 0xF98)    // aim stays normal
    write_float(pawn + 0x1240 + 4, custom_yaw)     // override camera yaw only
    sleep(1ms)
}
```

**Pros:**
- Simplest to implement — just memory writes in a loop.
- No hooking required.
- `v_angle` (aim) remains under player control.

**Cons:**
- The engine writes `m_angClientCamera` every frame. The override may flicker or be invisible if the engine writes after our write.
- 21 xrefs to this field suggest many systems touch it — fighting the engine.
- No control over write ordering relative to the render frame.

**Verdict:** Good for a quick proof-of-concept test. If the camera visually moves even briefly, it confirms the field controls what we think it does.

### Option B: Hook CCitadel_ThirdPersonCamera Vtable

**Complexity:** Medium-High | **Robustness:** High | **Risk:** Must identify correct vtable index

The `CCitadel_ThirdPersonCamera` class has 50 vtable entries. One of these is the camera update function (equivalent to `CalcView` in Source 1). Hooking it gives control over the camera angles at the right point in the frame pipeline.

```
// Approach
1. rtti vtable CCitadel_ThirdPersonCamera   // get vtable address
2. Disassemble each vtable entry            // find camera update function
3. Look for writes to m_angClientCamera      // identify the right function
4. Hook via MinHook (available in probe)     // intercept and modify angles
```

**Pros:**
- Operates at the correct point in the rendering pipeline.
- No race condition with engine writes.
- Full control over camera angle calculation.
- Can inject arbitrary logic (spinning, tracking, smoothing).

**Cons:**
- Must reverse-engineer 50 vtable entries to find the right one.
- Vtable offsets change between game updates.
- Hook must preserve calling convention and not break the camera system.

**Verdict:** The most robust approach for a persistent feature. Requires upfront RE work to identify the camera update function.

### Option C: SetClientCameraAngles Mechanism

**Complexity:** Medium | **Robustness:** High | **Risk:** May require finding dispatch function

`CCitadelUserMsg_SetClientCameraAngles` is a built-in protobuf network message that the server uses to force the client's camera angles (e.g., during abilities, cutscenes). If we can find and call the handler function directly, we get an engine-native camera override that the game respects properly.

```
// Approach
1. rtti find SetClientCameraAngles          // find the class
2. strings find "SetClientCameraAngles"      // find string references
3. Follow xrefs to find the dispatch handler // lambda or callback
4. Call the handler with custom angles       // engine-native override
```

**Pros:**
- Uses the engine's own mechanism for overriding camera angles.
- The game is designed to handle this — no fighting the engine.
- Clean integration with the rest of the camera system.

**Cons:**
- Must find the dispatch function (likely a lambda registered during initialization).
- The message may have side effects (UI state, ability interruption).
- May only work for one-shot overrides, not continuous control.

**Verdict:** Cleanest approach if the dispatch handler can be found and called repeatedly. Worth investigating the string xrefs.

### Option D: Hook CreateMove

**Complexity:** Medium | **Robustness:** High | **Risk:** Classic, well-understood approach

`CreateMove` is the standard Source engine function where client input is assembled into a command before being sent to the server. In Source 2, the strings `"CreateMoveSubTick"` and `"CreateMoveFirst"` mark the relevant code paths.

```
// Approach
1. strings xref <CreateMoveSubTick_addr>    // find referencing code
2. Disassemble the CreateMove function       // understand the command structure
3. Hook CreateMove via MinHook               // intercept input processing
4. In the hook:
   - Modify view angles in the command       // server receives modified angles
   - Keep camera rendering angles separate   // player sees normal view
```

**Pros:**
- This is the traditional and well-understood approach from Source 1 cheats.
- CreateMove is the canonical hook point for input manipulation.
- Full control over what angles the server receives vs. what the camera shows.
- Proven technique for anti-aim/desync in Source engines.

**Cons:**
- Source 2's CreateMove may have a different structure than Source 1.
- VAC/anti-cheat may monitor CreateMove hooks specifically.
- Must understand the `CUserCmd` structure in Source 2 (different from Source 1).

**Verdict:** The most proven approach from a Source engine perspective. If CreateMove can be located and hooked, this gives the most direct control over the aim/camera split.

---

## Observations and Gotchas

1. **`v_angle` is written every frame.** Setting a hardware write breakpoint on it will fire thousands of times per second, effectively crashing the game from the sheer volume of breakpoint hits. Use read breakpoints or pattern analysis instead.

2. **`m_angClientCamera` values are quantized.** Observed values like `36.386719` suggest network quantization (likely 16-bit angles mapped to 0-360). This means the field may be networked and subject to server reconciliation.

3. **`m_flTurnSpringSpeed` was nearly zero.** The observed value was `6e-6` (0.000006), suggesting the model turn rate is effectively instant — the model snaps to the target orientation rather than smoothly interpolating. This could simplify desync since there is no visible "catching up" animation.

4. **Scene node pitch is always 0.0.** Deadlock models do not visually tilt up or down. Only yaw is reflected in `m_angRotation` / `m_angAbsRotation`. Pitch is expressed purely through upper-body animations. This means pitch desync would be invisible at the model level.

5. **`dwLocalPlayerController` pattern does not match in Deadlock.** The CS2-style pattern for finding the local player controller does not work. Instead, resolve the local player by reading `m_hController` (pawn + `0x10A8`) on the pawn obtained from `dwPrediction + 0xD8`.

6. **Probe server is single-client.** The TCP server on port 9998 accepts only one connection at a time. When the GUI is connected, use the Claude Bridge (HTTP on port 9996) to route commands through the GUI's existing TCP connection.

7. **21 xrefs to m_angClientCamera.** This field is heavily used across the codebase. Any write-based approach must contend with the engine's own writes. A hook-based approach that intercepts the write at the source is more reliable than fighting the engine with post-hoc overwrites.

---

## Session 2 Findings (2026-03-08)

### Hardware Breakpoint on m_angClientCamera

Set a hardware write breakpoint on `m_angClientCamera.yaw`. Results:
- **295 hits**, all from the same function at `networksystem.dll + 0xE9B90` (RVA)
- Hits came from **15+ different thread IDs** (Source 2 threadpool)
- R10 always = networksystem.dll base (position-independent code pattern)
- RDX always = pawn+0x1240 (&m_angClientCamera, constant across all hits)
- R14 always = `0x5384136C300` (constant object, likely field descriptor)

### The Field Setter (networksystem.dll + 0xE9B90)

Two adjacent functions — both are **protobuf field setters** for QAngle fields:

```asm
;; Function A (0xE9B60) — with type validation
cmp dword ptr [r8+0x2C], 0x03    ; check field type == 3 (QAngle)
jnz fail
cmp dword ptr [r8+0x28], 0x03    ; check element count == 3
jnz fail
mov rcx, [rcx+0x10]              ; rcx = destination ptr (m_angClientCamera)
movsd xmm0, [r8]                 ; load pitch+yaw (8 bytes) from source
movsd [rcx], xmm0                ; write pitch+yaw
mov eax, [r8+0x08]               ; load roll
mov [rcx+0x08], eax              ; write roll
mov eax, 0x01                    ; return true
ret

;; Function B (0xE9B90) — no validation (the one that fires)
mov rdx, [rcx+0x10]              ; rdx = destination ptr
movsd xmm0, [r8]                 ; load pitch+yaw
movsd [rdx], xmm0                ; WRITE pitch+yaw ← hwbp fires HERE
mov eax, [r8+0x08]               ; load roll
mov [rdx+0x08], eax              ; write roll
mov eax, 0x01
ret
```

These are generic serialization-layer functions. `[rcx+0x10]` is a field descriptor that points to the target field's address. The fact that m_angClientCamera is written through this system means it's a **networked/replicated property**.

### POC: Camera Desync (VALIDATED)

Successfully wrote `m_angClientCamera.yaw = 90.0` while `v_angle.yaw = -173.87`:
- **264° desync** between camera and aim
- **Persisted for 2+ seconds** — sampled 3 times at 1-second intervals, value stayed at 90.0
- The field setter does NOT write every frame — only on server property updates (occasional)
- Crash occurred on `session restore` (race condition with the field setter writing simultaneously)

### CCitadelInput Class (CreateMove Owner)

RTTI scan found `CCitadelInput` in client.dll. Vtable at `client.dll + 0x22AB318` with 20 entries:

| Index | Address (RVA) | Notes |
|-------|--------------|-------|
| 0 | 0x5057E0 | Small function |
| 1 | 0x15D3610 | Large function |
| 2 | 0x15D9940 | |
| 3 | 0x15D47F0 | Large function |
| 4 | 0x5055A0 | |
| **5** | **0x15CC790** | **CreateMove (confirmed)** |
| 6 | 0x504860 | |
| 7 | 0x15D9950 | |
| 8 | 0x15CF510 | |
| 9 | 0x15D4C60 | |
| 10-15 | 0x504860-0x5048C0 | Small stubs (likely no-ops or simple getters) |
| 16 | 0x15DA300 | |
| 17 | 0x5055B0 | |
| 18 | 0x15CA440 | |
| 19 | 0x505420 | |

**Verification:** With arc-logic.dll injected, vtable[5] showed a MinHook double-trampoline (`JMP thunk → JMP [rip+0] → detour`). Without arc-logic, the original bytes are `85 D2` (`test edx, edx`) — a clean slot parameter check. This confirms **vtable[5] = CreateMove**.

### CreateMove Full Function Map (client.dll + 0x15CC790)

**Signature:** `void CCitadelInput::CreateMove(int slot, bool bActive)`
**Size:** 4237 bytes (0x108D) | **~900 instructions** | **28+ function calls**
**Entry:** `client.dll + 0x15CC790` | **Return:** `client.dll + 0x15CD81D`

```
CreateMove (client.dll + 0x15CC790 → 0x15CD81D)
│
├─ ENTRY (+0x00):
│   test edx, edx                    ; check slot parameter
│   jnz epilogue                     ; slot != 0 → skip (early out)
│
├─ PROLOGUE (+0x08):
│   mov rax, rsp
│   [store params: rcx=this, edx=slot, r8b=bActive]
│   push rbp / rbx / r12 / r15
│   sub rsp, 0x1D8                   ; 472 bytes of stack frame
│
├─ PHASE 1 — Init (+0x08 to +0x49):
│   r12 = CCitadelInput* (this)
│   call 0x...AC090                  ; get_local_player() → rdi = player state
│   test rax, rax / jz epilogue      ; bail if no player (null → early exit to +0x1078)
│   lea r13, [r12+0x228]             ; r13 = CCitadelInput command buffer base
│   call 0x...F0730                  ; get_tick_info()
│
├─ PHASE 2 — Tick Setup (+0x49 to +0x2C1):
│   r14d = [rax+0x6270]              ; read tick counter from player state
│   inc r14d                         ; advance tick
│   mov [rsi+0x6270], r14d           ; write new tick counter
│   call [vtable_ptr]                ; dispatch (profiling? timing?)
│   call 0x...93BA0                  ; more tick processing
│   References "CreateMoveSubTick" and "CreateMoveFirst" strings
│   Builds profiling/trace entries for sub-tick system
│
├─ PHASE 3 — Sub-tick Processing Loop (+0x2C1 to +0x533):
│   loop_count = [r13+0x5C]          ; number of sub-ticks
│   rdi = r13 + 0x70                 ; first sub-tick entry (source angles)
│
│   FOR each sub-tick:
│     Allocate CUserCmd from pool → rcx
│
│     ★ CUserCmd ANGLE WRITES:
│       [rcx+0x10] = flags           ; bit 1=has_input, bit 2=has_angle, bit 4=has_time
│       [rcx+0x18] = ptr             ; associated entity/controller?
│       [rcx+0x20] = byte            ; sub-tick index or type
│       [rcx+0x24] = [rdi-0x10]      ; sub-tick timestamp (float)
│       [rcx+0x28] = [rdi+0x00]      ; ★ pitch (float)
│       [rcx+0x2C] = [rdi+0x04]      ; ★ yaw (float)
│       [rcx+0x30] = [rdi+0x08]      ; ★ roll or forward move (float)
│       [rcx+0x34] = [rdi+0x0C]      ; ★ side move or extra (float)
│
│     Advance rdi to next sub-tick entry
│
├─ PHASE 4 — Post-processing (+0x533 to +0x1063):
│   call 0x...A2570                  ; CUserCmd dispatch (reads [rdx+0x20] flags)
│   call [rax+0x30]                  ; vtable: apply command to player state
│   call 0x...F11D0                  ; prediction / reconciliation
│   call [rax+0xB0]                  ; vtable: movement processing
│   call [rax+0xAA0]                 ; vtable: ability/weapon processing
│   call 0x...F9AF10                 ; get instance (returns global ptr)
│   call 0x...9FB260                 ; post-frame processing
│   ... more vtable calls through [rax+0xB8], [rax+0x18] ...
│   call 0x...EA040                  ; late-frame processing
│   call [vtable+0x18] x4            ; sequence of vtable dispatches
│   call 0x...48250                  ; ★ LAST functional call (finalize)
│
├─ EPILOGUE (+0x1068):
│   mov r13, [rsp+0x1C0]            ; restore nonvolatile regs
│   mov rsi, [rsp+0x1D0]
│   mov rdi, [rsp+0x1C8]
│   add rsp, 0x1D8                  ; deallocate stack
│   pop r15 / r12 / rbx / rbp
│   ret
│
└─ int3 padding (next function starts at +0x108E)
```

### CUserCmd Partial Layout (Source 2 / Deadlock)

Derived from CreateMove's angle writes:

| Offset | Type | Field | Notes |
|--------|------|-------|-------|
| 0x10 | uint32 | flags | Bit 1=has_input, bit 2=has_angle, bit 4=has_time |
| 0x18 | ptr | associated_entity | Controller or pawn handle? |
| 0x20 | byte | sub_tick_type | Sub-tick index or command type |
| 0x24 | float | tick_time | Sub-tick timestamp |
| 0x28 | float | pitch | View angle pitch |
| 0x2C | float | yaw | View angle yaw |
| 0x30 | float | unknown_1 | Roll or forward_move |
| 0x34 | float | unknown_2 | Side_move or extra data |

The source data comes from `CCitadelInput + 0x298` (`r13+0x70` where `r13 = this+0x228`). This is the accumulated input buffer containing mouse-derived view angles and sub-tick timing data.

### Hook Points for Camera Decoupling

Three candidate hook points identified, ordered by recommendation:

| Priority | RVA | Address | Location | Description |
|----------|-----|---------|----------|-------------|
| **1 (Best)** | `+0x00` | `0x15CC790` | Function entry | MinHook detour: call original → post-process angles |
| 2 | `+0x25B` | `0x15CCFEB` | CUserCmd angle write | Inline patch: modify yaw before `movss [rcx+0x2C], xmm0` |
| 3 | `+0x1068` | `0x15CD7F8` | After last call | Inline patch: insert code between finalize and epilogue |

**Recommended: Hook Point #1 (function entry)**

The function entry is the standard MinHook target. The detour pattern:
1. The first 2 bytes (`85 D2` = `test edx, edx`) get replaced with a JMP to the detour
2. MinHook creates a trampoline with the displaced bytes + JMP back
3. Our detour calls the trampoline (executing the full original CreateMove)
4. After it returns, we read `v_angle`, apply offset, write `m_angClientCamera`
5. Return

This is clean, reversible, and doesn't touch any code inside the function body.

### Probe Hook Infrastructure

The probe DLL has a complete MinHook wrapper (`src/core/hook-manager.hpp`) with:
- Singleton `HookManager::instance()` with init/shutdown lifecycle
- Fluent API: `add(name, target, detour, &original).install_all()`
- Per-hook enable/disable
- Thread-safe `HookGuard` RAII pattern for safe shutdown
- `begin_shutdown()` waits for active hooks to drain (max 5s)

**Currently unused** — no TCP command exposes it. Needs `src/commands/cmd-camera.cpp` to wire it up.

### SetClientCameraAngles in client.dll

Found the full protobuf handler mangled name:
```
?...@@EEAAXAEBVCCitadelUserMsg_SetClientCameraAngles_t@@@Z
```

Multiple string instances in client.dll:
- `0x7FF87DFCC2BB` — "SetClientCameraAngles" (plain)
- `0x7FF87DFF1F22` — "SetClientCameraAngles" (registration)
- `0x7FF87E0350B8` — "SetClientCameraAngles_t" (type name)
- `0x7FF87E8E628A` — full mangled handler signature

---

## Next Steps

### Immediate (once probe reconnects)

1. **Re-find CreateMove with fresh addresses.** Module bases change each session. Pattern scan for `"CreateMoveSubTick"` string reference to locate CreateMove.

2. **Deep-dive CreateMove's view angle processing.** Disassemble the full function (it's large — likely 2000+ bytes). Look for:
   - Where it reads/writes `v_angle` (pawn+0xF98)
   - Where it reads/writes `m_angClientCamera` (pawn+0x1240)
   - Where it calls into the sub-tick input processing
   - The `CUserCmd` structure layout

3. **Find the post-CreateMove camera update.** The ideal hook point is AFTER CreateMove sets v_angle but BEFORE the camera system reads it. This is where we insert the angle offset.

### Implementation Approaches

**Approach 1: Hook CreateMove end (add detour code)**
- Hook the tail of CreateMove (before RET)
- Detour reads current v_angle, applies yaw offset, writes to m_angClientCamera
- Uses probe's MinHook wrapper (HookManager exists but needs TCP command exposure)

**Approach 2: Code cave via write + call**
- `call VirtualAlloc` to allocate RWX memory for shellcode
- Write angle-manipulation shellcode to the cave
- Patch a CALL or JMP in CreateMove to redirect to the cave
- Cave applies offset, then returns to original code

**Approach 3: NOP the field setter + continuous write**
- NOP out the field setter at networksystem.dll+0xE9B90 (prevents server overwrites)
- Continuously write m_angClientCamera via TCP from a script
- Simplest but highest latency

### Feature Design

Once the hook is in place, the camera desync feature would support:
- **Offset mode:** Camera yaw = v_angle.yaw + configurable_offset
- **Spin mode:** m_angClientCamera.yaw increments each frame (visual 360° spin)
- **Reverse mode:** Camera faces opposite direction from aim (180° offset)
- **Lock mode:** Camera stays at a fixed yaw while aim follows mouse normally

---

## Implementation: cmd-camera.cpp

### Architecture
- New file: `src/commands/cmd-camera.cpp`
- Uses HookManager (src/core/hook-manager.hpp) to inline-hook CreateMove via MinHook
- Hook point: CreateMove entry at `client.dll + 0x15CC790` (CCitadelInput vtable[5])
- Detour calls original CreateMove, then post-processes angles

### Detour Logic
```
1. Call original CreateMove (trampoline)
2. Resolve local pawn: pattern scan dwPrediction → read ptr at +0xD8
3. Read v_angle.yaw (pawn + 0xF9C) and v_angle.pitch (pawn + 0xF98)
4. Compute camera angle based on mode:
   - Offset mode: camera_yaw = v_angle.yaw + offset_degrees
   - Spin mode: camera_yaw += spin_speed * delta_time (continuous rotation)
   - Reverse mode: camera_yaw = v_angle.yaw + 180.0
5. Clamp yaw to [-180, 180] range
6. Write m_angClientCamera (pawn + 0x1240): pitch from v_angle, yaw from step 4
7. Return
```

### TCP Commands
| Command | Description |
|---------|-------------|
| `camera desync <degrees>` | Set fixed yaw offset between aim and camera |
| `camera spin <degrees_per_sec>` | Continuous camera rotation at given speed |
| `camera reverse` | 180° offset (look behind while aiming forward) |
| `camera off` | Disable and restore normal camera |
| `camera status` | Show current mode, offset, hook state |

### Key Addresses (session-dependent, must pattern-scan each inject)
- CreateMove: Find via CCitadelInput RTTI → vtable[5]
- dwPrediction: Pattern `48 8D 05 ?? ?? ?? ?? C3 CC CC CC CC CC CC CC CC 40 53 56 41 54` with --resolve 3
- Local pawn: dwPrediction + 0xD8

### Thread Safety
- HookGuard RAII pattern from hook-manager.hpp
- Atomic bool for enable/disable toggle
- All state in static globals within cmd-camera.cpp

---

## Session 3 Findings (2026-03-08)

### Hardware Breakpoint Crash Fix
- The VEH handler used `EnterCriticalSection(&g_cs)` on every hardware BP hit (~180 hits/sec on hot camera paths)
- This caused deadlock: game thread stalls on g_cs while TCP thread holds it during `bp log`
- Fix: Lock-free fast-path for hardware BPs with per-hw-slot atomic counters
- Added `max_hits` auto-disable (default 512) — BP auto-disables after N hits by clearing DR7 in exception context
- New command syntax: `hwbp set <addr> [r|w|x] [1|2|4|8] [--max N]`
- Files modified: breakpoints.cpp, breakpoints.hpp, cmd-debugger.cpp

### Camera Object Confirmed
- CCitadel_ThirdPersonCamera global pointer: `client.dll + 0x31F3718`
- Camera object is heap-allocated and pointer changes (must re-read each session)
- Camera pitch at object+0x44, yaw at +0x48, roll at +0x4C (CBaseCamera layout)
- Camera position at +0x38, +0x3C, +0x40
- Second copy at +0xC8-0xDC

### Three Camera Yaw Writers (client.dll)
All three write to camera_object+0x48 (`movss [rdi+0x48], xmmN`) every frame:

1. **Writer 1** (RVA 0x57871A area, instruction at RVA 0x578715):
   - `addss xmm0, [rdi+0x44]` → pitch += delta
   - `movss [rdi+0x44], xmm0` → write pitch
   - `addss xmm1, [rdi+0x48]` → yaw += delta
   - `movss [rdi+0x48], xmm1` → write yaw (5 bytes: F3 0F 11 4F 48)
   - This is the primary mouse delta application to camera angles

2. **Writer 2** (RVA 0x57BD90):
   - `movss [rdi+0x44], xmm10` → write pitch
   - `addss xmm8, [rdi+0x48]` → yaw += delta
   - `movss [rdi+0x48], xmm8` → write yaw (6 bytes: F3 44 0F 11 47 48)
   - Near function epilogue (ret at RVA 0x57BD9F), different code path

3. **Writer 3** (RVA 0x578091):
   - `movss xmm0, [rdi+0x48]` → read current yaw
   - `call normalize` → angle normalization
   - `movss [rdi+0x48], xmm0` → write normalized yaw (5 bytes: F3 0F 11 47 48)
   - Reads camera+0x48, normalizes, writes back

### Pipeline Discovery
The pipeline is NOT what was initially assumed. Actual flow:

```
Mouse Input
    ↓
Camera Update Functions (writers 1/2/3) write to camera+0x48
    ↓
camera+0x48 FEEDS BACK into next frame's delta computation
    ↓
CCitadelInput+0x68C (accumulated yaw) — close but NOT identical to camera yaw
    ↓
CreateMove reads CCitadelInput → packs CUserCmd
    ↓
Server sets v_angle from CUserCmd
```

Key observation: camera yaw ≠ CCitadelInput yaw (off by ~6 degrees, likely smoothing/interpolation). But they're tightly coupled — freezing camera+0x48 also freezes CCitadelInput and v_angle.

### NOP Experiment Results
- NOPing all 3 yaw write sites: camera yaw frozen, mouse horizontal input has NO effect on anything (camera, body, aim all frozen horizontally). Pitch still works (not NOPed).
- Writing custom yaw to camera+0x48 with NOPs: persists, visually moves camera AND body together.
- The entire system is coupled through camera+0x48 as the central accumulator.

### Code Cave Experiment (Partial Success)
Created a code cave to override camera+0x48 after the real write:
- Cave at 0x7FF7FBF70000, control block at 0x7FF7FBF80000
- control+0: desync_enabled (int32), control+4: custom_yaw (float32)
- Patched writer 1 with JMP to cave
- Cave executes original write, then overrides camera+0x48 with custom_yaw
- **Result**: Camera yaw = custom value, CCitadelInput/v_angle = different value (DECOUPLED in data!)
- **BUT**: Body model visually followed camera, not v_angle. In third-person, the model renders facing camera direction.
- **AND**: The override feeds back — next frame's `addss xmm1, [rdi+0x48]` reads the override, corrupting the real yaw accumulation. Mouse input to CCitadelInput/v_angle stops flowing.

### Feedback Problem
The fundamental issue: `addss xmm1, [rdi+0x48]` in writer 1 reads camera+0x48 to compute the next yaw. Overriding camera+0x48 corrupts this computation. Need either:
1. Feedback correction in the cave (subtract override, add real yaw before the addss)
2. Override only at render time (after all pipeline processing)
3. Hook CreateMove to override CUserCmd angles independently

### Next Steps
- Check internal-v2 psilent feature for the correct decoupling approach
- The psilent technique likely hooks CreateMove to override CUserCmd angles while leaving camera alone
- Consider: in third-person games, model facing = camera direction. True visual decoupling may require overriding the model's render angles, not just v_angle.

### Useful Addresses (this session, client.dll base 0x7FF87B9F0000)
| Item | Address | Notes |
|------|---------|-------|
| client.dll base | 0x7FF87B9F0000 | Changes each launch |
| Camera global ptr | client.dll + 0x31F3718 | Pointer to CCitadel_ThirdPersonCamera |
| CCitadelInput ptr | client.dll + 0x2D1D770 | Pointer to input singleton |
| CCitadelInput yaw | singleton + 0x68C | Accumulated mouse yaw |
| Writer 1 (mouse delta) | client.dll + 0x578715 | movss [rdi+0x48], xmm1 (5 bytes) |
| Writer 2 (path 2) | client.dll + 0x57BD90 | movss [rdi+0x48], xmm8 (6 bytes) |
| Writer 3 (normalize) | client.dll + 0x578091 | movss [rdi+0x48], xmm0 (5 bytes) |
| Code cave | 0x7FF7FBF70000 | Allocated, rel32 reachable |
| Control block | 0x7FF7FBF80000 | +0=enabled, +4=custom_yaw, +8=saved_real_yaw |

---

## Tools Used

| Tool | Purpose |
|------|---------|
| **ARC Probe** (`probe-shell.dll`) | Injected into Deadlock via `probe-inject.exe`. Provides TCP command interface for memory read/write, disassembly, RTTI scanning, string scanning, breakpoints. |
| **Claude Bridge** (HTTP on `:9996`) | Routes probe commands through the Tauri GUI's existing TCP connection. Used when the GUI is connected and occupying the single-client TCP slot. |
| **Pattern scanning** (`pattern` command) | Resolved fresh global pointers (`dwGameEntitySystem`, `dwPrediction`) against the live binary. |
| **RTTI scanning** (`rtti scan`, `rtti find`) | Discovered the camera class hierarchy: `CCitadel_ThirdPersonCamera`, `CCitadelCameraController`, modifiers. |
| **String scanning + xref analysis** (`strings find`, `strings xref`) | Found `"CreateMoveSubTick"`, `"CreateMoveFirst"`, and the 21 xrefs to `"m_angClientCamera"`. |
| **source2gen SDK dump** | Provided exact field offsets for `C_BasePlayerPawn`, `CPlayer_CameraServices`, `CCitadelPlayer_MovementServices`, `CGameSceneNode`, and all other classes referenced in this document. |
| **Hardware breakpoints** (`hwbp set`) | Used to find what writes to angle fields. Revealed that `v_angle` is written every frame (too frequently for breakpoint-based analysis). |
