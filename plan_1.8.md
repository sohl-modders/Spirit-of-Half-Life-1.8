# Integration Plan: SOHL 1.8 Features into SOHL 1.5

This document provides a structured plan for integrating Spirit of Half-Life 1.8 changes
into a SOHL 1.5 codebase. Changes are organized into independent phases by priority,
dependency order, and risk level.

---

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Phase 1: Foundation & Build System](#phase-1-foundation--build-system)
- [Phase 2: Core Entity System Updates](#phase-2-core-entity-system-updates)
- [Phase 3: Weapon System Overhaul](#phase-3-weapon-system-overhaul)
- [Phase 4: Locus Calculation System](#phase-4-locus-calculation-system)
- [Phase 5: View, Camera & Rendering](#phase-5-view-camera--rendering)
- [Phase 6: Trigger & Game Logic](#phase-6-trigger--game-logic)
- [Phase 7: Inventory System](#phase-7-inventory-system)
- [Phase 8: HUD & Client Features](#phase-8-hud--client-features)
- [Phase 9: AI & Scripted Sequences](#phase-9-ai--scripted-sequences)
- [Phase 10: Assets & Documentation](#phase-10-assets--documentation)
- [Risk Assessment](#risk-assessment)
- [File Change Reference](#file-change-reference)

---

## Overview

SOHL 1.8 introduces approximately **59,000 lines of additions** and **4,700 lines of
deletions** across **255 files** compared to SOHL 1.6.

Since SOHL 1.5 predates SOHL 1.6, the actual diff against 1.5 will be even larger.
The changes between 1.5 and 1.6 must also be accounted for. This plan assumes those
1.5 → 1.6 changes are already in your codebase or will be handled separately.

### Key Architectural Changes
1. **Client-side weapon prediction** — Requires restructuring all weapon classes
2. **Locus system rewrite** — API signature changes affect many files
3. **Alias system rename** — `CBaseAlias` → `CBaseMutableAlias` (global rename)
4. **Decal system reversion** — Custom event system removed in favor of standard HL methods
5. **Inventory system** — Entirely new feature spanning server, client, and HUD

### Integration Approach
- **Merge in phases** — Each phase can be compiled and tested independently
- **Test after each phase** — Run the mod and verify no regressions
- **Keep 1.5-specific features** — Don't blindly overwrite; merge carefully

---

## Prerequisites

Before starting integration:

1. **Back up your SOHL 1.5 codebase** completely
2. **Set up version control** (Git recommended) with your 1.5 code as the base commit
3. **Ensure your 1.5 builds** — Verify clean compile of both client and server DLLs
4. **Identify your 1.5 customizations** — List all changes you've made to SOHL 1.5
   so they can be preserved during merge
5. **Set up VS .NET** (Visual Studio 2003+) — SOHL 1.8 adds `.sln`/`.vcproj` files
   alongside the existing `.dsp`/`.dsw` files

---

## Phase 1: Foundation & Build System

**Risk**: Low | **Dependencies**: None | **Estimated Effort**: 1–2 hours

### Tasks

- [ ] **1.1** Add OpenGL headers (`common/gl/gl.h`, `glext.h`, `glu.h`)
  - Copy the 3 files to `SpiritSource/common/gl/`
  - These are standard headers with no code changes needed

- [ ] **1.2** Add `cl_dll/glInclude.h` wrapper
  - Single 30-line header replacing scattered OpenGL includes

- [ ] **1.3** Add NVIDIA Cg headers (`common/cg/*.h`)
  - Copy all 12 Cg header files to `SpiritSource/common/cg/`
  - Required for future shader support, no code changes needed

- [ ] **1.4** Update project files
  - Add new source files to `hl.dsp` and `cl_dll.dsp`
  - Optionally add `.sln`/`.vcproj` files for VS .NET support

- [ ] **1.5** Update `SpiritSource/hl.dsw` workspace
  - Add combined workspace file if not present

- [ ] **1.6** Verify clean build after header additions

### Files to Add
```
SpiritSource/common/gl/gl.h
SpiritSource/common/gl/glext.h
SpiritSource/common/gl/glu.h
SpiritSource/common/cg/*.h (12 files)
SpiritSource/cl_dll/glInclude.h
```

---

## Phase 2: Core Entity System Updates

**Risk**: Medium | **Dependencies**: Phase 1 | **Estimated Effort**: 4–6 hours

### Tasks

- [ ] **2.1** Rename alias system: `CBaseAlias` → `CBaseMutableAlias`
  - Files: `cbase.h`, `cbase.cpp`, `alias.cpp`, `util.cpp`, `util.h`
  - Global find/replace: `CBaseAlias` → `CBaseMutableAlias`, `IsAlias()` → `IsMutableAlias()`
  - Add `FollowAlias()` to `CBaseEntity`

- [ ] **2.2** Add `USE_SPAWN = 7` use type
  - File: `cbase.h` — add enum value
  - File: `subs.cpp` — add `&` prefix handling in `FireTargets()`

- [ ] **2.3** Update `TakeHealth()` return value
  - File: `cbase.cpp` — return actual health given instead of 0/1

- [ ] **2.4** Add CBaseToggle acceleration system
  - File: `cbase.h` — add `m_flLinearAccel`, `m_flLinearDecel`, `m_flCurrentTime`,
    `m_flAccelTime`, `m_flDecelTime`, `m_bDecelerate` fields
  - File: `subs.cpp` — add `LinearMove()` overload with acceleration/deceleration
  - Define `ACCELTIMEINCREMENT = 0.1`

- [ ] **2.5** Add custom death notice fields
  - File: `cbase.h` — add `killname`, `killmethod` string_t fields

- [ ] **2.6** Remove `UTIL_AutoSetSize()`
  - File: `cbase.h` — remove declaration
  - Update callers to use explicit `UTIL_SetSize()`

- [ ] **2.7** Update `RADIATION_DURATION` (2 → 10)

- [ ] **2.8** Add `g_allowGJump` global

- [ ] **2.9** Remove `pev->colormap = ENTINDEX(pent)` from DispatchSpawn/DispatchRestore

- [ ] **2.10** Build and test — Verify all existing entities still work

### Key Risk
The alias rename (`CBaseAlias` → `CBaseMutableAlias`) affects multiple files. Use IDE
refactoring tools or careful find/replace to avoid missing instances.

---

## Phase 3: Weapon System Overhaul

**Risk**: High | **Dependencies**: Phase 2 | **Estimated Effort**: 8–16 hours

This is the largest and most complex change. Consider splitting into sub-phases.

### Phase 3a: Weapon Header Restructure

- [ ] **3a.1** Move weapon class declarations into `weapons.h`
  - Move `CGlock`, `CCrowbar`, `CPython`, `CMP5`, `CCrossbow`, `CShotgun`, `CRpg`,
    `CRpgRocket`, `CGauss`, `CEgon`, `CHgun`, `CHandGrenade`, `CSatchel`,
    `CSqueakGrenade`, `CTripmine` classes from their `.cpp` files to `weapons.h`
  - This is required for client-side prediction (Phase 3c)

- [ ] **3a.2** Update method signatures
  - `Holster()` → `Holster(int skiplocal = 0)` on all weapons
  - `SendWeaponAnim()`: add `int skiplocal = 1, int body = 0`
  - `DefaultDeploy()`: add `int skiplocal = 0, int body = 0`
  - `DefaultReload()`: add `int body = 0`

- [ ] **3a.3** Add `UseDecrement()` virtual method
  - Returns FALSE by default, TRUE when `CLIENT_WEAPONS` defined

- [ ] **3a.4** Remove deprecated weapon fields
  - `m_flTimeUpdate`, `m_flChargeTime`, `m_flShockTime`, `m_iChargeLevel`,
    `m_iOverloadLevel`, `m_fInAttack`, `m_iLastSkin`, `m_iBody`, `m_iLastBody`,
    `b_Restored`, `AnimRestore`, `PlayerHasSuit`
  - Remove `RestoreBody()`, `SUIT`, `NUM_HANDS`

- [ ] **3a.5** Revert decal system
  - Replace all `PLAYBACK_EVENT_FULL(m_usDecals, ...)` with standard
    `UTIL_BloodDecalTrace()` / `DecalGunshot()` in `combat.cpp`
  - Remove `m_usDecals`, `m_usEfx` from `weapons.cpp`

### Phase 3b: Individual Weapon Updates

- [ ] **3b.1** Update each weapon's animation enums (see changelog for details per weapon)
- [ ] **3b.2** Add per-weapon `PRECACHE_EVENT` calls
- [ ] **3b.3** Add `AddToPlayer()` where missing (glock, python, etc.)
- [ ] **3b.4** Wrap client-incompatible entities in `#ifndef CLIENT_DLL`
  (crossbow bolt, snark, etc.)
- [ ] **3b.5** Add `weapon_generic.cpp` and `debugger.cpp`
- [ ] **3b.6** Update `client.cpp` `GetWeaponData()` for weapon state packing
- [ ] **3b.7** Build server DLL and test all weapons

### Phase 3c: Client-Side Weapon Prediction

- [ ] **3c.1** Add client prediction files:
  - `cl_dll/com_weapons.cpp` / `com_weapons.h`
  - `cl_dll/hl/hl_baseentity.cpp`
  - `cl_dll/hl/hl_events.cpp`
  - `cl_dll/hl/hl_objects.cpp`
  - `cl_dll/hl/hl_weapons.cpp`
- [ ] **3c.2** Restructure `events.cpp` → call `Game_HookEvents()`
- [ ] **3c.3** Update `ev_hldm.cpp`: remove old events, add new ones
- [ ] **3c.4** Update `ev_hldm.h`: correct weapon enums, add `generic_e`
- [ ] **3c.5** Update `entity.cpp`: `Game_AddObjects()` replaces `EV_UpdateBeams()`
- [ ] **3c.6** Build both DLLs and test weapon prediction

### Key Risk
This is the highest-risk phase. Weapon class restructuring can introduce subtle bugs
in animation, firing behavior, and multiplayer prediction. **Test each weapon thoroughly**
after completing this phase.

---

## Phase 4: Locus Calculation System

**Risk**: High | **Dependencies**: Phase 2 | **Estimated Effort**: 6–10 hours

### Tasks

- [ ] **4.1** Update locus function signatures (output-parameter style)
  - `CalcPosition()` → `bool CalcPosition(CBaseEntity*, Vector* OUTresult)`
  - `CalcVelocity()` → `bool CalcVelocity(CBaseEntity*, Vector* OUTresult)`
  - `CalcRatio()` → `bool CalcNumber(CBaseEntity*, float* OUTresult)`
  - Add `CalcPYR()` for pitch/yaw/roll

- [ ] **4.2** Global rename: `CalcLocus_Ratio` → `CalcLocus_Number`
  - Affects: `triggers.cpp`, `doors.cpp`, `effects.cpp`, `monsters.cpp`, `locus.cpp`,
    `basemonster.h`, and more

- [ ] **4.3** Add new locus parsing capabilities
  - `TryParseVectorComponentwise()`: `'x y z'` and random range support
  - `TryParseLocusBrackets()`: nested expressions
  - `TryParseLocusNumber()`: vector swizzle (x/y/z/p/r), PITCH/YAW/LENGTH
  - `CMark` class for runtime position markers

- [ ] **4.4** Update all callers to use output-parameter style
  - Every file that calls `CalcPosition`/`CalcVelocity`/`CalcRatio` must be updated

- [ ] **4.5** Update FGD: add new `calc_*` entities, rename old ones

- [ ] **4.6** Build and test locus calculations with demo maps

### Key Risk
The API signature change affects many files. Missing a caller will cause compile errors
(which is good — they're caught at build time). But logic errors in the output-parameter
migration could cause subtle runtime bugs in entity calculations.

---

## Phase 5: View, Camera & Rendering

**Risk**: Medium | **Dependencies**: Phase 1 | **Estimated Effort**: 4–8 hours

### Tasks

- [ ] **5.1** Update sky rendering system (`view.cpp`)
  - Add `SKY_ON_DRAWING = 2` state
  - Add parallax sky support (`m_iSkyScale`)
  - Fix sky/camera coexistence

- [ ] **5.2** Update fog system
  - Add `FogSettings` struct to `hud.h`
  - Rewrite `MsgFunc_SetFog` in `hud_msg.cpp`
  - Update `RenderFog()` in `tri.cpp` to use `pTriAPI->Fog()` API
  - Add `ClearToFogColor()` and `BlackFog()` functions
  - Add smooth fog fade transitions in `hud_redraw.cpp`

- [ ] **5.3** Add view clamping system (LRC 1.8)
  - `inputw32.cpp`: `V_ClampYaw()`, `V_ClampPitch()`, `V_LimitClampSpeed()`
  - `hud.h/cpp`: `MsgFunc_ClampView`
  - `player.cpp`: `gmsgClampView` registration

- [ ] **5.4** Update StudioModelRenderer
  - Remove custom shadow hack → standard `GL_StudioDrawShadow()`
  - Replace direct GL includes → `glInclude.h`
  - Add sky culling (`kRenderFxEntInPVS` check)

- [ ] **5.5** Update camera system
  - Remove `studiohdr_t` eye lookups
  - Add `drawplayer`/`hideplayer` commands
  - Rewrite `V_CalcThirdPersonRefdef`

- [ ] **5.6** Update `CEnvMirror` entity

- [ ] **5.7** Build both DLLs and test rendering

---

## Phase 6: Trigger & Game Logic

**Risk**: Medium | **Dependencies**: Phase 2, Phase 4 | **Estimated Effort**: 3–5 hours

### Tasks

- [ ] **6.1** Update `trigger_relay` (`CTriggerRelay`)
  - Add `SF_RELAY_DEBUG` (0x02), `SF_FIRE_CAMERA` (0x04) spawnflags
  - Add `m_iszAltTarget` field
  - Add `USE_SAME`/`USE_SET` handling
  - Remove hardcoded hacks (c1a4f elevator, c5a1 stars)

- [ ] **6.2** Update doors (`doors.cpp`)
  - Add acceleration keyvalues (`acceleration`, `deceleration`, `speedmode`)
  - Use new `LinearMove()` overload
  - Fix blocked damage attribution

- [ ] **6.3** Update platforms (`plats.cpp`)
  - Fix train sound restart
  - Remove `CFuncVehicle`
  - Add game-start wait

- [ ] **6.4** Update buttons (`buttons.cpp`)
  - Adjust spark volumes
  - Add cyclic spark mode (spawnflag 16)

- [ ] **6.5** Update effects (`effects.cpp`)
  - `CLightning` attachment points
  - `env_elight` unique keys
  - `CGibShooter` batch-fire mode
  - `CEnvFog` base class change
  - `CEnvSky` parallax scale
  - `CParticle` active state changes

- [ ] **6.6** Build and test trigger logic with demo maps

---

## Phase 7: Inventory System

**Risk**: Medium | **Dependencies**: Phase 2, Phase 5 (HUD messages) | **Estimated Effort**: 4–6 hours

### Tasks

- [ ] **7.1** Add `items.h` header with item constants and class declarations

- [ ] **7.2** Update `items.cpp`
  - Add `CItemMedicalKit` (portable medkit with charge system)
  - Extract `CItemAntidote` to named class
  - Add `CItemAntiRad`, `CItemFlare`, `CItemCamera`
  - Add `I_Precache()` function
  - Add inventory message sending

- [ ] **7.3** Update `player.h/cpp`
  - Add `m_rgItems[MAX_ITEMS]` and `m_pItemCamera`
  - Register `gmsgInventory`
  - Reset inventory on spawn
  - Add timed damage rework (nerve gas, radiation behind CVAR)

- [ ] **7.4** Update `client.cpp`
  - Add `"inventory"` client command handler

- [ ] **7.5** Update client HUD
  - `hud.h/cpp`: Add `MsgFunc_Inventory`, `g_iInventory[]`
  - `vgui_TeamFortressViewport.cpp/h`: Add `InventoryButton` class

- [ ] **7.6** Build and test inventory items

---

## Phase 8: HUD & Client Features

**Risk**: Low–Medium | **Dependencies**: Phase 5 | **Estimated Effort**: 3–5 hours

### Tasks

- [ ] **8.1** Add camera HUD overlay
  - Flashing camera icon + corner reticle brackets
  - Sprites: `320_camera.spr`, `640_camera.spr`, `640_camera_rect.spr`

- [ ] **8.2** Add custom menu system
  - `CustomMenu.cpp`: VGUI-based custom menu panel
  - `tf_defs.h`: `MENU_CUSTOM = 10`

- [ ] **8.3** Add glow effect CVARs (FragBait0)
  - `r_glow`, `r_glowstrength`, `r_glowblur`, `r_glowdark`

- [ ] **8.4** Update death messages (`death.cpp`)
  - `%` delimiter for custom kill technique strings

- [ ] **8.5** Remove deprecated HUD messages
  - `gmsgSetBody`, `gmsgSetSkin`, `gmsgSetMirror`, `gmsgResetMirror`

- [ ] **8.6** Fix particle system (`particlesys.cpp`)
  - Entity lookup: `gEngfuncs.GetEntityByIndex()` instead of colormap search
  - PVS check for off-screen particles

- [ ] **8.7** Update input bindings (`input.cpp`)
  - `+hud`/`-hud`, `+briefing`/`-briefing`
  - Remove dead-check from mouse input

- [ ] **8.8** Update FMOD (`mp3.cpp`)
  - Library: `mp3dec.dll` → `fmod.dll`
  - Enable loop mode
  - Update media path

---

## Phase 9: AI & Scripted Sequences

**Risk**: Medium | **Dependencies**: Phase 2 | **Estimated Effort**: 3–5 hours

### Tasks

- [ ] **9.1** Add `AI_BaseNPC_Schedule.cpp`
  - New NPC schedule framework (1,621 lines)
  - Add to project files

- [ ] **9.2** Update scripted sequences (`scripted.cpp/h`)
  - Replace interrupt enum with flag system
  - Update `CanPlaySequence()` signature
  - Add keyvalues: `m_fTurnType`, `m_iRepeats`, `m_iPriority`
  - Add `InitIdleThink()`, expanded `PossessEntity()`

- [ ] **9.3** Update monster base (`basemonster.h`, `monsters.cpp`)
  - `CalcRatio` → `CalcNumber`
  - Add `GetState()` method
  - Dead monsters return ratio 0

- [ ] **9.4** Update monster maker (`monstermaker.cpp`)

- [ ] **9.5** Build and test AI behaviors

---

## Phase 10: Assets & Documentation

**Risk**: Low | **Dependencies**: None (can run in parallel) | **Estimated Effort**: 1–2 hours

### Tasks

- [ ] **10.1** Copy game assets to `Spirit/` directory
  - Demo maps (13 BSP + 13 RMF)
  - Models: `null.mdl`, `w_camera.mdl`, `w_portablemed.mdl`, `w_portablemedt.mdl`
  - Sprites (17 files)
  - Aurora particle: `teleport.aur`
  - Menu graphics: `btns_main.bmp`, `head_*.BMP`, etc.
  - `spirit.ico`, `Run Spirit.bat`

- [ ] **10.2** Add `sentences.txt` (1,376 lines)

- [ ] **10.3** Add `hud.txt` and `weapon_debug.txt` sprite definitions

- [ ] **10.4** Copy documentation (`Spirit/docs/`)

- [ ] **10.5** Update `spirit18.fgd` with new entities

- [ ] **10.6** Update `liblist.gam`
  - Game name, URL, version, DLL path, start/train maps

- [ ] **10.7** Add event scripts (`events/generic1-3.sc`, `mirror.sc`)

---

## Risk Assessment

| Phase | Risk | Reason | Mitigation |
|-------|------|--------|------------|
| 1. Foundation | Low | Header additions only | Build test after each file |
| 2. Core Entity | Medium | Alias rename touches many files | Use IDE refactoring; build often |
| 3. Weapons | **High** | Class restructuring + prediction | Test each weapon individually |
| 4. Locus | **High** | API changes affect many callers | Build catches missing updates; test with demo maps |
| 5. View/Render | Medium | Rendering changes hard to debug | Compare visually with 1.8 reference |
| 6. Triggers | Medium | Logic changes in triggers | Test with demo maps |
| 7. Inventory | Medium | New feature spanning many systems | Test each item type |
| 8. HUD | Low–Medium | Mostly additive changes | Visual inspection |
| 9. AI | Medium | AI behavior changes | Test with monster maps |
| 10. Assets | Low | File copy only | Verify paths match code |

### Critical Path
**Phase 1** → **Phase 2** → **Phase 3** (weapons) and **Phase 4** (locus) in parallel
→ **Phase 5–9** (can largely proceed in parallel) → **Phase 10** (anytime)

---

## File Change Reference

### New Server Files (SpiritSource/dlls/)
| File | Lines | Phase |
|------|-------|-------|
| `AI_BaseNPC_Schedule.cpp` | 1,621 | 9 |
| `debugger.cpp` | 296 | 3b |
| `weapon_generic.cpp` | 235 | 3b |
| `items.h` | 79 | 7 |

### New Client Files (SpiritSource/cl_dll/)
| File | Lines | Phase |
|------|-------|-------|
| `CustomMenu.cpp` | 428 | 8 |
| `com_weapons.cpp` | 277 | 3c |
| `com_weapons.h` | 26 | 3c |
| `glInclude.h` | 30 | 1 |
| `hl/hl_baseentity.cpp` | 349 | 3c |
| `hl/hl_events.cpp` | 84 | 3c |
| `hl/hl_objects.cpp` | 90 | 3c |
| `hl/hl_weapons.cpp` | 1,101 | 3c |

### New Common Files (SpiritSource/common/)
| File | Lines | Phase |
|------|-------|-------|
| `gl/gl.h` | 1,526 | 1 |
| `gl/glext.h` | 3,081 | 1 |
| `gl/glu.h` | 584 | 1 |
| `cg/*.h` (12 files) | ~3,064 | 1 |

### Heavily Modified Server Files
| File | ±Lines | Phase |
|------|--------|-------|
| `weapons.h` | +706 | 3a |
| `weapons.cpp` | +341 | 3a |
| `locus.cpp` | +1,201 | 4 |
| `triggers.cpp` | +1,124 | 6 |
| `items.cpp` | +705 | 7 |
| `rpg.cpp` | +550 | 3b |
| `player.cpp` | +485 | 7 |
| `gauss.cpp` | +367 | 3b |
| `egon.cpp` | +421 | 3b |
| `python.cpp` | +319 | 3b |
| `effects.cpp` | +291 | 6 |
| `debugger.cpp` | +296 | 3b |
| `shotgun.cpp` | +277 | 3b |
| `mp5.cpp` | +278 | 3b |
| `glock.cpp` | +250 | 3b |
| `crossbow.cpp` | +241 | 3b |
| `scripted.cpp` | +234 | 9 |
| `client.cpp` | +208 | 3c, 7 |
| `satchel.cpp` | +204 | 3b |
| `tripmine.cpp` | +188 | 3b |
| `crowbar.cpp` | +160 | 3b |
| `squeakgrenade.cpp` | +155 | 3b |
| `subs.cpp` | +154 | 2 |
| `util.cpp` | +134 | 2 |
| `hornetgun.cpp` | +122 | 3b |
| `plats.cpp` | +110 | 6 |
| `handgrenade.cpp` | +103 | 3b |
| `doors.cpp` | +100 | 6 |

### Heavily Modified Client Files
| File | ±Lines | Phase |
|------|--------|-------|
| `ev_hldm.cpp` | +1,317 | 3c |
| `view.cpp` | +356 | 5 |
| `StudioModelRenderer.cpp` | +238 | 5 |
| `hud_msg.cpp` | +174 | 5, 8 |
| `hud_redraw.cpp` | +107 | 5, 8 |
| `inputw32.cpp` | +82 | 5 |
| `input.cpp` | +69 | 8 |
| `hud.cpp` | +63 | 5, 8 |
| `particlesys.cpp` | +103 | 8 |
| `vgui_TeamFortressViewport.cpp` | +94 | 7, 8 |

---

## Notes

### Differences Between 1.5 and 1.6
This plan is based on changes from 1.6 → 1.8. If your 1.5 codebase doesn't already
include SOHL 1.6 features, you may encounter additional conflicts. Key 1.5 → 1.6
features to check for:
- Locus system (may be simpler in 1.5)
- Alias system (may not exist in 1.5)
- MoveWith system
- Aurora particle system
- env_mirror, env_sky, env_fog entities

### Recommended Testing Maps
After each phase, test with these demo maps (from SOHL 1.8):
- `ParticleTest` — Particle system
- `locusdemo` — Locus calculations
- `cogwheeldemo` — Mechanical entity movement
- `garagedoordemo` — Door/platform movement
- `gatlinggundemo` — Weapon/combat
- `gruntbattledemo` — AI/combat
- `retinalscannerdemo` — Trigger logic
- `seamlessteleportdemo` — Teleportation
- `shinyfloordemo` / `shinywatertest` — Rendering
- `spiritdemo` — Comprehensive feature demo
- `utskytest` — Sky rendering
