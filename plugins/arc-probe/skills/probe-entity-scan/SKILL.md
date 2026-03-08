---
name: probe-entity-scan
description: Scan and resolve game entities from a running Deadlock process via ARC Probe. Finds local player, resolves entity list, reads player data, and builds structs in the GUI. Use when the user wants to inspect live game entities.
argument-hint: [entity_type]
---

# ARC Probe — Deadlock Entity Scanner

Scan live Deadlock entities via the probe bridge and display them in the ARC Probe GUI.

## CRITICAL: Do NOT Assume Offsets Are Current

- Game offsets change with EVERY patch. Never assume hardcoded offsets from CLAUDE.md, memory files, or training data are correct.
- ALWAYS verify offsets against the latest schema dump before using them. The user may provide a fresh dump location.
- If an entity list pointer returns NULL or a read returns suspicious values (e.g., pointer-like values where health should be), the offsets are likely stale — do NOT continue blindly.
- Ask the user for the current schema dump location or use pattern scanning to re-derive offsets dynamically.
- Never go down a rabbit hole with old data. Check first, then proceed.

## Prerequisites

- ARC Probe GUI running with bridge on `:9996`
- probe-shell.dll injected into `deadlock.exe`
- Player must be in a match (not menu/lobby) for entity data

## Key Offsets

These offsets are from the schema dump and may change after game patches. Check `dezlock-dump` output for fresh values.

### Global Pointers (added to client.dll base)

| Name | Offset | Description |
|------|--------|-------------|
| `dwLocalPlayerPawn` | `0x2E76FE8` | Pointer to local player pawn |
| `dwEntityList` | `0x3862C28` | CGameEntitySystem pointer |
| `dwViewMatrix` | `0x3745490` | View matrix (4x4 float) |

### C_BaseEntity Field Offsets

| Field | Hex | Decimal | Type |
|-------|-----|---------|------|
| m_pGameSceneNode | 0x330 | 816 | pointer |
| m_iMaxHealth | 0x350 | 848 | int32 |
| m_iHealth | 0x354 | 852 | int32 |
| m_lifeState | 0x35C | 860 | int32 |
| m_flSimulationTime | 0x3C0 | 960 | float |
| m_iTeamNum | 0x3F3 | 1011 | uint8 |

### CBasePlayerController Field Offsets

| Field | Hex | Decimal | Type |
|-------|-----|---------|------|
| m_hPawn | 0x6BC | 1724 | uint32 |
| m_iszPlayerName | 0x6F0 | 1776 | pointer (string) |
| m_steamID | 0x778 | 1912 | uint64 |

### CGameSceneNode Field Offsets

| Field | Hex | Decimal | Type |
|-------|-----|---------|------|
| m_vecOrigin | 0x80 | 128 | float[3] (Vec3) |
| m_vecAbsOrigin | 0xC8 | 200 | float[3] (Vec3) |
| m_bDormant | 0x103 | 259 | bool |

## Steps

### 1. Find client.dll base

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"modules list"}' \
  | python -c "import sys,json; mods=json.load(sys.stdin)['data']['data']['modules']; [print(m['name'],m['base']) for m in mods if 'client' in m['name'].lower()]"
```

Store the base address (e.g., `0x7FFB0A8D0000`) for offset calculations.

### 2. Read local player pawn

```bash
CLIENT_BASE="0x7FFB0A8D0000"  # from step 1
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d "{\"action\":\"probe\",\"command\":\"read_ptr ${CLIENT_BASE}+0x2E76FE8\"}"
```

The `value` field in the response is the pawn address (e.g., `0x2E6C0389400`).

### 3. Verify the pawn is valid

```bash
PAWN="0x2E6C0389400"  # from step 2

# Read health + max health (8 bytes starting at 0x350)
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d "{\"action\":\"probe\",\"command\":\"read ${PAWN}+0x350 8\"}" \
  | python -c "
import sys,json,struct
d=json.load(sys.stdin)['data']['data']
raw=bytes.fromhex(d['hex'][:16])
maxhp=struct.unpack_from('<i',raw,0)[0]
hp=struct.unpack_from('<i',raw,4)[0]
print(f'Health: {hp}/{maxhp}')
"

# Read team
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d "{\"action\":\"probe\",\"command\":\"read ${PAWN}+0x3F3 1\"}" \
  | python -c "
import sys,json
d=json.load(sys.stdin)['data']['data']
team=int(d['hex'][:2],16)
teams={0:'World',1:'Spectator',2:'Amber',3:'Sapphire',4:'Neutral'}
print(f'Team: {teams.get(team,team)}')
"
```

### 4. Build struct in GUI

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d "{
  \"action\":\"batch\",\"actions\":[
    {\"action\":\"activity\",\"status\":\"working\",\"message\":\"Building local player struct...\"},
    {\"action\":\"store\",\"store\":\"struct\",\"method\":\"createStruct\",\"args\":[\"C_BaseEntity\",\"${PAWN}\",1024]},
    {\"action\":\"store\",\"store\":\"struct\",\"method\":\"addField\",\"args\":[\"C_BaseEntity\",0,\"pointer\",\"vtable\"]},
    {\"action\":\"store\",\"store\":\"struct\",\"method\":\"addField\",\"args\":[\"C_BaseEntity\",816,\"pointer\",\"m_pGameSceneNode\"]},
    {\"action\":\"store\",\"store\":\"struct\",\"method\":\"addField\",\"args\":[\"C_BaseEntity\",848,\"int32\",\"m_iMaxHealth\"]},
    {\"action\":\"store\",\"store\":\"struct\",\"method\":\"addField\",\"args\":[\"C_BaseEntity\",852,\"int32\",\"m_iHealth\"]},
    {\"action\":\"store\",\"store\":\"struct\",\"method\":\"addField\",\"args\":[\"C_BaseEntity\",860,\"int32\",\"m_lifeState\"]},
    {\"action\":\"store\",\"store\":\"struct\",\"method\":\"addField\",\"args\":[\"C_BaseEntity\",960,\"float\",\"m_flSimulationTime\"]},
    {\"action\":\"store\",\"store\":\"struct\",\"method\":\"addField\",\"args\":[\"C_BaseEntity\",1011,\"uint8\",\"m_iTeamNum\"]},
    {\"action\":\"store\",\"store\":\"struct\",\"method\":\"setActiveStruct\",\"args\":[\"C_BaseEntity\"]},
    {\"action\":\"store\",\"store\":\"label\",\"method\":\"setLabel\",\"args\":[\"${PAWN}\",\"LocalPlayerPawn\"]},
    {\"action\":\"navigate\",\"tab\":\"structs\"},
    {\"action\":\"activity\",\"status\":\"idle\",\"message\":\"Local player ready\"}
  ]
}"
```

### 5. Read entity list (for other players)

Entity list structure:
- `client_base + dwEntityList` = CGameEntitySystem pointer
- Entity system + 0x10 = first chunk pointer
- Chunks contain 0x70-byte CEntityIdentity entries (512 per chunk)
- Index: `chunk = index >> 9`, `chunk_index = index & 0x1FF`
- Entity pointer at identity + 0x00
- Designer name pointer at identity + 0x20

```bash
# Read entity list base
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d "{\"action\":\"probe\",\"command\":\"read_ptr ${CLIENT_BASE}+0x3862C28\"}"

# Read first chunk pointer (entity_system + 0x10)
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d "{\"action\":\"probe\",\"command\":\"read_ptr <entity_system>+0x10\"}"

# Read entity identity at index N: chunk_base + (N & 0x1FF) * 0x70
# Entity pointer: identity + 0x00
# Designer name: read_ptr at identity + 0x20, then read_string
```

### 6. Scan for specific entity types

To find players, iterate low entity indices (typically 1-12 for controllers) and check designer names:

```bash
# For each index, read identity entry and check designer name
# Players have designer name containing "player" or class name containing "CitadelPlayerPawn"
# Controllers have "player_controller"
```

## Entity Types

| Designer Name | What | Teams |
|---|---|---|
| `player_controller` | Player controller | 2,3 |
| Player pawns | No designer name; resolve via controller `m_hPawn` | 2,3 |
| `npc_trooper` | Lane creep | 2,3 |
| `npc_boss_tier2` | Guardian | 2,3 |
| `npc_boss_tier3` | Patron | 2,3 |
| `npc_trooper_neutral` | Jungle camp | 4 |

## Tips

- Entity indices are dynamically allocated — don't assume fixed slots
- Controller-to-pawn: read `m_hPawn` (uint32 handle), index = handle & 0x7FFF
- If pawn pointer is NULL or 0, the player may be dead or disconnected
- Team 2 = Amber, Team 3 = Sapphire
- `m_lifeState`: 0 = alive, 2 = dead
- Use `m_flSimulationTime` staleness to detect dead entities (delta > 1.0s from local pawn)
- Global offsets change every Deadlock patch — check fresh schema dump
- Pattern scan to find offsets: `pattern 48 8B 05 ?? ?? ?? ?? client.dll --resolve 3`
