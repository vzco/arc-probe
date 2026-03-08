# ARC Probe Tutorial: Collaborative Reverse Engineering — Human Scanner + LLM Analysis

This is Part 3 of the ARC Probe tutorial series. The [first tutorial](tutorial.md) covered connecting, RTTI, and struct building. The [second](advanced-tutorial.md) covered function discovery and cross-references. This tutorial demonstrates the workflow ARC Probe was built for: **a human and an LLM working together to reverse engineer a live process**, each contributing what they're best at.

**The premise:** You don't need to be an expert reverse engineer. You need to know your target (what happens when you buy an item that increases health?), and the LLM needs to know how to read memory. Together, you can map out game internals that would take a solo researcher hours.

## Prerequisites

- ARC Probe GUI running and connected to the target process
- `probe-shell.dll` injected
- Claude Code with the ARC Probe plugin (provides the bridge skills)
- A schema dump (optional, but dramatically improves results)

---

## The Scenario

We're connected to Deadlock (a Source 2 game by Valve). We want to understand how hero health scaling works — specifically, how the game computes max health at different levels. We don't know where this data lives in memory. We just know that when we buy vitality items in-game, our max health changes.

## Part 1: The Human — Value Scanning

The human drives the Scanner tab. This is the part that requires game knowledge and physical interaction with the game — things the LLM can't do.

### Step 1: Get a Known Value

In-game, open the shop UI or character stats panel. Note your current **max health** — let's say it's `658`.

### Step 2: First Scan

Open the **Scanner** tab in ARC Probe. Set:
- **Value type:** `int32` (health is a 32-bit integer in Source 2)
- **Scan type:** `Exact`
- **Value:** `658`

Click **First Scan**. The scanner searches the entire process memory for every int32 that equals 658. You'll get thousands of results — health is a common-ish value and many unrelated memory locations happen to contain it.

### Step 3: Change the Value In-Game

Now do something in the game that changes your max health:
- Buy a vitality item (e.g., Extra Health, which adds +125 HP)
- Level up (each level adds a hero-specific HP bonus)
- Use an ability that temporarily modifies max health

Your max health changes to a new value — say `757`.

### Step 4: Next Scan

Back in the Scanner tab:
- **Value:** `757`
- Click **Next Scan**

The scanner filters the previous results, keeping only addresses where the value changed from 658 to 757. Thousands of results drop to maybe a dozen.

### Step 5: Narrow It Down

Repeat: change max health again (sell the item, buy a different one, level up), note the new value, scan again. After 2-3 rounds, you'll have 1-5 addresses. These are the real locations where max health is stored or computed.

### Step 6: Hand Off to the LLM

You've found an address: `0x55E7F03FB64`. You can see it in the scanner results. You could click it to view it in the hex viewer, but you don't know what surrounds it, what struct it belongs to, or what the data means.

This is where you hand it to the LLM:

> "Here's an address: `0x55E7F03FB64`. I found it by scanning for max health changes. Can you figure out what it is?"

---

## Part 2: The LLM — Memory Analysis

The LLM receives the address and begins systematic analysis through the Claude Bridge. Here's the thought process, step by step.

### Step 1: Verify Connectivity and Dump the Address

First things first — make sure the probe is alive and read what's at the address:

```bash
# Health check
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"ping"}'

# Hex dump 256 bytes starting at the address
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"dump 0x55E7F03FB64 256"}'
```

The dump reveals:
```
0x55E7F03FB64: AB 09 00 00 0E 0A 00 00 71 0A 00 00 D4 0A 00 00
0x55E7F03FB74: 37 0B 00 00 9A 0B 00 00 FD 0B 00 00 60 0C 00 00
...
```

### Step 2: Pattern Recognition — The Key Insight

Reading these as little-endian int32s:
```
0x09AB = 2475
0x0A0E = 2574
0x0A71 = 2673
0x0AD4 = 2772
0x0B37 = 2871
...
```

The difference between consecutive values: **2574 - 2475 = 99. 2673 - 2574 = 99. 2772 - 2673 = 99.**

A perfectly constant delta of 99 across every entry. This is not random game state — it's a **precomputed lookup table** where each entry = `99 * index`.

### Step 3: Look at Surrounding Memory

Is this the only table, or part of a larger structure? Dump memory before and after:

```bash
# 256 bytes before
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"dump 0x55E7F03FA64 256"}'

# 256 bytes after
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"dump 0x55E7F03FC64 256"}'
```

Result: **five adjacent arrays**, each with a different constant delta:

| Start Address | Step | Entries | Values |
|---|---|---|---|
| `0xF984` | 87 | 40 | 87, 174, ..., 3480 |
| `0xFA44` | 78 | 40 | 78, 156, ..., 3120 |
| `0xFB04` | **99** | **45** | 99, 198, ..., **4455** |
| `0xFBC4` | 72 | 45 | 72, 144, ..., 3240 |
| `0xFC84` | 105 | 45 | 105, 210, ..., 4725 |

Each array is separated by a 32-byte marker:
```
00 00 00 00  00 00 00 00  00 00 00 00    (12 bytes padding)
XX XX XX XX  XX XX XX XX                  (8-byte hash identifier)
00 00 00 00                               (padding)
FF 00 12 21  or  9D 8D 78 FF             (4-byte marker)
00 00 00 00                               (padding)
```

### Step 4: Check for RTTI (Is This a C++ Object?)

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" \
  -d '{"action":"probe","command":"rtti resolve 0x55E7F03FB64"}'
# → "No valid RTTI found at address"
```

No RTTI — this is heap-allocated data, not a C++ virtual class. Likely loaded from game data files.

### Step 5: Form a Hypothesis

The clues:
1. **Constant-delta int32 arrays** → precomputed lookup tables
2. **Steps of 72, 78, 87, 99, 105** → all in the range of per-level stat gains in a MOBA
3. **40-45 entries per array** → number of hero levels in Deadlock
4. **Found by scanning for max health changes** → health-related data
5. **Multiple adjacent arrays** → different heroes or different stats

**Hypothesis:** These are per-level cumulative health bonus tables. Each array represents a different hero's health scaling: at level N, add `array[N-1]` to base health. The step value IS the per-level health gain for that hero.

### Step 6: Build a Preliminary Struct in the GUI

Even before confirming the hypothesis, create a struct so the human can see the data:

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"activity","status":"working","message":"Building struct..."},
    {"action":"store","store":"struct","method":"createStruct",
     "args":["PerLevelScaling_Step99","0x55E7F03FB04",256]},
    {"action":"store","store":"struct","method":"addField",
     "args":["PerLevelScaling_Step99",0,"int32","level_1_bonus (99)"]},
    {"action":"store","store":"struct","method":"addField",
     "args":["PerLevelScaling_Step99",96,"int32","level_25_bonus (2475) USER_ADDR"]},
    {"action":"store","store":"struct","method":"addField",
     "args":["PerLevelScaling_Step99",176,"int32","level_45_bonus (4455)"]},
    {"action":"store","store":"struct","method":"addField",
     "args":["PerLevelScaling_Step99",180,"int32","terminator (0)"]},
    {"action":"store","store":"struct","method":"addField",
     "args":["PerLevelScaling_Step99",184,"uint32","marker_0xFF001221"]},
    {"action":"store","store":"struct","method":"addField",
     "args":["PerLevelScaling_Step99",192,"int32","next_array_level1 (72)"]},
    {"action":"navigate","tab":"structs"},
    {"action":"activity","status":"idle","message":"Done"}
  ]
}'
```

The human can now see the struct in the GUI with live values, confirming the pattern visually.

---

## Part 3: Cross-Referencing with the Schema Dump

This is where the analysis goes from "probably health data" to "definitely `CitadelHeroData_t.m_mapScalingStats`."

### The Schema Dump

A schema dump (generated by source2gen or dezlock-dump against the game binary) contains every class, field, offset, and enum in the game's type system. It's the Rosetta Stone of Source 2 RE.

### Step 1: Search for Health-Related Scaling

```bash
grep -i "MaxHealth\|HealthScale\|ScalingStats" schema-dump/client.txt
```

Key finding:
```
CitadelHeroData_t.m_mapScalingStats = 0xC40
  (CUtlOrderedMap< EStatsType, HeroScalingStat_t >, 40)
```

### Step 2: Find HeroScalingStat_t

```bash
grep "HeroScalingStat_t\." schema-dump/server.txt
```

```
HeroScalingStat_t.eScalingStat = 0x0 (EStatsType, 4)
HeroScalingStat_t.flScale = 0x4 (float32, 4)
```

An 8-byte struct: stat type enum + float scale factor. The `flScale` value IS the per-level bonus — `99.0` for our hero.

### Step 3: Find the Enum

```bash
grep "EStatsType::" schema-dump/server.txt | head -10
```

```
EStatsType::EWeaponDPS = 0
EStatsType::EMeleeDamage_DEPRECATED = 1
EStatsType::EMaxHealth = 2          ← THIS ONE
EStatsType::EClipSize = 3
EStatsType::EBaseHealthRegen = 4
...
```

**`EMaxHealth = 2`** — confirmed. The scaling data is for max health.

### Step 4: Understand the Full Chain

```
CitadelHeroData_t (one per hero, loaded from VData)
├── +0xC18  m_mapStartingStats    → base values (e.g., base health = 550)
├── +0xC40  m_mapScalingStats     → per-level scaling definitions
│           └── key: EStatsType (e.g., EMaxHealth = 2)
│           └── value: HeroScalingStat_t
│                      ├── eScalingStat: EMaxHealth (2)
│                      └── flScale: 99.0 (HP per level)
└── +0xC68  m_groundDashPositionCurve
```

The runtime then precomputes `int32(flScale * level)` for levels 1-45 into the lookup arrays we found. The game reads `max_health = base_health + lookup[current_level - 1]` for instant access without floating-point math per frame.

### Step 5: Update the GUI with Real Names

```bash
curl -s -X POST http://localhost:9996 -H "Content-Type: application/json" -d '{
  "action":"batch","actions":[
    {"action":"store","store":"label","method":"setLabel",
     "args":["0x55E7F03FB04",
             "HeroScalingStat cache: EMaxHealth flScale=99.0"]},
    {"action":"store","store":"label","method":"setLabel",
     "args":["0x55E7F03FB64",
             "Level 25 cumulative: +2475 HP (m_mapScalingStats)"]},
    {"action":"navigate","tab":"structs"}
  ]
}'
```

---

## Part 4: What Each Side Contributed

### The Human Brought:
- **Game knowledge** — knowing that buying a vitality item increases max health
- **Physical interaction** — actually playing the game, buying items, observing stat changes
- **Value scanning** — using the Scanner tab to narrow thousands of results to one address
- **The schema dump** — running source2gen against the binary to produce the type database

### The LLM Brought:
- **Pattern recognition** — instantly spotting constant-delta sequences in hex dumps (99, 99, 99...)
- **Systematic exploration** — reading memory before/after the address to find the full table structure
- **Breadth of analysis** — checking RTTI, module boundaries, separator patterns, multiple adjacent arrays
- **Schema cross-referencing** — searching the dump for `HeroScalingStat_t`, `EStatsType`, `m_mapScalingStats`
- **Struct building** — creating annotated structs in the GUI with proper field names and live values
- **Documentation** — explaining the full data chain from VData to runtime cache

### Neither Could Do Alone:
- The human can't efficiently analyze 1500+ bytes of hex to spot a delta-99 pattern across 45 entries
- The LLM can't buy an item in-game and observe max health changing from 658 to 757
- The human might not know to search `server.txt` for `HeroScalingStat_t`
- The LLM might not know which of 97 stat types is relevant without the human's "I was scanning for health" context

---

## Part 5: The Complete Data Map

Here's what we mapped in approximately 5 minutes of collaborative work:

### Runtime Per-Level Scaling Cache

```
Address Range          Step   Entries  Interpretation
─────────────────────  ────   ───────  ──────────────────────
0x55E7F03F984–FAE0     87     40       Hero A: +87 HP/level
0x55E7F03FA44–FAE0     78     40       Hero B: +78 HP/level
0x55E7F03FB04–FBB8     99     45       Hero C: +99 HP/level ★
0x55E7F03FBC4–FC78     72     45       Hero D: +72 HP/level
0x55E7F03FC84–FD38     105    45       Hero E: +105 HP/level
```

### Schema Chain

```
CitadelHeroData_t
  ├── m_HeroID (+0x28)              → hero identifier
  ├── m_strHeroSortName (+0x30)     → hero name string
  ├── m_mapStartingStats (+0xC18)   → CUtlOrderedMap<EStatsType, float>
  │     └── EMaxHealth(2) → 550.0   (base health)
  └── m_mapScalingStats (+0xC40)    → CUtlOrderedMap<EStatsType, HeroScalingStat_t>
        └── EMaxHealth(2) → { eScalingStat: 2, flScale: 99.0 }

Runtime cache: int32 array[45] = { 99, 198, 297, ..., 4455 }
Formula: max_health = base_health + cache[level - 1]
At level 25: max_health = 550 + 2475 = 3025
```

### EStatsType Enum (Key Values)

| ID | Name | Description |
|----|------|-------------|
| 0 | EWeaponDPS | Weapon DPS |
| 2 | **EMaxHealth** | **Max health** |
| 4 | EBaseHealthRegen | Base HP regen |
| 7 | EMaxMoveSpeed | Move speed |
| 16 | ELightMeleeDamage | Light melee damage |
| 59 | ETechPower | Tech/spirit power |
| 64 | ELevelUpBaseWeaponDamageIncrease | Per-level weapon damage |
| 66 | ELevelUpBaseHealthIncrease | Per-level health increase |

---

## Key Takeaways

1. **The scanner is the human's superpower.** No amount of static analysis can replace "I changed this value in-game and scanned for it." The scanner turns unknown addresses into known data points.

2. **Pattern recognition is the LLM's superpower.** Spotting that 45 consecutive int32s differ by exactly 99 is trivial for an LLM and tedious for a human staring at hex.

3. **Schema dumps are force multipliers.** Without the dump, we'd have "probably health scaling tables." With it, we have `CitadelHeroData_t.m_mapScalingStats → HeroScalingStat_t{EMaxHealth, 99.0}` — exact class, field, and enum names.

4. **The bridge makes collaboration seamless.** The human uses the GUI. The LLM uses the bridge. Both see the same structs, labels, and hex views. The LLM can build a struct and the human sees it appear in real-time.

5. **Start with what you know, end with what the code knows.** Human game knowledge ("health changed") → scanner address → LLM memory analysis → schema cross-reference → fully annotated struct. Each step narrows the unknowns.
