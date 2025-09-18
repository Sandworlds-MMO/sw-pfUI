# Nameplates Auto-Hide Feature

## Overview

Implement a feature that automatically hides nameplates when not in combat and shows them only when the player enters battle. This applies to both hostile and friendly nameplates.

## Why This Feature Exists

Nameplates are very useful but pollute the game, especially in big cities, and take away immersion. This change makes the auto-hide feature enabled by default using the new custom Sandworlds theme.

Those who don't like it can disable it in settings under **Nameplates → Auto Show/Hide Nameplates When Entering/Leaving Combat**.

## What We're Trying to Do

Currently, nameplates in pfUI are controlled by simple on/off settings:
- **Hostile nameplates**: Controlled by `showhostile` setting
- **Friendly nameplates**: Controlled by `showfriendly` setting

The goal is to make these settings work **only in combat**, so nameplates automatically hide when out of combat and show when entering battle.

## Current System Analysis

### Nameplate Visibility Controls
**File**: `modules/nameplates.lua` (lines 1079-1096)

Current implementation:
```lua
nameplates.SetGameVariables = function()
  -- update visibility (hostile)
  if C.nameplates["showhostile"] == "1" then
    _G.NAMEPLATES_ON = true
    ShowNameplates()
  else
    _G.NAMEPLATES_ON = nil
    HideNameplates()
  end

  -- update visibility (friendly)
  if C.nameplates["showfriendly"] == "1" then
    _G.FRIENDNAMEPLATES_ON = true
    ShowFriendNameplates()
  else
    _G.FRIENDNAMEPLATES_ON = nil
    HideFriendNameplates()
  end
end
```

### Battle Detection System
**File**: `modules/infight.lua`

pfUI already has robust battle detection:
- **Core function**: `UnitAffectingCombat("player")`
- **Events**: `PLAYER_ENTER_COMBAT` and `PLAYER_LEAVE_COMBAT`
- **Red screen effect**: Screen edge glow when in combat

Detection logic:
```lua
if this.infight and UnitAffectingCombat("player") then visible = true end
```

### Combat Events Already Available
**File**: `modules/nameplates.lua` (lines 348-360)

The nameplate module already has combat tracking:
```lua
-- combat tracker
nameplates.combat = CreateFrame("Frame")
nameplates.combat:RegisterEvent("PLAYER_ENTER_COMBAT")
nameplates.combat:RegisterEvent("PLAYER_LEAVE_COMBAT")
nameplates.combat:SetScript("OnEvent", function()
  if event == "PLAYER_ENTER_COMBAT" then
    this.inCombat = 1
    if PlayerFrame then PlayerFrame.inCombat = 1 end
  elseif event == "PLAYER_LEAVE_COMBAT" then
    this.inCombat = nil
    if PlayerFrame then PlayerFrame.inCombat = nil end
  end
end)
```

## Implementation Plan

### Step 1: Modify SetGameVariables Function
**File**: `modules/nameplates.lua`

Replace the current `SetGameVariables` function with combat-aware logic:

```lua
nameplates.SetGameVariables = function()
  local inCombat = nameplates.combat.inCombat == 1  -- Use existing combat state
  local combatOnly = C.nameplates["display_nameplates_combat_only"] == "1"
  
  -- update visibility (hostile)
  if C.nameplates["showhostile"] == "1" and (not combatOnly or inCombat) then
    _G.NAMEPLATES_ON = true
    ShowNameplates()
  else
    _G.NAMEPLATES_ON = nil
    HideNameplates()
  end

  -- update visibility (friendly)
  if C.nameplates["showfriendly"] == "1" and (not combatOnly or inCombat) then
    _G.FRIENDNAMEPLATES_ON = true
    ShowFriendNameplates()
  else
    _G.FRIENDNAMEPLATES_ON = nil
    HideFriendNameplates()
  end
end
```

### Step 2: Add Dynamic Updates on Combat State Change
**File**: `modules/nameplates.lua`

Modify the existing combat event handler to update nameplate visibility:

```lua
nameplates.combat:SetScript("OnEvent", function()
  if event == "PLAYER_ENTER_COMBAT" then
    this.inCombat = 1
    if PlayerFrame then PlayerFrame.inCombat = 1 end
  elseif event == "PLAYER_LEAVE_COMBAT" then
    this.inCombat = nil
    if PlayerFrame then PlayerFrame.inCombat = nil end
  end
  
  -- Update nameplate visibility on combat state change
  if C.nameplates["display_nameplates_combat_only"] == "1" then
    nameplates:SetGameVariables()
  end
end)
```

### Step 3: Optional - Add Configuration Option
**File**: `modules/gui.lua`

Add a new checkbox setting with clear description:
```lua
CreateConfig(U["nameplates"], T["Auto Show/Hide Nameplates When Entering/Leaving Combat"], C.nameplates, "display_nameplates_combat_only", "checkbox")
```

### Step 4: Update Configuration Defaults
**File**: `api/config.lua`

Set default values:
```lua
pfUI:UpdateConfig("nameplates", nil, "display_nameplates_combat_only", "1")
```

## Files to Modify

1. **`modules/nameplates.lua`**
   - Modify `SetGameVariables` function (around line 1078)
   - Update combat event handler (around line 352)

2. **`modules/gui.lua`** (optional)
   - Add new configuration option in nameplate settings section

3. **`api/config.lua`** (optional)
   - Add default configuration value

4. **`env/translations_enUS.lua`** (optional)
   - Add translation: `["Auto Show/Hide Nameplates When Entering/Leaving Combat"] = nil,`

## Testing Strategy

1. **Out of Combat**: Verify nameplates are hidden
2. **Entering Combat**: Verify nameplates appear when combat starts
3. **Leaving Combat**: Verify nameplates hide when combat ends
4. **Settings Respect**: Verify existing `showhostile` and `showfriendly` settings still work
5. **Performance**: Ensure no performance impact from frequent combat state checks

## Benefits

- **Cleaner UI**: No nameplate clutter when exploring
- **Better Performance**: Fewer UI elements to render out of combat
- **Combat Focus**: Nameplates appear exactly when needed for combat
- **Backward Compatible**: Existing settings still work, just with combat restriction

## Feasibility Assessment

✅ **HIGHLY FEASIBLE**

- Battle detection system already exists and is robust
- Nameplate visibility controls are well-structured
- Combat events are already being tracked
- Minimal code changes required
- No breaking changes to existing functionality
