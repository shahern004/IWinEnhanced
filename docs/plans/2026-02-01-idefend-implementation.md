# `/idefend` Shield Swap Implementation Plan

**Goal:** Add a 3-phase deterministic `/idefend` command: Phase 1 equips shield + Defensive Stance, Phase 2 casts defensive cooldowns, Phase 3 (shift + press) swaps back to DW.

**Architecture:** State is implicit from equipment + modifier key — no flags or combat variables needed. `IsShieldEquipped()` determines phase 1 vs 2/3, `IsShiftKeyDown()` determines phase 2 vs 3. GCD system provides natural spam protection. Weapon names persist in `IWin_Settings` (SavedVariablesPerCharacter).

**Equipment swap mechanism:** Call SuperCleveRoidMacros' `/equipoh` handler from Lua via dynamic `SlashCmdList` lookup (`IWin:RunSlashCmd`). SuperCleveRoidMacros is a mandatory dependency that provides proven in-combat equip-by-name.

---

## Key Design Decisions

### Equipment operations use SuperCleveRoidMacros slash commands
Do NOT use low-level WoW cursor APIs (`PickupContainerItem`, `PickupInventoryItem`) for equipment swapping — they silently fail in practice. Instead, call SuperCleveRoidMacros' `/equipoh` handler which already solves bag search + slot targeting:

```lua
IWin:RunSlashCmd("/equipoh", "Draconian Deflector")  -- equip shield
IWin:RunSlashCmd("/equipoh", "Brutality Blade")      -- swap back to OH
```

### State detection is implicit, not explicit
No `IWin_CombatVar["defendMode"]` flag needed. Phase is determined by:
- `not IsShieldEquipped()` → Phase 1 (equip shield)
- `IsShieldEquipped() and not IsShiftKeyDown()` → Phase 2 (cooldowns)
- `IsShieldKeyDown() and IsShiftKeyDown()` → Phase 3 (swap back)

### Swap-back is always manual
The player presses shift + `/idefend` to return to DW. No automatic swap-back based on buff expiry.

### MH weapon is never changed
Only the OH slot (17) changes between shield and weapon. MH slot (16) stays equipped throughout. We save MH name for safety but only `/equipoh` is called.

### Cache invalidation is free
`InitializeRotationCore()` clears `cachedShieldEquipped` every key press. After the equip, the next key press detects the new equipment state automatically.

---

## Task 1: Rage costs ✅ DONE

Add `["Shield Wall"] = 0` and `["Last Stand"] = 0` to `IWin_RageCost` in `warrior/data.lua`.

---

## Task 2: Settings defaults and configuration ✅ DONE

- `warrior/event.lua` — defaults for shield, laststand, savedMH, savedOH
- `warrior/setup.lua` — validation + assignment for `/iwin shield` and `/iwin laststand`

---

## Task 3: Action functions for shield equip, cooldowns, and swap-back

**Files:** `warrior/action.lua`

### Step 1: Add `RunSlashCmd()` helper

Dynamic slash command handler lookup. Finds the handler for a given command string (e.g., `/equipoh`) in the global `SlashCmdList` and calls it.

```lua
function IWin:RunSlashCmd(cmd, args)
    for name, handler in pairs(SlashCmdList) do
        local i = 1
        while _G["SLASH_" .. name .. i] do
            if _G["SLASH_" .. name .. i] == cmd then
                handler(args or "")
                return
            end
            i = i + 1
        end
    end
end
```

### Step 2: Add `SaveDualWieldWeapons()`

Reads currently equipped MH (slot 16) and OH (slot 17) names via `GetInventoryItemLink` → `GetItemID` → `GetItemInfo` and saves to `IWin_Settings`. (Already implemented, no changes needed.)

### Step 3: Rewrite `EquipShield()` to use `RunSlashCmd`

```lua
function IWin:EquipShield()
    if IWin_Settings["shield"] == "" then
        DEFAULT_CHAT_FRAME:AddMessage("|cff0066ff /idefend: No shield configured. Use /iwin shield <name>|r")
        return
    end
    if not IWin:IsShieldEquipped() then
        IWin:SaveDualWieldWeapons()
        IWin_CombatVar["queueGCD"] = false
        IWin:RunSlashCmd("/equipoh", IWin_Settings["shield"])
    end
end
```

### Step 4: Rewrite `ReequipDualWield()` to use `RunSlashCmd`

```lua
function IWin:ReequipDualWield()
    if IWin_Settings["savedOH"] ~= "" then
        IWin:RunSlashCmd("/equipoh", IWin_Settings["savedOH"])
    end
    IWin_CombatVar["queueGCD"] = false
end
```

Note: MH equip removed entirely — MH is never changed during the shield swap flow.

### Step 5: Delete deprecated functions

Remove `FindItemInBags()` and `EquipItemToSlot()` completely — these used the failed `PickupContainerItem`/`PickupInventoryItem` approach.

### Step 6: Keep cooldown functions unchanged

`DefensiveStanceDefend()`, `LastStandDefend()`, `ShieldWallDefend()`, `LastStandDefendNormal()` — no changes needed.

---

## Task 4: `/idefend` slash command ✅ DONE

Already in `warrior/rotation.lua`. No changes needed — the rotation calls `EquipShield()` and `ReequipDualWield()` which are being rewritten in Task 3.

---

## Task 5: README ✅ DONE

---

## Task 6: In-game verification

After implementation, test with `/reload`:

1. **Configure**: `/iwin shield <name>`, `/iwin laststand 40`
2. **Phase 1**: Press `/idefend` — shield equips, Defensive Stance swaps
3. **Phase 2**: Press `/idefend` — Shield Wall / Last Stand casts
4. **Phase 3**: Shift + `/idefend` — OH weapon re-equips
5. **Persistence**: `/reload`, then `/iwin` — verify saved weapons persisted
