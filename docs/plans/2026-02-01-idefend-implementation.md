# `/idefend` Shield Swap Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add a 3-phase deterministic `/idefend` command: Phase 1 equips shield + Defensive Stance, Phase 2 casts defensive cooldowns, Phase 3 (shift + press) swaps back to DW.

**Architecture:** State is implicit from equipment + modifier key — no flags or combat variables needed. `IsShieldEquipped()` determines phase 1 vs 2/3, `IsShiftKeyDown()` determines phase 2 vs 3. GCD system provides natural spam protection. Weapon names persist in `IWin_Settings` (SavedVariablesPerCharacter).

**Tech Stack:** WoW 1.12 Lua API, `EquipItemByName(name, slot)` for in-combat weapon swaps, `IsShiftKeyDown()` for modifier detection, `IWin:IsShieldEquipped()` for phase detection.

---

## Key Design Decisions

### State detection is implicit, not explicit
No `IWin_CombatVar["defendMode"]` flag needed. Phase is determined by:
- `not IsShieldEquipped()` → Phase 1 (equip shield)
- `IsShieldEquipped() and not IsShiftKeyDown()` → Phase 2 (cooldowns)
- `IsShieldEquipped() and IsShiftKeyDown()` → Phase 3 (swap back)

This avoids state sync bugs (e.g. player manually unequips shield).

### Swap-back is always manual
The player presses shift + `/idefend` to return to DW. No automatic swap-back based on buff expiry. The shield itself is a defensive tool independent of cooldowns.

### Last Stand priority is dynamic
When `HP < /iwin laststand threshold`, Last Stand leapfrogs Shield Wall in priority. Otherwise order is Shield Wall → Last Stand. One cast per press.

### Weapon memory lives in `IWin_Settings`
`IWin_CombatVar` resets on `/reload`. DW weapon names persist in `IWin_Settings` (SavedVariablesPerCharacter) so they survive reloads mid-raid.

### Cache invalidation is free
`InitializeRotationCore()` (core/action.lua:35) clears `cachedShieldEquipped` every key press. After `EquipItemByName`, the next key press detects the new equipment state automatically.

### Spam protection is the GCD
Each phase sets `queueGCD = false`. Equipment swap triggers ~1.5s GCD. Cooldowns each consume a GCD. Mashing naturally sequences through phases at ~1.5s intervals. Shift must be deliberately held for swap-back.

---

## Task 1: Add rage costs for Shield Wall and Last Stand

**Files:**
- Modify: `warrior/data.lua:59-70` (add entries to `IWin_RageCost` table)

**Step 1: Add the rage cost entries**

Add after the `["Recklessness"] = 0,` line (line 59) in `IWin_RageCost`:

```lua
	["Shield Wall"] = 0,
	["Last Stand"] = 0,
```

**Step 2: Verify no syntax errors**

Visual inspection: ensure trailing commas are correct, no duplicate keys.

**Step 3: Commit**

```bash
git add warrior/data.lua
git commit -m "feat(warrior): add Shield Wall and Last Stand rage costs for /idefend"
```

---

## Task 2: Add settings defaults and configuration

**Files:**
- Modify: `warrior/event.lua:14-30` (add default settings in ADDON_LOADED)
- Modify: `warrior/setup.lua` (add validation + assignment for `shield` and `laststand`)

**Step 1: Add setting defaults in event.lua**

After line 30 (`if IWin_Settings["burst"] == nil then IWin_Settings["burst"] = "on" end`), add:

```lua
		if IWin_Settings["shield"] == nil then IWin_Settings["shield"] = "" end
		if IWin_Settings["laststand"] == nil then IWin_Settings["laststand"] = 50 end
		if IWin_Settings["savedMH"] == nil then IWin_Settings["savedMH"] = "" end
		if IWin_Settings["savedOH"] == nil then IWin_Settings["savedOH"] = "" end
```

**Step 2: Add validation block in setup.lua**

In the first `if/elseif` chain (validation section, lines 11-90), add before the closing `end` on line 90:

```lua
	elseif arguments[1] == "shield" then
		if arguments[2] == nil then
			DEFAULT_CHAT_FRAME:AddMessage("|cff0066ff Unknown parameter. Provide a shield name. Example: /iwin shield Draconian Deflector|r")
			return
		end
	elseif arguments[1] == "laststand" then
		if arguments[2] ~= nil
			and (tonumber(arguments[2]) == nil or tonumber(arguments[2]) < 0 or tonumber(arguments[2]) > 100) then
				DEFAULT_CHAT_FRAME:AddMessage("|cff0066ff Unknown parameter. Possible values: 0-100 (percent HP).|r")
				return
		end
```

**Step 3: Add assignment block in setup.lua**

In the second `if/elseif` chain (assignment section, lines 92-139), add before the `else` on line 125:

```lua
	elseif arguments[1] == "shield" then
		-- Rebuild the shield name from all remaining args (handles multi-word names)
		local shieldName = arguments[2]
		for i = 3, table.getn(arguments) do
			shieldName = shieldName .. " " .. arguments[i]
		end
		IWin_Settings["shield"] = shieldName
		DEFAULT_CHAT_FRAME:AddMessage("|cff0066ff Shield: |r" .. IWin_Settings["shield"])
	elseif arguments[1] == "laststand" then
		IWin_Settings["laststand"] = tonumber(arguments[2])
		DEFAULT_CHAT_FRAME:AddMessage("|cff0066ff Last Stand threshold: |r" .. tostring(IWin_Settings["laststand"]) .. "%")
```

**Step 4: Add help text in setup.lua**

In the help text `else` block (lines 125-139), add before the closing `end`:

```lua
		DEFAULT_CHAT_FRAME:AddMessage("|cff0066ff /iwin shield [|r" .. IWin_Settings["shield"] .. "|cff0066ff]:|r Shield name for /idefend")
		DEFAULT_CHAT_FRAME:AddMessage("|cff0066ff /iwin laststand [|r" .. tostring(IWin_Settings["laststand"]) .. "|cff0066ff]:|r HP% threshold for Last Stand in /idefend")
```

**Step 5: Commit**

```bash
git add warrior/event.lua warrior/setup.lua
git commit -m "feat(warrior): add /iwin shield and /iwin laststand settings for /idefend"
```

---

## Task 3: Add action functions for shield equip, defensive cooldowns, and swap-back

**Files:**
- Modify: `warrior/action.lua` (add new functions after the existing `ShieldSlam` function, ~line 955)

**Step 1: Add `SaveDualWieldWeapons()` function**

Reads currently equipped MH (slot 16) and OH (slot 17) item names and saves to `IWin_Settings`.

```lua
function IWin:SaveDualWieldWeapons()
	local mhLink = GetInventoryItemLink("player", 16)
	local ohLink = GetInventoryItemLink("player", 17)
	if mhLink then
		local mhName = GetItemInfo(tonumber(IWin:GetItemID(mhLink)))
		if mhName then
			IWin_Settings["savedMH"] = mhName
		end
	end
	if ohLink then
		local ohName = GetItemInfo(tonumber(IWin:GetItemID(ohLink)))
		if ohName then
			IWin_Settings["savedOH"] = ohName
		end
	end
end
```

**Step 2: Add `EquipShield()` function**

Saves current DW weapons, then equips configured shield. Sets GCD lock.

```lua
function IWin:EquipShield()
	if IWin_Settings["shield"] == "" then
		DEFAULT_CHAT_FRAME:AddMessage("|cff0066ff /idefend: No shield configured. Use /iwin shield <name>|r")
		return
	end
	if not IWin:IsShieldEquipped() then
		IWin:SaveDualWieldWeapons()
		IWin_CombatVar["queueGCD"] = false
		EquipItemByName(IWin_Settings["shield"])
	end
end
```

**Step 3: Add `ReequipDualWield()` function**

Re-equips saved DW weapons.

```lua
function IWin:ReequipDualWield()
	if IWin_Settings["savedMH"] ~= "" then
		EquipItemByName(IWin_Settings["savedMH"], 16)
	end
	if IWin_Settings["savedOH"] ~= "" then
		EquipItemByName(IWin_Settings["savedOH"], 17)
	end
	IWin_CombatVar["queueGCD"] = false
end
```

**Step 4: Add `DefensiveStanceDefend()` function**

Simplified stance swap — just Defensive Stance if not already there.

```lua
function IWin:DefensiveStanceDefend()
	if IWin:IsSpellLearnt("Defensive Stance")
		and not IWin:IsStanceActive("Defensive Stance") then
			CastSpellByName("Defensive Stance")
	end
end
```

**Step 5: Add `LastStandDefend()` function**

Casts Last Stand if learned, off cooldown, buff not active, and HP below threshold.

```lua
function IWin:LastStandDefend()
	if IWin:IsSpellLearnt("Last Stand")
		and IWin_CombatVar["queueGCD"]
		and not IWin:IsOnCooldown("Last Stand")
		and not IWin:IsBuffActive("player", "Last Stand")
		and (UnitHealth("player") / UnitHealthMax("player") * 100) < IWin_Settings["laststand"] then
			IWin_CombatVar["queueGCD"] = false
			CastSpellByName("Last Stand")
	end
end
```

**Step 6: Add `ShieldWallDefend()` function**

Casts Shield Wall if learned, off cooldown, shield equipped, in Defensive Stance, buff not active.

```lua
function IWin:ShieldWallDefend()
	if IWin:IsSpellLearnt("Shield Wall")
		and IWin_CombatVar["queueGCD"]
		and not IWin:IsOnCooldown("Shield Wall")
		and IWin:IsShieldEquipped()
		and IWin:IsStanceActive("Defensive Stance")
		and not IWin:IsBuffActive("player", "Shield Wall") then
			IWin_CombatVar["queueGCD"] = false
			CastSpellByName("Shield Wall")
	end
end
```

**Step 7: Commit**

```bash
git add warrior/action.lua
git commit -m "feat(warrior): add /idefend action functions (shield equip, Shield Wall, Last Stand, swap back)"
```

---

## Task 4: Add the `/idefend` slash command rotation

**Files:**
- Modify: `warrior/rotation.lua` (add new slash command after the existing `/ifuryprotaoe` block, ~line 304)

**Step 1: Add the `/idefend` slash command**

3-phase deterministic command with shift modifier for swap-back:

```lua
SLASH_IDEFENDWARRIOR1 = "/idefend"
function SlashCmdList.IDEFENDWARRIOR()
	IWin:InitializeRotation()
	-- Phase 3: Shift held + shield equipped → swap back to DW
	if IsShiftKeyDown() and IWin:IsShieldEquipped() then
		IWin:ReequipDualWield()
		return
	end
	-- Phase 1: Shield not equipped → equip shield + Defensive Stance
	if not IWin:IsShieldEquipped() then
		IWin:EquipShield()
		IWin:DefensiveStanceDefend()
		return
	end
	-- Phase 2: Shield equipped → cast defensive cooldowns (one per press)
	-- Last Stand takes priority when HP is low
	IWin:LastStandDefend()
	IWin:ShieldWallDefend()
	IWin:LastStandDefend()
end
```

Note: `LastStandDefend()` appears twice — first with HP threshold check (high priority), then after Shield Wall (normal priority when HP is fine). The second call only fires if Shield Wall didn't consume the GCD and Last Stand is off cooldown with HP above threshold... actually, the second `LastStandDefend()` has the HP threshold baked in, so it will only fire when HP < threshold regardless. We need a separate `LastStandDefendNormal()` without the HP check for the normal-priority position. See corrected version:

```lua
SLASH_IDEFENDWARRIOR1 = "/idefend"
function SlashCmdList.IDEFENDWARRIOR()
	IWin:InitializeRotation()
	-- Phase 3: Shift held + shield equipped → swap back to DW
	if IsShiftKeyDown() and IWin:IsShieldEquipped() then
		IWin:ReequipDualWield()
		return
	end
	-- Phase 1: Shield not equipped → equip shield + Defensive Stance
	if not IWin:IsShieldEquipped() then
		IWin:EquipShield()
		IWin:DefensiveStanceDefend()
		return
	end
	-- Phase 2: Shield equipped → cast defensive cooldowns (one per press)
	-- Last Stand jumps to top priority when HP is low
	IWin:LastStandDefend()
	-- Shield Wall is default first cooldown
	IWin:ShieldWallDefend()
	-- Last Stand at normal priority (when HP is above threshold but CD is available)
	IWin:LastStandDefendNormal()
end
```

This requires adding `LastStandDefendNormal()` to action.lua (same as `LastStandDefend` but without the HP threshold check):

```lua
function IWin:LastStandDefendNormal()
	if IWin:IsSpellLearnt("Last Stand")
		and IWin_CombatVar["queueGCD"]
		and not IWin:IsOnCooldown("Last Stand")
		and not IWin:IsBuffActive("player", "Last Stand") then
			IWin_CombatVar["queueGCD"] = false
			CastSpellByName("Last Stand")
	end
end
```

**Step 2: Commit**

```bash
git add warrior/rotation.lua warrior/action.lua
git commit -m "feat(warrior): add /idefend slash command with 3-phase modifier-key toggle"
```

---

## Task 5: Update README with new commands

**Files:**
- Modify: `README.md` (add `/idefend`, `/iwin shield`, `/iwin laststand` documentation)

**Step 1: Add `/idefend` to slash command list**

Find the warrior slash commands section and add:

```
- `/idefend` — Defensive shield swap (press: equip shield + cooldowns, shift+press: swap back to DW)
```

**Step 2: Add settings to the `/iwin` settings section**

```
- `/iwin shield <name>` — Shield name for `/idefend` (e.g. `/iwin shield Draconian Deflector`)
- `/iwin laststand <0-100>` — HP% threshold for Last Stand priority in `/idefend` (default: 50)
```

**Step 3: Commit**

```bash
git add README.md
git commit -m "docs: add /idefend command and settings to README"
```

---

## Task 6: In-game verification

**No files modified — manual testing.**

After implementation, test in-game with `/reload`:

1. **Configure shield**: `/iwin shield <your shield name>` — verify it echoes back the name
2. **Configure threshold**: `/iwin laststand 40` — verify echo
3. **Check help**: `/iwin` — verify new settings appear in help text with current values
4. **No-config error**: Clear shield with `/iwin shield`, press `/idefend` — verify error message
5. **Phase 1 — equip**: Set shield name, press `/idefend` — verify shield equips to slot 17
6. **Phase 1 — stance**: Press `/idefend` again — verify Defensive Stance swap (if not already in it)
7. **Phase 2 — Shield Wall**: Press `/idefend` with shield equipped + Defensive Stance — verify Shield Wall casts
8. **Phase 2 — Last Stand (normal)**: Press `/idefend` again — verify Last Stand casts (HP above threshold)
9. **Phase 2 — Last Stand (priority)**: Damage below threshold, press `/idefend` — verify Last Stand fires BEFORE Shield Wall
10. **Phase 2 — nothing**: All CDs active/on CD, press `/idefend` — verify nothing happens
11. **Phase 3 — swap back**: Hold shift + press `/idefend` — verify DW weapons re-equip
12. **Rotation preservation**: While shield equipped, press `/ifuryprot` — verify shield stays
13. **Persistence**: `/reload`, then `/iwin` — verify shield name and saved weapons persisted

---

## File Change Summary

| File | Change Type | What Changes |
|------|------------|--------------|
| `warrior/data.lua` | Add 2 lines | `IWin_RageCost` entries for Shield Wall (0) and Last Stand (0) |
| `warrior/event.lua` | Add 4 lines | Default settings: shield, laststand, savedMH, savedOH |
| `warrior/setup.lua` | Add ~20 lines | Validation + assignment for shield/laststand settings + help text |
| `warrior/action.lua` | Add ~65 lines | 7 new functions: SaveDualWieldWeapons, EquipShield, ReequipDualWield, DefensiveStanceDefend, LastStandDefend, ShieldWallDefend, LastStandDefendNormal |
| `warrior/rotation.lua` | Add ~18 lines | `/idefend` slash command with 3-phase shift-modifier toggle |
| `README.md` | Add ~5 lines | Command and settings documentation |
| `warrior/init.lua` | No changes | No new combat vars needed (state is implicit) |
| `warrior/condition.lua` | No changes | Existing utilities suffice |
