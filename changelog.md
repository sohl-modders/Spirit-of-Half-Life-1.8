# Changelog: Spirit of Half-Life 1.6 → 1.8

This document details all changes introduced in the "Update to SOHL 1.8" commit,
comparing SOHL 1.6 (commit `135144a`) with SOHL 1.8 (commit `0f60f63`).

**Statistics**: 255 files changed, +59,181 insertions, −4,723 deletions

---

## Table of Contents

- [1. Weapon System Overhaul](#1-weapon-system-overhaul)
- [2. Client-Side Weapon Prediction](#2-client-side-weapon-prediction)
- [3. Locus Calculation System Rewrite](#3-locus-calculation-system-rewrite)
- [4. Inventory System (AJH)](#4-inventory-system-ajh)
- [5. Core Entity System Changes](#5-core-entity-system-changes)
- [6. Doors, Platforms & Buttons](#6-doors-platforms--buttons)
- [7. Trigger System Overhaul](#7-trigger-system-overhaul)
- [8. View & Camera System](#8-view--camera-system)
- [9. Rendering & Effects](#9-rendering--effects)
- [10. HUD System](#10-hud-system)
- [11. Scripted Sequences & AI](#11-scripted-sequences--ai)
- [12. Particle System Fixes](#12-particle-system-fixes)
- [13. FGD Entity Definitions](#13-fgd-entity-definitions)
- [14. Game Assets & Documentation](#14-game-assets--documentation)
- [15. Build System & Project Files](#15-build-system--project-files)
- [16. Bug Fixes & Misc Changes](#16-bug-fixes--misc-changes)

---

## 1. Weapon System Overhaul

### Architecture Change
- **Class declarations moved** from individual `.cpp` files into `weapons.h` for all weapons
  (`CGlock`, `CCrowbar`, `CPython`, `CMP5`, `CCrossbow`, `CShotgun`, `CRpg`, `CRpgRocket`,
  `CGauss`, `CEgon`, `CHgun`, `CHandGrenade`, `CSatchel`, `CSqueakGrenade`, `CTripmine`,
  `CWpnGeneric`). This is required for client-side weapon prediction.

### Signature Changes (all weapons)
- `Holster()` → `Holster(int skiplocal = 0)`
- `SendWeaponAnim()` gained `int skiplocal = 1, int body = 0` parameters
- `DefaultDeploy()` gained `int skiplocal = 0, int body = 0` parameters
- `DefaultReload()` gained `int body = 0` parameter
- Added `virtual BOOL UseDecrement()` (returns TRUE with CLIENT_WEAPONS)

### Removed Fields (weapons.h)
- `m_flTimeUpdate`, `m_flChargeTime`, `m_flShockTime`, `m_iChargeLevel`, `m_iOverloadLevel`
- `m_fInAttack`, `m_iLastSkin`, `m_iBody`, `m_iLastBody`, `b_Restored`, `AnimRestore`, `PlayerHasSuit`
- `RestoreBody()` method
- `SUIT` and `NUM_HANDS` macros
- Global `m_usDecals` and `m_usEfx` event declarations

### New Files
- **`weapon_generic.cpp`** — Mapper-configurable generic weapon with custom models (`m_iszModel`)
  and custom clip sizes (`m_iszClip`); uses `events/generic1-3.sc`
- **`debugger.cpp`** — Development `weapon_debug` entity with 15 debug impulse modes
  (entity info, AI state, node viewer, etc.)

### Per-Weapon Changes

| Weapon | Key Changes |
|--------|-------------|
| **Crossbow** | Bolt entity wrapped in `#ifndef CLIENT_DLL`; zoom toggle added to SecondaryAttack; animation enum expanded (`FIRE1/FIRE2/FIRE3`, `FIDGET2`); new precache events `m_usCrossbow`/`m_usCrossbow2` |
| **Crowbar** | Simplified attack logic; removed `m_flTimeUpdate` usage; removed `m_usCrowbar` event |
| **Egon** | Animation enum overhauled (added `EGON_FIRE1-4`, `EGON_ALTFIRECYCLE`; removed `FIRESTOP`/`FIRECYCLE`); volume constants renamed |
| **Gauss** | Added `GetFullChargeTime()` (1.5s multiplayer, default otherwise); renamed volume constants; `g_pGameRules->IsMultiplayer()` checks |
| **Glock** | Removed `GLOCK_HOLSTER2`/`GLOCK_DEL_SILENCER`; added `AddToPlayer()` with bug fix; new events `m_usFireGlock1`/`m_usFireGlock2` |
| **MP5** | Animation enum reordered; new events `m_usMP5`/`m_usMP52` |
| **Python** | Added `AddToPlayer()` with pickup message; removed laser spot; zoom reworked; event `m_usFirePython` |
| **RPG** | Animation enum expanded (FIDGET, FIRE2, HOLSTER1/2, DRAW1, DRAW_UL, IDLE_UL, FIDGET_UL); `CLaserSpot` moved here; full `CRpgRocket` Save/Restore |
| **Shotgun** | Animation enum reordered (IDLE4, IDLE_DEEP added); Save/Restore for reload state; events `m_usDoubleFire`/`m_usSingleFire` |
| **Satchel** | Collision box commented out (was blocking players) |
| **Squeakgrenade** | Snark entity wrapped in `#ifndef CLIENT_DLL` |
| **Tripmine** | World model changed (`w_tripmine.mdl` → `v_tripmine.mdl` body=3); removed mirror beam (`m_pMirBeam`); explicit SetSize replaces AutoSetSize |
| **Handgrenade** | `m_iChargeLevel` replaced with `m_flReleaseThrow`; throw timing uses `gpGlobals->time`; removed charge display logic |
| **Hornetgun** | `IsUseable()` extracted; event `m_usHornetFire` |

### Decal System Reverted
- All `PLAYBACK_EVENT_FULL(m_usDecals, ...)` calls in `combat.cpp` replaced with standard
  `UTIL_BloodDecalTrace()` / `DecalGunshot()` calls (1.6's custom decal event system removed)

---

## 2. Client-Side Weapon Prediction

### New Client-Side Files
| File | Lines | Purpose |
|------|-------|---------|
| `cl_dll/com_weapons.cpp` | 277 | Weapon prediction utility stubs |
| `cl_dll/com_weapons.h` | 26 | Weapon prediction header |
| `cl_dll/hl/hl_baseentity.cpp` | 349 | Client-side stub entity implementations |
| `cl_dll/hl/hl_events.cpp` | 84 | `Game_HookEvents()` with all weapon event registrations |
| `cl_dll/hl/hl_objects.cpp` | 90 | `Game_AddObjects()` with Egon beam update |
| `cl_dll/hl/hl_weapons.cpp` | 1,101 | Full client-side weapon simulation (all weapons) |

### Event System Restructured
- `events.cpp` simplified to call `Game_HookEvents()` (from `hl/hl_events.cpp`)
- New events: `EV_FireGlock2`, `EV_GenericFire1/2/3`, `EV_FireMP52`, `EV_FireSnark`
- Removed: `EV_FireCrowbar`, `EV_UpdateBeams`, `EV_HLDM_GetMirroredPosition`
- Weapon enums reordered to match standard HL SDK (shotgun, glock, mp5)
- New `generic_e` enum for weapon_generic events

### Client Weapon Data
- Full `GetWeaponData()` implementation in `client.cpp` packing weapon state
- `#if defined(CLIENT_WEAPONS)` weapon decay timer logic in `player.cpp` `PostThink()`

---

## 3. Locus Calculation System Rewrite

### API Changes (`locus.cpp`, `locus.h`, `cbase.h`)
- **Return style changed**: Output-parameter pattern replacing direct returns
  - `Vector CalcPosition(CBaseEntity*)` → `bool CalcPosition(CBaseEntity*, Vector* OUTresult)`
  - `Vector CalcVelocity(CBaseEntity*)` → `bool CalcVelocity(CBaseEntity*, Vector* OUTresult)`
  - `float CalcRatio(CBaseEntity*)` → `bool CalcNumber(CBaseEntity*, float* OUTresult)`
  - New: `CalcPYR()` for pitch/yaw/roll calculation
- All `CalcLocus_Ratio` calls renamed to `CalcLocus_Number` throughout codebase

### New Capabilities
- **Component-wise vector parsing**: `TryParseVectorComponentwise()` supports `'x y z'` and
  `'x1..x2 y1..y2 z1..z2'` random ranges
- **Bracket parsing**: `TryParseLocusBrackets()` for nested locus expressions
- **Number parsing**: `TryParseLocusNumber()` with vector swizzle (x/y/z/p/r components),
  PITCH/YAW/LENGTH properties
- **`CMark` class**: Runtime-created position/number marker entity

### New FGD Entities
| Entity | Replaces | Purpose |
|--------|----------|---------|
| `calc_numfromnum` | `calc_ratio` | Number adjustment |
| `calc_posfroment` | `calc_position` | Position from entity |
| `calc_vecfroment` | `calc_subvelocity` | Velocity from entity |
| `calc_vecfrompos` | `calc_velocity_path` | Velocity for travel path |
| `calc_vecfromvec` | `calc_velocity_polar` | Velocity modification |
| `calc_angles` | *(new)* | Angle calculation |
| `calc_numfromcvar` | *(new)* | Value from console variable |
| `calc_fallback` | *(new)* | Error handling for failed calcs |
| `calc_subratio` | *(new)* | Number from entity properties |
| `calc_numfroment` | *(new)* | Number from entity (health, speed) |
| `calc_numfromvec` | *(new)* | Number from vector component |

---

## 4. Inventory System (AJH)

### Player Changes (`player.cpp`, `player.h`)
- `m_rgItems[MAX_ITEMS]` array for player inventory
- `m_pItemCamera` pointer for remote camera item
- New network message: `gmsgInventory`
- Spawn resets inventory and camera state

### New Items (`items.cpp`, `items.h`)
| Item | Class | Description |
|------|-------|-------------|
| `item_medicalkit` | `CItemMedicalKit` | Portable medkit with charge system (50 charge, max 200) |
| `item_antidote` | `CItemAntidote` | Anti-toxin (extracted from anonymous class) |
| Anti-radiation | `CItemAntiRad` | Anti-radiation syringe (cures radiation damage) |
| `item_flare` | `CItemFlare` | Deployable flare |
| `item_camera` | `CItemCamera` | Remote viewing camera |
- New `I_Precache()` function precaching all inventory items
- All items send `gmsgInventory` HUD updates on pickup

### Client Commands (`client.cpp`)
- New `"inventory"` command with sub-commands: 1=medkit, 2=antitox, 3=antirad, 6=flare, 7=camera

### HUD Integration
- `vgui_TeamFortressViewport.cpp/h`: `InventoryButton` class, `g_iInventory[]` array,
  inventory selection UI
- `!SHOWINVENTORY` command menu button type

---

## 5. Core Entity System Changes

### CBaseEntity (`cbase.h`, `cbase.cpp`)
- **Alias system renamed**: `CBaseAlias` → `CBaseMutableAlias`, `IsAlias()` → `IsMutableAlias()`
- `FollowAlias()` promoted from alias-only to all CBaseEntity
- New use type: `USE_SPAWN = 7` with fire-target prefix `&`
- Custom death notice: `killname`, `killmethod` string_t fields
- `RADIATION_DURATION` changed: 2 → 10
- Added `g_allowGJump` (single-player Gauss jump toggle)
- **Removed**: `UTIL_AutoSetSize()` (studio header auto-collision)
- `TakeHealth()` rewritten (AJH): returns actual health amount given (not just 0/1)
- Removed `pev->colormap = ENTINDEX(pent)` from DispatchSpawn/DispatchRestore

### CBaseToggle Acceleration (`subs.cpp`)
- New fields: `m_flLinearAccel`, `m_flLinearDecel`, `m_flCurrentTime`, `m_flAccelTime`,
  `m_flDecelTime`, `m_bDecelerate`
- New `LinearMove(dest, speed, accel, decel)` overload implementing gradual
  acceleration/deceleration (time-step thinking at `ACCELTIMEINCREMENT = 0.1`)

### Fire Targets (`subs.cpp`)
- `USE_SPAWN` prefix `&` support
- `FireTargets()` uses `CalcLocusParameter()` for activator resolution

---

## 6. Doors, Platforms & Buttons

### Accelerating Doors (`doors.cpp`)
- New keyvalues: `acceleration`, `deceleration`, `speedmode`
- `DoorGoUp`/`DoorGoDown` use `LinearMove(dest, speed, accel, decel)` when speedmode ≠ 1
- Blocked damage now attributed to activator instead of door entity
- Save/Restore for acceleration fields

### Platforms (`plats.cpp`)
- `CFuncTrain::Next()`: movement sound only restarts when state is OFF
- `ThinkDoNext()`: waits for game start (`gpGlobals->time != 1.0`)
- **Removed**: `CFuncVehicle` class (~75 lines of unfinished vehicle code)

### Buttons (`buttons.cpp`)
- Spark volume range: `(0.1, 0.25)` → `(0.25, 0.75)`
- `CEnvSpark`: New spawnflag 16 for cyclic mode (`SparkCyclic()`/`SparkWait()`)

### Momentary Door
- `CalcRatio` → `CalcNumber` (locus system update)

---

## 7. Trigger System Overhaul

### trigger_relay → CTriggerRelay
- Added `SF_RELAY_DEBUG` (0x02) spawnflag with extensive debug logging
- Added `SF_FIRE_CAMERA` (0x04): only fires when player views through camera
- New `m_iszAltTarget` field for alternate target when locked by master
- `USE_SAME` and `USE_SET` handling with proper value passing
- Removed hardcoded HACK for c1a4f elevator

### trigger_auto
- Removed hardcoded HACK overrides for c5a1 (stars effect)

### General
- `CalcLocus_Ratio` → `CalcLocus_Number` throughout all triggers

---

## 8. View & Camera System

### Sky Rendering (`view.cpp`)
- New `SKY_ON_DRAWING = 2` state for two-pass rendering pipeline
- **Parallax sky support**: `m_iSkyScale` offsets sky view by `m_vecSkyPos + v_origin/m_iSkyScale`
- Fixed sky/camera coexistence bugs (AJH)
- `ClearToFogColor()` call on first pass

### Camera System (`view.cpp`)
- Removed `studiohdr_t` eye-position lookups
- `m_iCameraMode=2` flag for drawing player in camera view (AJH)
- New commands: `drawplayer`/`hideplayer`
- Complete `V_CalcThirdPersonRefdef` rewrite with two-pass sky support

### View Clamping (LRC 1.8, `inputw32.cpp`)
- New `gmsgClampView` message
- `V_ClampYaw()`: server-defined yaw range with 360° wraparound
- `V_ClampPitch()`: min/max pitch clamp
- `V_LimitClampSpeed()`: angular velocity limiting
- Applied to both mouse and joystick input

---

## 9. Rendering & Effects

### StudioModelRenderer
- **Removed**: Custom shadow drawing hack (`GL_StudioDrawShadow` pointer, `DrawShadows()`)
  → standard `IEngineStudio.GL_StudioDrawShadow()` call
- **Removed**: Platform-specific OpenGL includes → `glInclude.h`
- Player model marker: checks `models/null.mdl` instead of `m_iCameraMode`
- Mirror rendering moved to `StudioDrawPlayer` (manipulates `m_protationmatrix`)
- Sky culling: `kRenderFxEntInPVS` check skips entities during sky pass

### Fog System (`tri.cpp`, `hud.cpp`, `hud_msg.cpp`, `hud_redraw.cpp`)
- **New `FogSettings` struct**: `fogColor[3]`, `startDist`, `endDist`
- **Smooth fog transitions**: Pre/post fade states with `UTIL_Lerp()` interpolation
- `MsgFunc_SetFog`: reads 3 color bytes + fade duration + start/end distances
- `RenderFog()` uses engine `pTriAPI->Fog()` instead of raw GL calls
- `BlackFog()` for transparent triangle rendering
- `ClearToFogColor()` for sky pass

### Effects (`effects.cpp`)
- `CLightning`: attachment points (`m_iStartAttachment`/`m_iEndAttachment`)
- `env_elight`: unique per-entity keys, proper Save/Restore
- `CGibShooter`: re-enabled batch-fire mode (`m_flDelay == 0`)
- `CEnvFog`: base class changed to `CPointEntity`; added `Resume2Think`
- `CEnvSky`: `pev->frags` for skybox scale (parallax)
- `CParticle`: active state from targetname; `USE_SPAWN` support
- `CEnvMirror`: complete rewrite with `PLAYBACK_EVENT_FULL`, `SF_MIRROR_DRAWPLAYER`

### OpenGL Headers (new)
| File | Lines | Purpose |
|------|-------|---------|
| `common/gl/gl.h` | 1,526 | Standard OpenGL API |
| `common/gl/glext.h` | 3,081 | GL extensions |
| `common/gl/glu.h` | 584 | GLU utility library |
| `common/cg/*.h` | ~3,064 | NVIDIA Cg shader runtime (12 files) |
| `cl_dll/glInclude.h` | 30 | Portable GL include wrapper |

---

## 10. HUD System

### New Messages
- `gmsgClampView` — view angle clamping (LRC 1.8)
- `gmsgInventory` — inventory item updates (AJH)
- `gmsgKeyedELight` — entity lights (replaces `gmsgKeyedDLight`)

### Removed Messages
- `gmsgSetBody`, `gmsgSetSkin`, `gmsgSetMirror`, `gmsgResetMirror`

### New HUD Elements
- Camera overlay (AJH): flashing camera icon + corner reticle brackets when viewing through camera
- Custom menu system (`CustomMenu.cpp`): VGUI-based scrollable menu panel (`MENU_CUSTOM = 10`)
- Glow effect CVARs (FragBait0): `r_glow`, `r_glowstrength`, `r_glowblur`, `r_glowdark`

### Removed
- `r_shadows` CVAR

### Sky Message
- `gmsgSetSky` size increased 7→8 (includes `m_iSkyScale` byte)

### Death Messages (`death.cpp`)
- Custom death technique: `%` delimiter splits technique string for custom kill messages
  (e.g. `" was eaten by %'s pet"` → `"Player1 was eaten by Player2's pet"`)

---

## 11. Scripted Sequences & AI

### Scripted Sequences (`scripted.cpp/h`)
- **Priority system** (LRC): enum replaced with flags:
  `SS_INTERRUPT_IDLE=0x0`, `SS_INTERRUPT_ALERT=0x1`, `SS_INTERRUPT_ANYSTATE=0x2`,
  `SS_INTERRUPT_SCRIPTS=0x4`
- `CanPlaySequence(BOOL, int)` → `CanPlaySequence(int interruptFlags)`
- New keyvalues: `m_fTurnType`, `m_iRepeats`/`m_iRepeatsLeft`/`m_fRepeatFrame`, `m_iPriority`
- `InitIdleThink()` for proper idle animation initialization
- Expanded `PossessEntity()` with move-to cases 0-6
- Debug output changed from `at_console` to `at_debug`

### AI Schedule System
- **`AI_BaseNPC_Schedule.cpp`** (1,621 lines): Comprehensive NPC schedule framework with
  new schedule types, task definitions, condition-based behavior transitions

### Monster Changes
- `CalcRatio` → `CalcNumber` (health ratio)
- Dead monsters return health ratio 0
- `GetState()` added returning ON/OFF based on dead flag

---

## 12. Particle System Fixes

### Entity Lookup Fix (`particlesys.cpp`)
- Changed from `GetClientEntityWithServerIndex()` (searching by colormap) to
  `gEngfuncs.GetEntityByIndex()` (direct index lookup) — **significant bug fix**
- Added PVS check: skip particle updates for entities outside player's view

### Removed
- `sprayroll` parameter
- `ParticleIsVisible()` frustum culling (was unreliable)
- `angles` member from particle structs

### Audio
- `mp3.cpp`: FMOD library changed from `mp3dec.dll` to `fmod.dll`; re-enabled loop mode;
  media path changed to root game directory

---

## 13. FGD Entity Definitions

### Renamed: `Spirit1.5.fgd` → `spirit18.fgd`
- **+2,142 lines added, −309 lines removed**

### New Entities (17 total)
`calc_angles`, `calc_numfromcvar`, `calc_fallback`, `calc_subratio`, `calc_numfroment`,
`calc_numfromnum`, `calc_numfromvec`, `calc_posfroment`, `calc_vecfroment`, `calc_vecfrompos`,
`calc_vecfromvec`, `item_camera`, `item_flare`, `item_medicalkit`, `monster_flyer`,
`watcher_alias`, `watcher_number`, `weapon_eagle`

### Key Entity Modifications
- `env_explosion`: `radiusscale` keyvalue (NEW 1.8)
- `func_pushable`: spawnflag 512 "Can't Pull"
- `worldspawn`: `max_cameras`, `max_medkit`, `timed_damage` keyvalues
- `scripted_sequence`: size changed from `(-16 -16 -16, 96 96 96)` to `(-16 -16 0, 16 16 72)`
- `trigger_once` base classes changed
- `env_shockwave`: gained `MoveWith` base

---

## 14. Game Assets & Documentation

### New Documentation (Spirit/docs/)
| File | Lines | Content |
|------|-------|---------|
| `EntityGuide.html` | 7,987 | Full entity reference |
| `Spirit of Half-Life Entity Guide.htm` | 16,337 | Comprehensive entity guide |
| `Features.htm` | 1,178 | Feature list |
| `Features.html` | 429 | Feature overview |
| `FAQ.html` | 117 | Frequently asked questions |
| `Setup.html` | 59 | Worldcraft setup instructions |
| `LOCUS [LR] RATIO REFERENCE.htm` | 140 | Locus ratio reference |
| MHLT docs | 518 | Modified Half-Life Tools docs |

### New Demo Maps (13 BSP + 13 RMF)
`ParticleTest`, `cogwheeldemo`, `garagedoordemo`, `gatlinggundemo`, `gruntbattledemo`,
`locusdemo`, `migrainedemo`, `retinalscannerdemo`, `seamlessteleportdemo`, `shinyfloordemo`,
`shinywatertest`, `spiritdemo` (1.4MB), `utskytest`

### New Models
- `null.mdl` (empty/null model), `w_camera.mdl`, `w_portablemed.mdl`, `w_portablemedt.mdl`
- Modified: `v_9mmAR.mdl`

### New Sprites (17)
- Camera HUD: `320_camera.spr`, `640_camera.spr`, `640_camera_rect.spr`
- Sky/background: `BenSky.spr`, `bgspr.spr`
- Glows: `blueglow.spr`, `redglow.spr`
- Editor helpers: `calc.spr`, `env.spr`, `envbeam.spr`, `game.spr`, `info.spr`,
  `trigger.spr`, `multimanager.spr`, `pathcorner.spr`, `scriptedsequence.spr`
- Config: `GameCfg.wc`

### New HUD/Sprite Definitions
- `hud.txt` (166 lines): Complete HUD sprite layout including camera system sprites
- `weapon_debug.txt` (9 lines): Debug weapon sprite definitions

### New Audio
- `sentences.txt` (1,376 lines, 1,052 sentences): Complete Half-Life sentence definitions
- `logo.avi`: Actual video (was 26 bytes → 144KB)

### New Aurora Particles
- `teleport.aur`: Teleport particle effect (4,000 particles)
- `AuroraReference.html`: Particle system documentation

### Menu/UI Graphics
- `btns_main.bmp` (840KB): Main menu buttons
- 25+ `head_*.BMP` files: Menu headers
- `splash.bmp`: Modified splash screen
- `Colors.lst`: Color definitions

### Other
- `spirit.ico`: Application icon
- `Run Spirit.bat`: Launcher (`hl -dev -console -game Spirit`)
- `events/generic1-3.sc`, `mirror.sc`: Empty event scripts
- **Removed**: `load.bat` (30 lines)

---

## 15. Build System & Project Files

### New VS .NET Files
| File | Purpose |
|------|---------|
| `dlls/hl.sln` | Server DLL solution |
| `dlls/hl.vcproj` | Server DLL project (3,542 lines) |
| `dlls/hl.suo` | Server DLL user options |
| `cl_dll/cl_dll.sln` | Client DLL solution |
| `cl_dll/cl_dll.vcproj` | Client DLL project (2,209 lines) |
| `cl_dll/cl_dll.suo` | Client DLL user options |
| `SpiritSource/hl.dsw` | Combined workspace (41 lines) |
| `SpiritSource/hl.opt` | Workspace options |

### Updated Project Files
- `dlls/hl.dsp`: +124 lines (new source files added)
- `cl_dll/cl_dll.dsp`: +147 lines (new source files added)

### liblist.gam Changes
| Field | 1.6 | 1.8 |
|-------|-----|-----|
| `game` | "SoHL:Custom Build" | "Spirit of Half-Life" |
| `url_info` | shamteam.cbi.ru | spirit.valve-erc.com |
| `version` | "1.6" | "1.2" |
| `startmap` | "weathdemo" | "ParticleTest" |
| `trainmap` | "mirrordemo" | "ShinyWaterTest" |
| `gamedll` | "cl_dlls/server.dll" | "dlls/spirit.dll" |

---

## 16. Bug Fixes & Misc Changes

### Bug Fixes
- **Particle entity lookup**: Fixed unreliable `GetClientEntityWithServerIndex()` (colormap search)
  → `gEngfuncs.GetEntityByIndex()` (direct index)
- **Glock AddToPlayer**: Fix noted as "Fix old Half-life bug"
- **TakeHealth**: Now returns actual health amount given instead of 0/1
- **Train sound**: Only restarts movement sound when state changes to OFF
- **Sky/camera coexistence**: Fixed rendering conflicts (AJH)
- **Fog fading**: Complete rewrite with proper interpolation
- **Blocked damage attribution**: Doors now attribute to activator, not door entity

### Removed Features
- `CFuncVehicle` (unfinished vehicle code)
- Custom shadow drawing hack
- `UTIL_AutoSetSize()` (auto collision box)
- `r_shadows` CVAR
- Mirror beam on tripmine
- `kRenderFxMirror` in `StudioFxTransform`
- Body/skin network messages

### Misc
- `CBasePlayerAmmo` now derives from `CBasePlayerItem`
- `CEnvFog` base class: `CBaseEntity` → `CPointEntity`
- `hud_spectator.cpp`: `double()` cast for VS.NET compatibility
- `rain.cpp`: Copyright updated to "BUzer" (original author)
- Debug output changed from `at_console` to `at_debug` in scripted sequences

---

## Credits (from Spirit/README.HTML)

- **Laurie Cheers** (LRC) — Design, code, documentation
- **MR** — SpiritDemo map, icon, menu background
- **Codiac** — SDK 2.3 port
- **Autolycus / valve-erc.com** — Hosting
- **AJH** — Inventory system, camera system, custom HUD, view fixes
- **FragBait0** — Glow effect system
