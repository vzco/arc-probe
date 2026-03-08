# Panorama Panel Bible — Source 2 HUD Internals for Deadlock

A comprehensive reverse-engineering reference for Valve's Panorama UI system as implemented in Deadlock (Source 2). Covers architecture, data flow, hooking strategies, and practical techniques for extracting game state through the UI layer.

**Target game:** Deadlock (`deadlock.exe`)
**Engine:** Source 2 (shared with CS2, Dota 2)
**Key modules:** `panorama.dll`, `panoramauiclient.dll`, `client.dll`, `engine2.dll`
**Schema source:** dezlock-dump (2026-03-08)

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [How Panorama Works](#2-how-panorama-works)
3. [Data Flow: C++ to JavaScript](#3-data-flow-c-to-javascript)
4. [Game Rules & Match Clock](#4-game-rules--match-clock)
5. [HUD Panel Catalog](#5-hud-panel-catalog)
6. [Modifier & Ability VData → HUD](#6-modifier--ability-vdata--hud)
7. [Panel Engine Internals](#7-panel-engine-internals)
8. [Event System](#8-event-system)
9. [JavaScript V8 Integration](#9-javascript-v8-integration)
10. [Hooking Strategies](#10-hooking-strategies)
11. [Practical Guide: Finding Things with ARC Probe](#11-practical-guide-finding-things-with-arc-probe)
12. [Reference Tables](#12-reference-tables)

---

## 1. Architecture Overview

Panorama is Valve's HTML/CSS/JS-like UI framework embedded in Source 2. It replaces the old VGUI system. Think of it as a custom web browser running inside the game engine.

```
                    ┌──────────────────────────────────────────────┐
                    │              Source 2 Engine                  │
                    │                                              │
  ┌──────────┐     │   ┌─────────────┐     ┌──────────────────┐  │
  │ Game C++  │────►│   │  Panorama    │     │  Panorama UI     │  │
  │ (client   │     │   │  Engine      │     │  Client          │  │
  │  .dll)    │     │   │ (panorama    │     │ (panoramaui      │  │
  │           │     │   │  .dll)       │     │  client.dll)     │  │
  └──────────┘     │   └──────┬──────┘     └────────┬─────────┘  │
                    │          │                      │             │
                    │   ┌──────▼──────────────────────▼─────────┐  │
                    │   │           V8 JavaScript Engine         │  │
                    │   │   (embedded, executes panel scripts)   │  │
                    │   └───────────────────────────────────────┘  │
                    │          │                                    │
                    │   ┌──────▼──────────────────────────────┐    │
                    │   │  Layout XML + CSS + JS files         │    │
                    │   │  (loaded from VPK / filesystem)      │    │
                    │   └─────────────────────────────────────┘    │
                    └──────────────────────────────────────────────┘
```

### Module Responsibilities

| Module | Role |
|--------|------|
| `panorama.dll` | Core engine — panel tree, layout, style, rendering, V8 runtime |
| `panoramauiclient.dll` | Widget library — CLabel, CButton, CImagePanel, CDropDown, etc. |
| `panorama_text_pango.dll` | Text rendering via Pango/HarfBuzz |
| `client.dll` | Game-specific HUD panels (CCitadelHud*), entity system, game rules |
| `engine2.dll` | CGameUIService bridge between engine and Panorama |
| `v8.dll` | Google V8 JavaScript engine (embedded) |

### The Three Layers

1. **Engine layer** (`panorama.dll`) — CUIEngine, CUIPanel, CTopLevelWindow, CPanelStyle, CLayoutManager
2. **Widget layer** (`panoramauiclient.dll`) — CLabel, CButton, CImagePanel, CConsole, CDebugger, etc.
3. **Game layer** (`client.dll`) — CCitadelHud* panels, C_PointClientUIHUD, C_PointClientUIWorldPanel

---

## 2. How Panorama Works

### Panel Tree

Everything in Panorama is a **panel** (`CUIPanel`). Panels form a tree:

```
CTopLevelWindow (root window)
  └── Root CUIPanel
        ├── HUD container panel
        │     ├── CCitadel_Hud_Health
        │     ├── CCitadel_Hud_Abilities
        │     ├── CCitadel_Hud_TopBar
        │     └── CCitadel_Hud_Modifiers
        ├── Menu container panel
        │     └── CCitadel_Hud_EscapeMenu
        └── Overlay container panel
              └── CCitadelHudDamageIndicator
```

### Layout Files

UI is defined in XML layout files (`.xml` / `.vxml`) loaded from VPK archives:

```xml
<root>
  <Panel class="HudHealth">
    <Label id="HealthText" text="{d:health}" />
    <Panel class="HealthBar" style="width: {d:health_pct}%;" />
  </Panel>
</root>
```

### CSS Styling

Panorama uses a CSS-like style system with `CPanelStyle` (156 vtable functions). Styles reference `.css`/`.vcss` files. The `CStyleProperty*` family has 100+ property classes:

- Background: `CStylePropertyBackgroundColor`, `CStylePropertyBackgroundImage`, etc.
- Border: `CStylePropertyBorder`, `CStylePropertyBorderRadius`
- Text: `CStylePropertyFont`, `CStylePropertyTextColor`, `CStylePropertyTextShadow`
- Layout: `CStylePropertyMargin`, `CStylePropertyPadding`, `CStylePropertyPosition`
- Effects: `CStylePropertyOpacity`, `CStylePropertyTransform`, `CStylePropertyBlur`
- Animation: `CStylePropertyTransition`, `CStylePropertyAnimation`

### JavaScript

Each panel can have associated JavaScript that runs in a V8 isolate. JS has access to:
- Panel manipulation (find children, set styles, show/hide)
- Game API bindings (registered C++ functions callable from JS)
- Event subscription (listen for game events)
- Dialog variable access (read/write variables set by C++)
- Custom event dispatch

---

## 3. Data Flow: C++ to JavaScript

This is the core question: **how does game state get into the HUD?**

There are **four primary mechanisms:**

### 3.1 Dialog Variables (SetDialogVariable)

The **main data channel**. C++ code sets named variables on panels, JS reads them.

```
C++ side:                                    JS side:
panel->SetDialogVariable("health", 150);  →  $.GetContextPanel().GetDialogVariable("health")
panel->SetDialogVariableInt("ammo", 30);  →  $.GetContextPanel().GetDialogVariableInt("ammo")
panel->SetDialogVariableTime("cd", 5.2);  →  $.GetContextPanel().GetDialogVariableTime("cd")
```

In layout XML, dialog variables are referenced with `{d:variable_name}`:
```xml
<Label text="HP: {d:health}" />
```

**Debug panel**: `CDebugPanelDialogVariables` in `panoramauiclient.dll` — an inspector that shows all dialog variables on a panel. Has 84 vtable functions, stores variable list at field `+0x08`.

### 3.2 Network-Replicated Entity Fields

Fields marked `[MNetworkEnable]` auto-sync from server to client entity memory. When fields change, the schema system fires `[MNetworkChangeCallback]` which can trigger UI updates.

```
Server entity field changes
    ↓ (delta networking)
Client entity memory updated
    ↓ ([MNetworkChangeCallback] fires)
C++ callback runs
    ↓ (SetDialogVariable or custom logic)
Panorama panel updated
```

Key fields with `[MNetworkChangeCallback]`:
- `C_CitadelGameRules.m_eGameState` — game phase changes
- `CCitadelAbilityComponent.m_hSelectedAbility` — ability selection changes
- `CCitadelAbilityComponent.m_flTimeScale` — haste/slow effects

### 3.3 JavaScript API Bindings

Source 2 registers C++ functions into the V8 context as global API objects. Deadlock/Source 2 likely exposes objects like:

| JS Object | Purpose |
|-----------|---------|
| `Game` | Game state queries (time, mode, phase) |
| `Players` | Player data (health, team, name) |
| `Entities` | Entity system access |
| `Abilities` | Ability state (cooldowns, charges) |
| `Items` | Item/upgrade information |
| `GameEvents` | Event subscription |
| `$` | Panel manipulation (jQuery-like) |

These are registered at engine init through `CUIEngine`'s JavaScript runtime subsystem (field at `CUIEngine+0x0720`).

### 3.4 Protobuf User Messages

Server sends structured messages to client, which dispatches them to Panorama:

| Message | Fields | Purpose |
|---------|--------|---------|
| `CCitadelUserMsg_HudGameAnnouncement` | `title_locstring`, `description_locstring`, `dialog_variable_name`, `dialog_variable_locstring` | Game announcements with dialog variable injection |
| `CUserMessageHudMsg` | `channel`, `x`, `y`, `color1`, `color2`, `effect`, `message` | Generic positioned HUD messages |
| `CUserMessageHudText` | `message` | Simple text display |
| `CUserMsg_HudError` | `order_id` | Error ordering |
| `CUserMessageResetHUD` | — | Reset all HUD state |

The `CCitadelUserMsg_HudGameAnnouncement` is particularly interesting — it directly sets dialog variables by name, showing the pipeline from server to UI.

### 3.5 Data Flow Summary

```
                Game State Sources
                       │
    ┌──────────────────┼──────────────────┐
    │                  │                  │
    ▼                  ▼                  ▼
Entity Fields    Game Rules         Server Messages
(MNetworkEnable) (C_CitadelGameRules) (Protobuf UserMsg)
    │                  │                  │
    │                  │                  │
    ▼                  ▼                  ▼
    └──────────► SetDialogVariable ◄──────┘
                       │
                       ▼
              Panorama Panel Tree
                       │
              ┌────────┼────────┐
              │        │        │
              ▼        ▼        ▼
           {d:var}   JS API   CSS Binding
           in XML   calls     (dynamic styles)
```

---

## 4. Game Rules & Match Clock

### C_CitadelGameRules — The Master Game State

**Size:** 0x9F40 (40,768 bytes)
**Inheritance:** C_TeamplayRules → C_MultiplayRules → C_GameRules → CGameEventListener → IGameEventListener2 → IEntityListener

Access: Find `C_CitadelGameRulesProxy` entity (designer name `"citadel_game_rules"`) → read ptr at `+0x5F0` → `C_CitadelGameRules*`

### Timing Fields

| Offset | Type | Field | Purpose |
|--------|------|-------|---------|
| 0x5C | GameTime_t | `m_fLevelStartTime` | When map loaded |
| 0x60 | GameTime_t | `m_flGameStartTime` | When gameplay began (post-countdown) |
| 0x64 | GameTime_t | `m_flGameStateStartTime` | Current phase start |
| 0x68 | GameTime_t | `m_flGameStateEndTime` | Current phase end |
| 0x6C | GameTime_t | `m_flRoundStartTime` | Round start |
| 0x70 | float32 | `m_flPlayOfTheGameStateEndTime` | POTG duration |
| **0x9E3C** | **float32** | **`m_flMatchClockAtLastUpdate`** | **THE HUD MATCH TIMER (elapsed seconds)** |
| 0x9E38 | int32 | `m_nMatchClockUpdateTick` | Last clock sync tick |
| 0x9EB4 | GameTime_t | `m_flPreGameWaitEndTime` | Pre-game countdown end |
| 0x9EC4 | GameTime_t | `m_flGGEndsAtTime` | GG/surrender end |
| 0x9F08 | GameTime_t | `m_flHeroDiedTime` | Most recent hero death |

### Game State Enum

**EGameState** (int32 at offset 0x74, `[MNetworkEnable][MNetworkChangeCallback]`):

| Value | Name | Description |
|-------|------|-------------|
| 0 | `EGameState_Invalid` | Not initialized |
| 1 | `EGameState_Init` | Initialization |
| 2 | `EGameState_WaitingForPlayersToJoin` | Lobby |
| 3 | `EGameState_HeroSelection` | Hero draft |
| 4 | `EGameState_MatchIntro` | Loading/intro |
| 5 | `EGameState_WaitForMapToLoad` | Map loading |
| 6 | `EGameState_PreGameWait` | Freeze/countdown |
| 7 | `EGameState_GameInProgress` | Active gameplay |
| 8 | `EGameState_PostGame` | Match ended |
| 9 | `EGameState_PostGame_PlayOfTheGame` | POTG replay |
| 10 | `EGameState_Abandoned` | Abandoned |
| 11 | `EGameState_End` | Final state |

### Pause System

| Offset | Type | Field | Purpose |
|--------|------|-------|---------|
| 0x9E30 | bool | `m_bServerPaused` | Pause active `[MNetworkEnable]` |
| 0x9E34 | int32 | `m_iPauseTeam` | Team that paused `[MNetworkEnable]` |
| 0x9E40 | float64 | `m_flPauseTime` | Accumulated pause time |
| 0x9E48 | CPlayerSlot | `m_pausingPlayerId` | Who paused |
| 0x9E4C | CPlayerSlot | `m_unpausingPlayerId` | Who unpaused |
| 0x9E50 | float32 | `m_fPauseRawTime` | Raw clock at pause start |
| 0x9E54 | float32 | `m_fPauseCurTime` | Game time at pause start |
| 0x9E58 | float32 | `m_fUnpauseRawTime` | Raw clock at unpause |
| 0x9E5C | float32 | `m_fUnpauseCurTime` | Game time at unpause |

### Other Game State Fields

| Offset | Type | Field | Notes |
|--------|------|-------|-------|
| 0x58 | bool | `m_bFreezePeriod` | Initial freeze |
| 0x74 | EGameState | `m_eGameState` | Current phase `[MNetworkChangeCallback]` |
| 0x78 | CHandle | `m_hTowerAmber` | Amber base entity |
| 0x7C | CHandle | `m_hTowerSapphire` | Sapphire base entity |
| 0x80-0x83 | bool×4 | `m_bEnemyIn*Base` | Base invasion flags |
| 0x84 | Vector | `m_vMinimapMins` | Minimap bounds min |
| 0x90 | Vector | `m_vMinimapMaxs` | Minimap bounds max |
| 0x9C | bool | `m_bMatchSafeToAbandon` | Safe to leave |
| 0x9E-0xA3 | bool×6 | Testing flags | NoDeath, FastCD, UnlimAmmo, etc. |
| 0xA4 | ECitadelMatchMode | `m_eMatchMode` | Match mode |
| 0xA8 | ECitadelGameMode | `m_eGameMode` | Game mode |
| 0xBC | int32 | `m_iWinningTeam` | Winner |
| 0xCC | int32 | `m_iMidbossKillCount` | Mid boss kills `[MNetworkEnable]` |
| 0xD0 | int32 | `m_iAmberRejuvCount` | Amber rejuv count `[MNetworkEnable]` |
| 0xD4 | int32 | `m_iSapphireRejuvCount` | Sapphire rejuv count `[MNetworkEnable]` |
| 0xD8 | float32 | `m_tNextMidBossSpawnTime` | Next mid boss `[MNetworkEnable]` |
| 0x9EC8 | MatchID_t | `m_unMatchID` | Match ID (8 bytes) |
| 0x9F18 | CStreetBrawlController | `m_tStreetBrawl` | Street Brawl state (40 bytes) |

### Server-Side Extended Fields (CCitadelGameRules, server.dll)

The server version (0x29F8 bytes) adds time-scaling fields not present on client:

| Offset | Type | Field | Purpose |
|--------|------|-------|---------|
| 0x17E0 | GameTime_t | `m_flTimeScaleStart` | Time scale effect start |
| 0x17FC | float32 | `m_flTimeScale` | Current time multiplier |
| 0x1804 | bool | `m_bTimeScaleActive` | Time scale active flag |
| 0x0438 | GameTime_t | `m_flGameTimeAllPlayersDisconnected` | All disconnected time |

### Reading Match Time with ARC Probe

```bash
# 1. Find the game rules proxy entity
probe.exe "pattern 63 69 74 61 64 65 6C 5F 67 61 6D 65 5F 72 75 6C 65 73 client.dll"

# Or scan entities for it:
# (scan entity list, check designer names for "citadel_game_rules")

# 2. Read the proxy → game rules pointer
probe.exe "read_ptr <proxy_entity>+0x5F0"

# 3. Read match clock
probe.exe "read_float <game_rules>+0x9E3C"

# 4. Read game state enum
probe.exe "read_int <game_rules>+0x74"
```

---

## 5. HUD Panel Catalog

### Deadlock HUD Panels (95+ classes in client.dll)

#### System & Container

| Class | Vtable Size | Purpose |
|-------|-------------|---------|
| `CCitadelHudGameSystem` | 63 funcs | Main HUD system coordinator |
| `CCitadel_Hud_Main` | 85 funcs | Primary HUD container |

#### Health & Status

| Class | Key Fields | Purpose |
|-------|------------|---------|
| `CCitadel_Hud_Health` | `+0xDC` (int32) | Health display |
| `CCitadel_Hud_Health_Container` | — | Health display wrapper |
| `CCitadel_Hud_Health_SingleBar` | — | Single health bar |
| `CCitadel_Hud_Health_Stacked` | — | Stacked health display |
| `CCitadel_Hud_Shields_Container` | — | Shield display |
| `CCitadelHudActivePlayerStats` | `+0x84` (uint32) | Player statistics |
| `CCitadelHudMovementSpeed` | — | Movement speed indicator |
| `CCitadel_Hud_SoulAP_Container` | — | Soul/AP display |

#### Abilities

| Class | Key Fields | Purpose |
|-------|------------|---------|
| `CCitadel_Hud_Abilities` | `+0x28`, `+0x40`, `+0x9A-9C` | Ability buttons container |
| `CCitadel_Hud_AbilityButton` | 90 vtable funcs | Single ability button |
| `CCitadel_Hud_AbilityUpgradeList` | 90 vtable funcs | Upgrade selection list |
| `CCitadel_Hud_AbilityUpgradePips` | — | Upgrade level indicators |
| `CCitadel_Hud_AbilityResource` | — | Ability resource bars |
| `CCitadel_Hud_PassiveAbilities` | — | Passive ability display |

#### HUD Elements (Ability Sub-Components)

| Class | Key Fields | Purpose |
|-------|------------|---------|
| `CCitadel_AbilityHUDPanels_Container` | `+0x38`, `+0x40` | Container for ability panels |
| `CCitadel_AbilityHUDElement_Gun` | `+0x16C` | Weapon/ammo display |
| `CCitadel_AbilityHUDElement_Progress` | `+0x38` | Cooldown progress bars |
| `CCitadel_AbilityHUDElement_ButtonHint` | — | Keybind hints |
| `CCitadel_AbilityHUDElement_UnitTarget` | — | Target unit indicator |
| `CCitadel_AbilityHUDElement_ZiplineLaneSelection` | — | Zipline lane picker |

#### Combat Feedback

| Class | Key Fields | Purpose |
|-------|------------|---------|
| `CCitadelHudDamageIndicator` | `+0x4A`, `+0x58` | Directional damage indicator |
| `CCitadelHudDamageImpact` | — | Impact flash |
| `CCitadelHudDamageEnemyOffscreenIndicator` | — | Off-screen damage |
| `CCitadel_Hud_DamageReport` | — | Damage statistics |
| `CCitadel_Hud_DamageSummary` | — | Post-game damage summary |
| `CCitadelHudAbilityInterrupt` | `+0x20`, `+0x28`, `+0x30` | Ability interrupt notification |

#### Modifiers & Buffs

| Class | Purpose |
|-------|---------|
| `CCitadel_Hud_Modifiers` | Modifier/buff display |
| `CCitadel_Hud_Modifiers_Entry` | Individual modifier entry |
| `CCitadel_Hud_ModifierHistory` | Modifier history log |
| `CCitadelHudModifierIcons` | Buff/debuff icons |
| `CCitadelHudModifierMessage` | Modifier notification |
| `CCitadelHudModBarGraph` | Modifier bar graph |

#### Top Bar

| Class | Purpose |
|-------|---------|
| `CCitadel_Hud_TopBar` | Top bar container |
| `CCitadel_Hud_TopBar_Player` | Individual player stat |
| `CCitadel_Hud_TopBar_Team` | Team stat |
| `CCitadel_Hud_TopBar_LaneSwapContainer` | Lane swap indicator |
| `CCitadel_Hud_TopBarChat` | Chat in top bar |

#### Casting & Channels

| Class | Purpose |
|-------|---------|
| `CCitadel_Hud_Castbars` | Cast/channel bar display |
| `CCitadel_Hud_Castbar_Entry` | Individual cast bar |

#### Data Feeds

| Class | Purpose |
|-------|---------|
| `CHudDataFeed` | Kill feed / event feed |
| `CHudDataFeedBossKilled` | Boss kill announcement |
| `CHudDataFeedGenericTeamMsg` | Team event message |
| `CHudDataFeedInfo` | Event info |

#### Minimap

| Class | Purpose |
|-------|---------|
| `CHudMinimap` | Minimap display |
| `CHudMinimapRender` | Minimap rendering engine |
| `CCitadel_Hud_MinimapEffects` | Minimap overlays |

#### Game Phase Screens

| Class | Purpose |
|-------|---------|
| `CCitadel_Hud_Pregame` | Pre-match lobby |
| `CCitadel_Hud_Pregame_Full` | Full pregame screen |
| `CCitadel_Hud_Pregame_Simple` | Simplified pregame |
| `CCitadel_Hud_Pregame_HeroReveal` | Hero reveal phase |
| `CCitadel_Hud_Pregame_TeamReveal` | Team reveal phase |
| `CCitadel_Hud_PregameCountdown` | Countdown timer |
| `CCitadel_Hud_MatchStart` | Match start announcements |
| `CCitadel_Hud_MatchEnd` | End-of-match screen |
| `CCitadel_Hud_PlayOfTheGame` | POTG highlight |
| `CCitadel_Hud_ReportCard` | Post-game report card |

#### Shopping

| Class | Purpose |
|-------|---------|
| `CCitadel_Hud_Quickbuy` | Quick item purchase |
| `CCitadel_Hud_Quickbuy_Entry` | Single quickbuy item |
| `CCitadel_Hud_Hero_Shop` | Hero shop interface |
| `CCitadel_Hud_GoldHistory` | Gold earned history |
| `CCitadel_Hud_ItemBarGraph` | Item bar graph |

#### Screen Effects

| Class | Purpose |
|-------|---------|
| `CCitadel_Hud_ScreenEffect_HornetScope` | Hornet scope overlay |
| `CCitadel_Hud_ScreenEffect_LowHealth` | Low health visual |
| `CCitadel_Hud_ScreenEffect_Muted` | Muted status |
| `CCitadel_Hud_ScreenEffect_Silenced` | Silenced status |
| `CCitadel_Hud_ScreenEffect_Stunned` | Stunned status |

#### Communication

| Class | Purpose |
|-------|---------|
| `CCitadel_Hud_ChatWheel` | Quick chat wheel |
| `CCitadel_Hud_ChatWheelBase` | Chat wheel base |
| `CCitadelHudQuickResponse` | Quick response system |
| `CCitadelHudPingIndicator` | Ping/latency display |

#### Miscellaneous

| Class | Purpose |
|-------|---------|
| `CCitadel_Hud_EscapeMenu` | Pause/escape menu |
| `CCitadel_Hud_HeroBuilds` | Hero build info |
| `CCitadel_Hud_Hero_Testing` | Hero testing mode |
| `CCitadel_Hud_Hideout` | Hideout menu |
| `CCitadel_Hud_Hints` | In-game hints |
| `CCitadel_Hud_JoinTeam` | Team selection |
| `CCitadel_Hud_PausedDialog` | Pause dialog |
| `CCitadel_Hud_PlayerIntents` | Player objectives |
| `CCitadel_Hud_Replay` | Replay viewer |
| `CCitadel_Hud_Sandbox_Tutorial` | Sandbox tutorial |
| `CCitadel_Hud_Personal_Best` | Personal best stats |
| `CCitadelHudSubtitles` | Subtitles |
| `CCitadelHudZiplineBoostIcon` | Zipline boost |
| `CCitadelHudStreetBrawlRoundScoringDisplay` | Street Brawl scoring |
| `CHudCloseCaption` | Closed captions |
| `CHudSpectateBar` | Spectator bar |
| `CHudIcons` | Icon display |
| `CHudSkinModifier` | Skin display |

### World UI Entities

These are in-world 3D panels (floating health bars, objective markers, etc.):

#### C_BaseClientUIEntity (size 0x9D8)

Base class for all UI entities. Parent: `C_BaseModelEntity`.

| Offset | Type | Field | Notes |
|--------|------|-------|-------|
| 0x9A8 | bool | `m_bEnabled` | Panel active `[MNetworkEnable]` |
| 0x9B0 | CUtlSymbolLarge | `m_DialogXMLName` | XML layout path `[MNetworkEnable]` |
| 0x9B8 | CUtlSymbolLarge | `m_PanelClassName` | JS panel class `[MNetworkEnable]` |
| 0x9C0 | CUtlSymbolLarge | `m_PanelID` | Unique panel ID |
| 0x9D0-0x… | CEntityOutput×10 | `m_CustomOutput0-9` | Event dispatch outputs (32 bytes each) |

#### C_PointClientUIHUD (size 0xB98)

2D HUD panel entity. Extends `C_BaseClientUIEntity`.

| Offset | Type | Field | Notes |
|--------|------|-------|-------|
| 0xB50 | bool | `m_bIgnoreInput` | Click-through `[MNetworkEnable]` |
| 0xB54 | float32 | `m_flWidth` | Panel width `[MNetworkEnable]` |
| 0xB58 | float32 | `m_flHeight` | Panel height `[MNetworkEnable]` |
| 0xB5C | float32 | `m_flDPI` | DPI scaling `[MNetworkEnable]` |
| 0xB60 | float32 | `m_flInteractDistance` | Interaction range `[MNetworkEnable]` |
| 0xB64 | float32 | `m_flDepthOffset` | Z-ordering `[MNetworkEnable]` |
| 0xB68 | uint32 | `m_unOwnerContext` | Owner filter `[MNetworkEnable]` |
| 0xB6C | uint32 | `m_unHorizontalAlign` | H-align `[MNetworkEnable]` |
| 0xB70 | uint32 | `m_unVerticalAlign` | V-align `[MNetworkEnable]` |
| 0xB74 | uint32 | `m_unOrientation` | Orientation `[MNetworkEnable]` |
| 0xB78 | bool | `m_bAllowInteractionFromAllSceneWorlds` | `[MNetworkEnable]` |
| 0xB80 | C_NetworkUtlVectorBase | `m_vecCSSClasses` | CSS classes `[MNetworkEnable]` |

#### C_PointClientUIWorldPanel (size 0xBF0)

3D world-space panel (floating health bars, etc.). Extends `C_BaseClientUIEntity`.

| Offset | Type | Field | Notes |
|--------|------|-------|-------|
| 0x9E0 | CTransform | `m_anchorDeltaTransform` | 3D position (32 bytes) |
| 0xB70 | ptr | `m_pOffScreenIndicator` | Off-screen indicator |
| 0xB98 | bool | `m_bIgnoreInput` | Click-through |
| 0xB99 | bool | `m_bLit` | Lit by scene |
| 0xB9C | float32 | `m_flWidth` | Width |
| 0xBA0 | float32 | `m_flHeight` | Height |
| 0xBA4 | float32 | `m_flDPI` | DPI |
| 0xBE0 | bool | `m_bOpaque` | Opaque rendering |
| 0xBE1 | bool | `m_bNoDepth` | No depth testing |
| 0xBE4 | bool | `m_bUseOffScreenIndicator` | Off-screen indicator |
| 0xBE7 | bool | `m_bOnlyRenderToTexture` | Render-to-texture mode |

Subclasses: `CCitadelInWorldEventTimer`, `CCitadelUnitStatusStagger`, `CInWorldItemPanel`, `CPointOffScreenIndicatorUi`, `C_InWorldKeyBindPanel`, `C_PointClientUIWorldTextPanel`

#### CInfoOffscreenPanoramaTexture (size 0x7F8)

Renders a Panorama panel to a texture for in-world display:

| Offset | Type | Field |
|--------|------|-------|
| 0x5F0 | bool | `m_bDisabled` |
| 0x5F4 | int32 | `m_nResolutionX` |
| 0x5F8 | int32 | `m_nResolutionY` |
| 0x600 | CUtlSymbolLarge | `m_szPanelType` |
| 0x608 | CUtlSymbolLarge | `m_szLayoutFileName` |
| 0x610 | CUtlSymbolLarge | `m_RenderAttrName` |
| 0x618 | C_NetworkUtlVectorBase | `m_TargetEntities` |
| 0x638 | C_NetworkUtlVectorBase | `m_vecCSSClasses` |

---

## 6. Modifier & Ability VData → HUD

### How Modifiers Display in HUD

Every modifier has a `CCitadelModifierVData` (0x750 bytes) that configures its HUD appearance:

| Offset | Type | Field | Purpose |
|--------|------|-------|---------|
| 0x470 | ModifierOverheadDrawType_t | `m_eDrawOverheadStatus` | Overhead draw behavior |
| 0x474 | bool | `m_bReverseHudProgressBar` | Reverse progress bar |
| 0x478 | CUtlString | `m_strSmallIconCssClass` | CSS class for small icon |
| 0x480 | CUtlString | `m_strHintText` | Tooltip text |
| 0x488 | CUtlString | `m_strModifierOverrideStatusID` | Override status ID |
| **0x490** | **CPanoramaImageName** | **`m_strHudIcon`** | **Icon displayed in HUD (16 bytes)** |
| **0x4A0** | **HudDisplayLocation_t** | **`m_eHudDisplayLocation`** | **Where in HUD (enum, 4 bytes)** |
| 0x4A4 | ModifiersDisplayLocation_t | `m_eModifierDisplayLocaiton` | Alternative location |
| 0x4A8 | CUtlString | `m_strHudMessageText` | Message text |
| 0x4B0 | bool | `m_bIsHiddenOverhead` | Hide from overhead |
| 0x4B8 | CUtlVector | `m_vecAlwaysShowInStatModifierUI` | Always show in stats |

**Modifier → HUD pipeline:**
```
Modifier applied to entity
    ↓
CCitadelModifierVData loaded
    ↓
m_strHudIcon (0x490) → Panorama renders icon
    ↓
m_eHudDisplayLocation (0x4A0) → Positioned in HUD
    ↓
m_strHudMessageText (0x4A8) → Optional message text
    ↓
m_OnCreateResponse (0x4D0) → Particles, sounds, camera effects
```

#### Modifier Response System

`CCitadelModifierResponseRules_t` at VData offset 0x4D0 (56 bytes) — triggers effects on modifier creation:

| Offset | Type | Field | Size |
|--------|------|-------|------|
| 0x508 | CitadelCameraOperationsSequence_t | `m_cameraSequenceCreated` | 136 |
| 0x598 | CitadelCameraOperationsSequence_t | `m_cameraSequenceRemoved` | 136 |

### How Abilities Display in HUD

`CitadelAbilityVData` (0x1818 bytes) configures ability HUD:

| Offset | Type | Field | Purpose |
|--------|------|-------|---------|
| 0xF40 | CUtlString | `m_strCSSClass` | CSS class for styling |
| 0xF48 | CPanoramaImageName | `m_strAbilityImage` | Ability icon (16 bytes) |
| **0xF60** | **CitadelAbilityHUDPanel_t** | **`m_HUDPanel`** | **HUD panel config (56 bytes)** |
| 0xF98 | bool | `m_bShowInPassiveItemsArea` | Show in passives |
| 0xF99 | bool | `m_bForceHideHUDPanel` | Force hide |
| 0xF9A | bool | `m_bForceShowHUDPanel` | Force show |
| 0x1630 | CResourceNameTyped | `m_HudSharedStyle` | Panorama CSS resource (224 bytes) |

#### CitadelAbilityHUDPanel_t Structure (56 bytes)

| Offset | Type | Field | Size |
|--------|------|-------|------|
| 0x00 | CUtlVector | `m_vecHUDElements` | 24 (array of CitadelAbilityHUDElement_t) |
| 0x18 | CUtlVector | `m_vecButtonHints` | 24 (array of CitadelAbilityHUDElementButtonHint_t) |
| 0x30 | bool | `m_bForceDrawDefaultCastBars` | 1 |

#### CitadelAbilityHUDElement_t (272 bytes)

| Offset | Type | Field |
|--------|------|-------|
| 0x00 | ECitadelAbilityHUDElementType_t | `m_eType` |
| 0x08 | CUtlString | `m_strContext` |
| 0x18 | CUtlString | `m_strAdditionalClasses` |
| 0x20 | CUtlString | `m_Layout` |
| 0x28 | CResourceNameTyped | `m_Style` (224 bytes, Panorama CSS) |
| 0x108 | bool | `m_bReverseProgress` |
| 0x109 | bool | `m_bShowStacksOnProgress` |

### Custom HUD Display Interface

`ICitadelModifierCustomHudDisplay` (8 bytes) — interface for modifiers that need custom rendering. Implemented by specific modifiers to override default HUD behavior.

### Player HUD Visibility

`C_BasePlayerPawn` has `m_iHideHUD` at offset 0xFB0 (uint32) — bitfield controlling which HUD elements are visible. Inherited by `C_CitadelPlayerPawn`.

---

## 7. Panel Engine Internals

### CUIEngine (panorama.dll) — 147 vtable functions

The central engine object. Size ~0xB58 bytes.

#### Subsystem Pointers

| Offset | Purpose | Vtable Access |
|--------|---------|---------------|
| 0x0120 | System manager | idx_71 (ref) |
| 0x0268 | Streaming/callback system | idx_17 (read) |
| 0x0270 | Event dispatch system | idx_21 (read) |
| 0x0480 | Manager reference | idx_116 (ref) |
| 0x04C0 | Another system reference | — |
| 0x0590 | Active game state | idx_20 (read/write) |
| 0x05D0 | Active panel manager | — |
| 0x0600 | Shared resource cache | — |
| 0x0630 | Animation/timing engine | — |
| 0x0680 | Render subsystem | — |
| 0x06A8 | Input subsystem | — |
| 0x06D0 | File/resource loader | — |
| **0x0720** | **JavaScript runtime (V8)** | — |
| 0x0778 | Dialog/window manager | — |
| 0x07C8 | Style system | — |
| 0x07F0 | Layout engine | — |
| 0x0880 | Audio/animation system | idx_22 (read) |

#### Key Timing Fields

| Offset | Type | Purpose |
|--------|------|---------|
| 0x01C0 | float64 | Frame time / animation clock |
| 0x01C8 | float64 | Delta time |
| 0x01E0 | float64 | Mutable time value (writable) |

#### Panel Iteration

CUIEngine iterates panels via a linked list at `+0x0180` with count at `+0x0190`:

```
Signature (idx_7):
48 89 5C 24 08 48 89 74 24 10 57 48 83 EC 20 33 DB 48 8B F1
39 99 90 01 00 00 7E 2F 8B FB 66 90 48 8B 86 98 01 00 00 ...
```

Stride: 8 bytes per panel pointer.

### CUIPanel (panorama.dll) — 351 vtable functions

The base panel class. Size ~0x2A8 bytes. Every visible element inherits from this.

#### Core Field Layout

| Offset | Size | Purpose |
|--------|------|---------|
| 0x0000 | 8 | Vtable pointer |
| 0x0008 | 8 | Next sibling / linked list |
| 0x0010 | 8 | Previous sibling |
| 0x0018 | 8 | Associated layout template |
| 0x0020 | 8 | Metadata pointer |
| **0x0028** | **8** | **Script instance / V8 JS context** |
| 0x0030 | 8 | Event handler table |
| 0x0040 | 8 | Animation system |
| 0x0058 | 4 | Opacity (float) |
| 0x005C | 4 | Z-index / layer |
| 0x0060 | 8 | State flags |
| 0x0068 | 8 | Physics/bounds structure |

#### State Flags

| Offset | Size | Purpose |
|--------|------|---------|
| 0x0114 | 1 | Visibility flag |
| 0x0115 | 1 | Interactive flag |
| 0x0116 | 1 | Dormancy flag |
| 0x0117 | 1 | Rendering control |
| 0x0118 | 1 | Type flag |

#### Layout & Transform

| Offset | Size | Purpose |
|--------|------|---------|
| 0x0124-0x012C | 3×4 | Position floats (X, Y, computed) |
| 0x0198-0x01AC | 4×4 | Bounding box (L, T, R, B) |
| 0x01A8-0x01BC | 4×4 | Computed bounds |
| 0x01D8 | 8 | Animation time (float64) |

#### Panel Tree

| Offset | Size | Purpose |
|--------|------|---------|
| 0x0178 | 8 | Child count |
| 0x0180 | 8 | First child pointer |
| 0x0188 | 8 | Last child |
| 0x0190 | 8 | Parent panel |

#### Style & Events

| Offset | Size | Purpose |
|--------|------|---------|
| 0x0140 | 8 | Style instance (CPanelStyle*) |
| 0x01E8 | 8 | Style class list |
| 0x01F0 | 8 | Applied styles |
| 0x0208 | 2 | Event mask/flags |
| 0x020A | 2 | Event state |

#### Vtable Function Categories

| Index Range | Category |
|-------------|----------|
| 0-30 | Initialization, creation, destruction |
| 31-70 | Property getters/setters (visibility, position, style) |
| 71-130 | Layout and rendering |
| 131-200 | Event handling and callbacks |
| 201-280 | Scripting / JavaScript binding |
| 281-350 | Animation, transforms, effects |

### Widget Classes (panoramauiclient.dll)

All inherit from CUIPanel's vtable. Each has 84 vtable functions (62 analyzed).

| Class | Size | Key Fields |
|-------|------|------------|
| `CLabel` | ~0xE0 | Text at 0x80, 0xD0; length at 0x98; style at 0xE0 |
| `CButton` | ~0x18 | Minimal — behavior via vtable |
| `CImagePanel` | ~0xD0 | Image format at 0x34; data at 0x40, 0x98; scale at 0x58 |
| `CPanel2D` | ~0x18 | Minimal — 2D container |
| `CDropDown` | — | Dropdown selection |
| `CCheckBox` | — | Toggle switch |
| `CSlider` | — | Value slider |
| `CTextEntry` | — | Text input |
| `CMoviePanel` | — | Video playback |
| `CCarousel` | — | Scrollable carousel |
| `CContextMenu` | — | Right-click menus |
| `CConsole` | — | Debug console |
| `CDebugger` | — | UI debugger |

### CTopLevelWindow (panorama.dll) — 78 vtable functions

Root window that holds the panel tree:

| Offset | Size | Purpose |
|--------|------|---------|
| 0x0028-0x0034 | 4×4 | X, Y, Width, Height |
| 0x0048 | 8 | Parent window |
| 0x0060 | 4 | Scale X (float) |
| 0x0064 | 4 | Scale Y (float) |
| 0x0068 | 4 | Opacity (float) |
| 0x0080 | 8 | Root CUIPanel pointer |
| 0x0088 | 8 | Input handler (CUIWindowInput*) |
| 0x0090 | 2 | Window ID |
| 0x00F8 | 4 | Backbuffer width |
| 0x0108 | 4 | Backbuffer height |

### CPanelStyle (panorama.dll) — 156 vtable functions

Per-panel style object:

| Offset | Purpose |
|--------|---------|
| 0x0008 | Style flags |
| 0x0010 | Inherited styles reference |
| 0x0068 | Style property map |
| 0x0070 | Animation properties |
| 0x0088 | Transform matrix (float array) |

### CLayoutManager (panorama.dll) — 8 vtable functions

Manages loading XML layout files:

| Offset | Purpose |
|--------|---------|
| 0x00A0 | Layout file cache |
| 0x00AC | Cache size/count |
| 0x00BC | Active layout index |

### CGameUIService (engine2.dll) — 40 vtable functions

Bridge between engine and Panorama:

| Index | Purpose |
|-------|---------|
| idx_12, idx_18 | Write field at +0x0028 |
| idx_25 | Game time access |
| idx_29 | Float read at +0x0218 (GameTime) |
| idx_31, idx_32 | Entity/state access |

---

## 8. Event System

### Game Event Listener Chain

C_CitadelGameRules inherits `CGameEventListener` → `IGameEventListener2` → `IEntityListener`. It stores event IDs for dispatch:

| Offset | Field | Purpose |
|--------|-------|---------|
| 0x9EDC | `m_nPlayerDeathEventID` | Player death events |
| 0x9EE0 | `m_nReplayChangedEvent` | Replay/camera changes |
| 0x9EE4 | `m_nGameOverEvent` | Match conclusion |

### Panorama Event System

`CUIEvent` is a templated event class in `panorama.dll`. Variants include:

| Template | Parameter Type | Signature Pattern |
|----------|---------------|-------------------|
| `CUIEvent<void>` | No params | `CUIEvent@$00$$V@panorama` |
| `CUIEvent<int>` | Integer | `CUIEvent@$00H@panorama` |
| `CUIEvent<const char*>` | String | `CUIEvent@$00PEBD@panorama` |

Each event type has 4 vtable functions: construct, copy, fire, cleanup.

### Custom Entity Outputs

`C_BaseClientUIEntity` has 10 custom output slots (`m_CustomOutput0` through `m_CustomOutput9`), each a `CEntityOutputTemplate<CUtlString>` (32 bytes). These fire events from the entity system into Panorama panels.

### Panel Event Flow

```
Game Event (e.g., player death)
    ↓
CGameEventListener::FireGameEvent()
    ↓
Game C++ code
    ↓
SetDialogVariable() on panel    OR    CUIEvent fired
    ↓                                      ↓
Panel XML {d:var} updates            JS event handler runs
    ↓                                      ↓
Visual update                        JS updates panel
```

---

## 9. JavaScript V8 Integration

### V8 Engine Components

Located in `v8.dll`, accessed through `panorama.dll`:

#### ActualScript (v8_inspector) — 23 vtable functions

Script context wrapper:

| Offset | Size | Purpose |
|--------|------|---------|
| 0x0060 | 8 | Script content/source |
| 0x0098 | 8 | V8 Isolate/context pointer |
| 0x00C0 | 4 | Script ID |
| 0x00C4 | 1 | Is module flag |
| 0x00C5 | 1 | Is compiled flag |
| 0x00F0 | 4 | Line count |
| 0x00F4 | 4 | Column count |
| 0x00F8 | 4 | Bytecode size |
| 0x0100 | 8 | Compiled code pointer |
| 0x0108 | 8 | Debug symbol table |

#### FunctionMirror / ObjectMirror (v8_inspector)

- `FunctionMirror`: 7 vtable functions, field at +0x08 = function pointer
- `ObjectMirror`: 7 vtable functions, field at +0x08 = object ref, +0x10 = property table

### JS → C++ Binding Model

CUIPanel field at offset `+0x0028` holds the script instance / V8 context for that panel. The engine registers C++ functions as callable JS methods at initialization through `CUIEngine+0x0720` (JavaScript runtime subsystem).

Known API pattern (from Dota 2 / Source 2 documentation):

```javascript
// Panel manipulation
var panel = $.GetContextPanel();
panel.SetDialogVariable("health", 150);
var hp = panel.GetDialogVariable("health");

// Game API (registered C++ bindings)
var time = Game.GetGameTime();
var state = Game.GetState();
var localPlayer = Players.GetLocalPlayer();
var health = Entities.GetHealth(entityIndex);

// Events
GameEvents.Subscribe("player_death", function(data) {
    // handle death event
});

// Custom events
$.DispatchEvent("CustomEvent", panel, arg1, arg2);

// Schedule
$.Schedule(1.0, function() {
    // runs after 1 second
});
```

### Panorama Debugger

Built-in debugger accessible via `CDebugger@panorama` (84 vtable functions):

| Offset | Purpose |
|--------|---------|
| 0x0008 | Debugger state |
| 0x0010 | UI reference |
| 0x0020 | Breakpoint data |
| 0x0028 | Script context |
| 0x002C | Breakpoint count |

`CPanoramaClientDebugger` wraps this with a 4-function vtable, field at +0x08 = debugger instance pointer.

### CJSFile (panorama.dll)

Manages compiled JavaScript files loaded from VPK/filesystem. Part of `CLayoutManager`'s resource loading pipeline.

---

## 10. Hooking Strategies

### Strategy 1: Hook SetDialogVariable (Recommended — captures all HUD data)

`SetDialogVariable` is the single funnel where C++ game state enters Panorama. Hook it to capture every value being set on every HUD panel.

**Finding it:**
1. Search for strings like `"health"`, `"ammo"`, `"game_time"` in `client.dll`
2. Xref those strings — they're passed as first arg to `SetDialogVariable`
3. The function signature: `void SetDialogVariable(const char* name, <type> value)`
4. It's likely a CUIPanel vtable function in the **idx 266-278 range**

**With ARC Probe:**
```bash
# Find "health" string in client.dll
probe.exe "strings find health client.dll"

# Find xrefs to that string
probe.exe "strings xref <addr>"

# Disassemble around xref — look for the call
probe.exe "disasm <xref_addr> 20"

# The function being called with "health" as arg is SetDialogVariable
# Hook it:
probe.exe "hook detour <setdialogvar_addr> SetDialogVariable"

# Wait for game to run, then read captures:
probe.exe "hook log SetDialogVariable"
```

### Strategy 2: Hook CUIEngine Time Functions

CUIEngine vtable functions that read timing:

| Vtable Index | Signature | Reads |
|--------------|-----------|-------|
| idx_8 | `F2 0F 10 81 C0 01` | Float at +0x1C0 (frame time) |
| idx_61 | `F2 0F 10 81 E0 01` | Float at +0x1E0 (mutable time) |

```bash
# Find CUIEngine vtable via RTTI
probe.exe "rtti find CUIEngine"
probe.exe "rtti vtable CUIEngine"

# Read vtable entry at index 8 (8*8 = 0x40 offset)
probe.exe "read_ptr <vtable>+0x40"

# Hook that function
probe.exe "hook detour <func_addr> UIEngine_GetTime"
```

### Strategy 3: Direct Memory Read (No hooking needed)

Read game state directly from entity memory. No hooks, no risk of crashing.

**Match time:**
```bash
# Find game rules proxy
# (scan entities for "citadel_game_rules" designer name)
probe.exe "read_ptr <proxy>+0x5F0"        # → C_CitadelGameRules*
probe.exe "read_float <rules>+0x9E3C"     # → match clock (seconds)
probe.exe "read_int <rules>+0x74"         # → EGameState enum
```

**Player health:**
```bash
probe.exe "read_int <pawn>+0x354"          # m_iHealth
probe.exe "read_int <pawn>+0x350"          # m_iMaxHealth
```

**Ability cooldowns:**
```bash
probe.exe "read_float <ability>+0x764"     # m_flCooldownStart
probe.exe "read_float <ability>+0x768"     # m_flCooldownEnd
probe.exe "read_int <ability>+0x780"       # m_iRemainingCharges
```

### Strategy 4: Hook Protobuf Message Dispatch

Intercept `CCitadelUserMsg_HudGameAnnouncement` messages to catch server-driven HUD updates. These contain `dialog_variable_name` and `dialog_variable_locstring` fields.

Find the message handler by searching for the protobuf message name string, then xref to the dispatch function.

### Strategy 5: Hook MNetworkChangeCallback

Fields with `[MNetworkChangeCallback]` trigger C++ callbacks when their values change over the network. Hook the callback function to intercept state changes at the moment they arrive.

Key targets:
- `C_CitadelGameRules.m_eGameState` — fires on every game phase change
- `CCitadelAbilityComponent.m_hSelectedAbility` — ability selection
- `CCitadelAbilityComponent.m_flTimeScale` — haste/slow changes

### Strategy 6: Inject JavaScript

If you can get a reference to the V8 context (CUIPanel offset `+0x0028`), you could inject JavaScript into the Panorama runtime. This would let you:
- Call `Game.GetGameTime()` and read the return value
- Subscribe to `GameEvents` and log them
- Read/write dialog variables on any panel
- Execute arbitrary panel manipulation

This is the most powerful approach but also the most complex.

---

## 11. Practical Guide: Finding Things with ARC Probe

### Finding Panorama Panel Instances in Memory

Panorama panels are **not** in the entity list. They live in `panorama.dll`'s internal panel tree. To find them:

```bash
# 1. Find CUIEngine singleton via RTTI
probe.exe "rtti find CUIEngine"
probe.exe "rtti scan panorama.dll"

# 2. Get the window manager
probe.exe "read_ptr <engine>+0x0778"  # Dialog/window manager

# 3. Get the root panel from a window
probe.exe "read_ptr <window>+0x0080"  # Root CUIPanel

# 4. Walk the child tree
probe.exe "read_ptr <panel>+0x0180"   # First child
probe.exe "read_ptr <child>+0x0008"   # Next sibling
```

### Finding Game-Specific HUD Entities

These **are** in the entity list:

```bash
# Find C_PointClientUIHUD entities
probe.exe "rtti find PointClientUIHUD"

# Or scan entity list for them
# They have designer names and m_DialogXMLName fields

# Read the XML layout file path
probe.exe "read_ptr <entity>+0x9B0"  # m_DialogXMLName (CUtlSymbolLarge)
probe.exe "read_string <symbol_ptr>"  # Follow to get the string

# Read the panel class name
probe.exe "read_ptr <entity>+0x9B8"  # m_PanelClassName
```

### Finding SetDialogVariable

```bash
# Method 1: String search for known variable names
probe.exe "strings find health client.dll"
probe.exe "strings xref <health_string_addr>"
# Disassemble around xref to find the call pattern

# Method 2: RTTI vtable analysis
probe.exe "rtti find CUIPanel"
probe.exe "rtti vtable CUIPanel"
# SetDialogVariable is in vtable idx ~266-278 range
# Read those vtable entries and disassemble them

# Method 3: Pattern scan for the signature
# SetDialogVariable typically takes (this, const char* name, value)
# Look for: mov rcx, <panel_ptr>; lea rdx, <string>; call <func>
```

### Finding Game Rules for Match Time

```bash
# Method 1: Entity scan for game rules proxy
# The proxy entity has designer name "citadel_game_rules"
# Scan entity list, check designer name at identity+0x20

# Method 2: Pattern scan in client.dll
probe.exe "pattern 48 8B 05 ?? ?? ?? ?? 48 85 C0 74 ?? 80 B8 client.dll"
# Look for patterns that access a singleton and then check offset 0x74 (m_eGameState)

# Once found:
probe.exe "read_ptr <proxy>+0x5F0"       # C_CitadelGameRules*
probe.exe "read_float <rules>+0x9E3C"    # Match clock
probe.exe "read_int <rules>+0x74"        # Game state
probe.exe "read_int <rules>+0xBC"        # Winning team
```

### Monitoring HUD Updates in Real-Time

```bash
# Set a hardware breakpoint on a dialog variable being written
# First, find where a specific variable value is stored in a panel

# 1. Find a CUIPanel instance
# 2. Look at field +0x0238 region (dialog variable data)
# 3. Set hardware write watchpoint
probe.exe "hwbp set <addr> w 4"

# 4. Wait for the HUD to update
# 5. Read the breakpoint log
probe.exe "bp log <bp_id>"
# This gives you RIP — the exact instruction writing the dialog variable

# 6. Disassemble the writer
probe.exe "disasm_function <rip>"
# Now you see the full C++ function that sets this HUD value
```

### Dumping All Panorama Strings

```bash
# Find all UI-related strings in panorama.dll
probe.exe "strings scan panorama.dll"
# Look for: property names, CSS classes, layout file paths, event names

# Find strings in panoramauiclient.dll
probe.exe "strings scan panoramauiclient.dll"

# Find game-specific HUD strings in client.dll
probe.exe "strings find Citadel_Hud client.dll"
probe.exe "strings find dialog_variable client.dll"
probe.exe "strings find GameTime client.dll"
```

---

## 12. Reference Tables

### GameTime_t

A 4-byte float representing game simulation time in seconds. Not real-world time — it pauses when the server pauses. All timing comparisons in Panorama use this clock.

To calculate remaining cooldown:
```
remaining = m_flCooldownEnd - current_game_time
```

### CPanoramaImageName

16-byte structure storing a Panorama image resource path. Used for ability icons, modifier icons, hero portraits. Points to textures loaded from VPK.

Found in:
- `CCitadelModifierVData.m_strHudIcon` (0x490)
- `CitadelAbilityVData.m_strAbilityImage` (0xF48)
- `CitadelHeroData_t.m_strIconImageSmall` (0x3C0)
- `CitadelHeroData_t.m_strIconHeroCard` (0x3D0)

### CUtlSymbolLarge

8-byte pointer to a symbol table entry. Used for XML layout paths, panel class names, CSS class names. Dereference the pointer and read the string at the target.

### Panorama Resource Types

| Resource Type | Purpose |
|---------------|---------|
| `InfoForResourceTypeCPanoramaLayout` | Layout XML files |
| `InfoForResourceTypeCPanoramaStyle` | CSS stylesheet files |
| `InfoForResourceTypeCPanoramaDynamicImages` | Dynamic image resources |

### Panel Hierarchy Memory Map

```
CUIEngine (0xB58+ bytes)
  ├── 0x0720 → JavaScript Runtime (V8)
  ├── 0x0778 → Dialog/Window Manager
  │   └── CTopLevelWindow instances
  │       └── 0x0080 → Root CUIPanel
  │           └── 0x0180 → First child (linked list via 0x0008)
  ├── 0x07C8 → Style System
  │   └── CPanelStyle instances (per panel, at panel+0x0140)
  ├── 0x07F0 → Layout Engine
  │   └── CLayoutManager → CLayoutFile → CLayoutContent
  ├── 0x06A8 → Input Subsystem
  │   └── CUIInputEngine → CUIWindowInput (per window)
  └── 0x0680 → Render Subsystem
```

### HUD Visibility Bitfield (m_iHideHUD)

Located at `C_BasePlayerPawn+0xFB0` (uint32). Each bit controls visibility of a HUD element group. Setting bits hides the corresponding group.

### Key Panorama.dll Signatures

```
CUIEngine vtable access patterns:
  idx_17:  48 8B 81 68 02 00 00    # read +0x268 (streaming/callback)
  idx_18:  48 8B 81 D8 04 00 00    # read +0x4D8 (view matrix related)
  idx_20:  48 8B 81 90 05          # read +0x590 (game state)
  idx_21:  48 8B 81 70 02 00       # read +0x270 (event dispatch)
  idx_22:  48 8B 81 80 08 00       # read +0x880 (audio/anim)

Panel iteration:
  48 89 5C 24 08 48 89 74 24 10 57 48 83 EC 20 33 DB 48 8B F1
  39 99 90 01 00 00 7E 2F ...

Time reads:
  F2 0F 10 81 C0 01  # movsd xmm0, [rcx+0x1C0] (frame time)
  F2 0F 10 81 E0 01  # movsd xmm0, [rcx+0x1E0] (mutable time)
```

### CUIWindowInput Key Fields

| Offset | Purpose |
|--------|---------|
| 0x0018 | Focus panel ptr |
| 0x0020 | Hover panel ptr |
| 0x002A | Mouse button state |
| 0x0048 | Mouse X |
| 0x0058 | Mouse Y |
| 0x0068 | Key code |

### Source 2 Interface System

Game-specific interfaces exposed through `panorama.dll` and `client.dll`:

```bash
# List all Source 2 interfaces
probe.exe "interfaces list"

# Dump a specific interface vtable
probe.exe "interfaces vtable Source2Client002"
```

---

## Appendix A: Common Dialog Variable Names

Based on analysis of Source 2 games and Deadlock HUD panel names, likely dialog variable names include:

| Variable | Type | Source |
|----------|------|--------|
| `health` | int | C_BaseEntity.m_iHealth |
| `max_health` | int | C_BaseEntity.m_iMaxHealth |
| `health_pct` | float | Computed |
| `ammo` | int | C_CitadelBaseAbility.m_iClip |
| `game_time` | float | C_CitadelGameRules.m_flMatchClockAtLastUpdate |
| `game_state` | int | C_CitadelGameRules.m_eGameState |
| `team` | int | C_BaseEntity.m_iTeamNum |
| `player_name` | string | CBasePlayerController.m_iszPlayerName |
| `hero_name` | string | Hero data |
| `level` | int | C_CitadelPlayerPawn.m_nLevel |
| `cooldown_pct` | float | Computed from ability times |
| `charges` | int | C_CitadelBaseAbility.m_iRemainingCharges |
| `gold` / `souls` | int | Player resource |

**Note:** These are educated guesses. To discover the actual names, hook `SetDialogVariable` or use the `CDebugPanelDialogVariables` debug panel.

---

## Appendix B: Cross-References to Other Docs

| Topic | Document |
|-------|----------|
| Entity system structure | `CLAUDE.md` (standalone) — Entity List Structure section |
| Ability offsets | `CLAUDE.md` (standalone) — Key Ability Offsets section |
| Debug server protocol | `docs/debug-server.md` |
| ARC Probe commands | `docs/openapi.yaml` |
| Deadlock entity catalog | `docs/deadlock-entities.md` |

---

## Appendix C: Schema Dump File Locations

All data in this document was sourced from the dezlock-dump:

```
E:\WEB_PROJECTS\ARCRAIDERS\dezlock-dump\build\bin\Release\schema-dump\deadlock\
├── client.txt                          # Client-side class definitions (21.5 MB)
├── server.txt                          # Server-side class definitions (24.3 MB)
├── sdk/
│   ├── panorama/_rtti/                 # 194 Panorama engine RTTI files
│   ├── panoramauiclient/_rtti/         # 229 Panorama UI widget RTTI files
│   ├── panorama_text_pango/_rtti/      # Text rendering RTTI
│   ├── client/_rtti/                   # 22+ CitadelHud panel RTTI files
│   ├── client/_protobuf/              # HUD protobuf messages
│   ├── engine2/_rtti/                  # CGameUIService RTTI
│   └── v8/_rtti/                       # V8 JavaScript engine RTTI
├── signatures/
│   ├── panorama.txt                    # 9000+ function signatures (840 KB)
│   ├── client.txt                      # Client function signatures
│   └── engine2.txt                     # Engine function signatures
└── hierarchy/                          # Entity inheritance chains
```
