# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

**GOLDEN DIRECTIVE**: ALWAYS ASK THE USER TO PROVIDE VALUABLE INFORMATION BEFORE FILLING KNOWLEDGE GAPS WITH ASSUMPTIONS OR GUESSES DURING YOUR THINKING

## Project Overview

IWinEnhanced is a World of Warcraft 1.12.1 (Turtle WoW) addon providing full 1-button rotation macros for 3 classes: Druid, Paladin, and Warrior. Other classes are NYI. Players bind a single slash command (e.g., `/idps`) to a key; each press evaluates conditions and casts the optimal next ability.

There is no build system, test framework, or linter. The addon is loaded directly by the WoW client from the `.toc` file. To test changes, reload the UI in-game with `/reload`.

### User Turtle WOW Character Information

- Race: Human
- Class: Warrior
- Level: 60
- Specialization 1: 2handed Arms with Mortal Strike & improved slam
- Specialization 2: 2handed Fury with Bloodthirst and Master Strike  (no mortal strike or improved slam)

## Architecture

### Global Namespace

All addon state lives on a single global frame: `IWin = CreateFrame("frame", nil, UIParent)`. Functions are methods on this frame (`IWin:FunctionName()`). A tooltip frame `IWin.t` is used for buff/debuff scanning.

### Load Order (defined in IWinEnhanced.toc)

The TOC enforces strict load order across 6 layers. Every layer loads core first, then all 9 class files:

1. **IWinEnhanced.lua** - Creates the `IWin` frame
2. **data.lua** - Global constants (core), then class-specific constants
3. **init.lua** - Class module initialization and combat variable declarations
4. **condition.lua** - Buff/debuff utilities (core), then class-specific condition checks
5. **action.lua** - Core actions (targeting, items), then class-specific spell functions
6. **event.lua** - Class event listeners and `ADDON_LOADED` initialization
7. **setup.lua** - `/iwin` configuration subcommands per class
8. **rotation.lua** - Core hydration command, then class rotation slash commands

### Class Module Structure

Each class has its own directory (e.g., `warrior/`) with 7 files following a consistent pattern. Every class file starts with a guard clause:

```lua
if UnitClass("player") ~= "Warrior" then return end
```

This means all class files load for every player but only execute for the matching class.

| File | Purpose |
|------|---------|
| `init.lua` | Declares `IWin_CombatVar` (runtime state) and `IWin_Target` (target state) tables |
| `data.lua` | Class-specific lookup tables (spell costs, talent modifiers) |
| `condition.lua` | Predicate functions (proc windows, resource checks, buff states) |
| `action.lua` | Spell-casting functions with multi-condition guards |
| `event.lua` | `RegisterEvent()` calls and `OnEvent` handler; settings init on `ADDON_LOADED` |
| `setup.lua` | `/iwin` subcommand parsing for per-character settings |
| `rotation.lua` | Slash command definitions (e.g., `SLASH_IDPSWARRIOR1 = "/idps"`) containing ordered ability priority lists |

### Key Patterns

**GCD tracking**: `IWin_CombatVar["queueGCD"]` is set to `false` when a spell is cast, preventing double-casts in the same frame. Each action function checks this before casting.

**Resource reservation**: `IWin_CombatVar["reservedRage"]` (or mana/energy equivalent) tracks resources that should be saved for higher-priority upcoming abilities.

**Proc windows**: Timed proc availability tracked via `GetTime()` comparisons (e.g., `IWin_CombatVar["overpowerAvailable"] = GetTime() + 5`).

**Rotation structure**: Each rotation slash command is a flat sequence of function calls in priority order. The first ability whose conditions are met gets cast; subsequent calls are no-ops due to `queueGCD = false`.

```lua
function SlashCmdList.IDPSWARRIOR()
    IWin:InitializeRotation()  -- reset combat vars
    IWin:TargetEnemy()
    IWin:HighPriorityAbility()
    IWin:MediumPriorityAbility()
    IWin:LowPriorityAbility()
    IWin:StartAttack()         -- ensure auto-attack
end
```

**Settings**: Stored in `IWin_Settings` (SavedVariablesPerCharacter). Initialized with nil-checks in `event.lua` on `ADDON_LOADED`.

### Dependencies

**Required client mods**: SuperWoW (expanded Lua API), UnitXP (macro conditions), Nampower (cast efficiency/range checks)

**Required addon**: SuperCleveRoidMacros (provides `CleveRoids.libdebuff` for debuff tracking, referenced as `IWin.libdebuff`)

**Optional addons**: SP_SwingTimer (swing timing for Slam), PallyPowerTW, LibramSwap, TimeToKill, MonkeySpeed

The user has the SP_SwingTimer, TImetoKill, and MonkeySpeed addons installed.

### API Notes

This targets the WoW 1.12.1 API (Interface: 11200). Key API functions used throughout:
- `CastSpellByName(name)` - Cast a spell
- `GetPlayerBuff(index, filter)` / `GetPlayerBuffTimeLeft(index)` - Buff scanning
- `UnitBuff(unit, index)` / `UnitDebuff(unit, index)` - Unit aura scanning
- `GetTime()` - High-resolution timer for proc/cooldown tracking
- `SlashCmdList` / `SLASH_*` globals - Slash command registration

The addon relies on SuperWoW extensions CleveRoids library functions, and UnitXP that are not part of the standard 1.12.1 API.

**SuperWoW,  and CleveRoids Library Function Reference Guide**:  `./TURTLEWOW_API GUIDE.md`

## Known Issues / Next Steps

### Completed: Mortal Strike blocked in Berserker Stance
**Root cause**: `SetSlamQueued()` set `slamQueued = true`, which blocked Mortal Strike's guard condition (`not slamQueued`). Combined with Slam's rage reservation, MS was double-blocked in Berserker stance.
**Fix applied**: Commented out `SetSlamQueued()` calls in `warrior/rotation.lua` (lines 31, 83). MS now fires with the same priority in Berserker stance as Battle stance.

### Completed: Cleave never casts in /icleave rotation
**Root cause**: Rage reservation stacking. Cleave is evaluated last in the rotation (`warrior/rotation.lua:101`). By that point, all `SetReservedRage()` calls above it have accumulated into `reservedRage`. The `IsRageAvailable("Cleave")` check in `core/condition.lua:280-287` computes: `rageRequired = 20 (Cleave cost) + reservedRage + 20 (next-melee penalty at line 283)`. With typical reservations totaling ~120+ rage, effective cost exceeds 100 max. The fallback `UnitMana("player") > 80` was also rarely met mid-combat.
**Fix applied**: Replaced Cleave's weak fallback in `warrior/action.lua:185-196` with a smart fallback mirroring HeroicStrike's pattern (`warrior/action.lua:425-442`). Cleave now casts when rage > 60 AND both Whirlwind and Sweeping Strikes are on cooldown (or unlearned), bypassing the reservation system safely. Fixes both `/icleave` and `/ihodor` rotations.
