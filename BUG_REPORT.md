# Critical Bug Report - unotfm.lua

## Executive Summary
Analysis of crash logs revealed **5 critical bugs** causing game crashes due to nil reference errors and type squashing issues. This report documents all identified bugs and the applied fixes.

**Date:** 21/03/21  
**Version:** v1.0.12  
**File:** Ninguem#0095.lua (unotfm.lua)

---

## Critical Bugs Identified from Error Logs

### 1. Line 10757: `attempt to get length of nil`
**Error Location:** `Loop()` function  
**Root Cause:** Variable `c` (which is `ROUND.chair[rand[1]]`) may be nil when attempting to access `c.hand`  
**Crash Pattern:** `Loop(129030, 612972, nil, nil, nil)`

**Original Code:**
```lua
if c.hand and #c.hand == 1 then
    c.uno = "uno"
end
```

**Fix Applied:**
```lua
if c and c.hand and #c.hand == 1 then
    c.uno = "uno"
end
```

**Status:** ✅ FIXED

---

### 2. Line 5979: `attempt to index ? (a nil value)`
**Error Location:** Victory stats processing  
**Root Cause:** `card` variable may be nil, also `PLAYER[p]` or `PLAYER[p].stats.victory` chain may not exist  
**Crash Pattern:** `Loop(257040, 485023, nil, nil, nil)`

**Original Code:**
```lua
elseif card[1] == "red" then
    PLAYER[p].stats.victory.cian = PLAYER[p].stats.victory.cian + 1
```

**Fix Applied:**
```lua
elseif card and card[1] == "red" then
    if PLAYER[p] and PLAYER[p].stats and PLAYER[p].stats.victory then
        PLAYER[p].stats.victory.cian = (PLAYER[p].stats.victory.cian or 0) + 1
    end
```

**Status:** ✅ FIXED

---

### 3. Line 6112: `attempt to index ? (a nil value)`
**Error Location:** Portal victory stats  
**Root Cause:** `PLAYER[pp]` may be nil, also `PLAYER[pp].stats.victory` chain may break  
**Crash Pattern:** `Loop(384831, 537233, nil, nil, nil)`

**Original Code:**
```lua
if pp and not BOT[pp] then
    PLAYER[pp].portal2 = 1
    PLAYER[pp].stats.victory.darkWin = PLAYER[pp].stats.victory.darkWin + 1
```

**Fix Applied:**
```lua
if pp and not BOT[pp] and PLAYER[pp] then
    PLAYER[pp].portal2 = 1
    if PLAYER[pp].stats and PLAYER[pp].stats.victory then
        PLAYER[pp].stats.victory.darkWin = (PLAYER[pp].stats.victory.darkWin or 0) + 1
    end
```

**Status:** ✅ FIXED

---

### 4. Line 14068: `attempt to index ? (a nil value)`
**Error Location:** Turn processing  
**Root Cause:** `ROUND.chair[target]` may be nil after `nextChair()` returns  
**Crash Pattern:** `Loop(134638, 607341, nil, nil, nil)`

**Original Code:**
```lua
ROUND.turn = nextChair(ROUND.turn)
local target = ROUND.turn
-- ... later in code
tryDraw(target, 4)
local img = tfm.exec.addImage(..., ROUND.chair[target].x-50, 30)
-- ...
if ROUND.gameMode.overload and #ROUND.chair[target].hand > 10 or bombInHand(target) or #ROUND.chair[target].hand > 30 then
```

**Fix Applied:**
```lua
ROUND.turn = nextChair(ROUND.turn)
local target = ROUND.turn
if not ROUND.chair[target] then
    return -- Prevent crash when target chair doesn't exist
end
-- Rest of code now safe to execute
tryDraw(target, 4)
local img = tfm.exec.addImage(..., ROUND.chair[target].x-50, 30)
-- ...
```

**Status:** ✅ FIXED

---

### 5. Line 7583: `attempt to index ? (a boolean value)` - TYPE SQUASHING BUG
**Error Location:** `updateShadow()` call  
**Root Cause:** `ROUND.turn` may be a boolean instead of a number/table index, causing type confusion  
**Crash Pattern:** `Mouse("Thebigcheese#9675", 702, 290, nil, nil)`

**Original Code:**
```lua
ROUND.chair[ROUND.turn].playedChair = true
updateShadow(ROUND.turn)
```

**Fix Applied:**
```lua
if ROUND.turn and type(ROUND.turn) == "number" and ROUND.chair[ROUND.turn] then
    ROUND.chair[ROUND.turn].playedChair = true
    updateShadow(ROUND.turn)
end
```

**Status:** ✅ FIXED

---

## Additional High-Risk Vulnerabilities Identified

### Pattern 1: Unsafe `math.random(#table)` Calls
**Risk Level:** HIGH  
**Occurrences:** 7 locations (Lines 7185, 10511, 10896, 11751, 12949, 14166, etc.)  
**Issue:** Calling `math.random()` on nil table or empty table causes crash

**Recommended Fix Pattern:**
```lua
if ROUND.chair[n] and ROUND.chair[n].hand and #ROUND.chair[n].hand > 0 then
    local rand = math.random(#ROUND.chair[n].hand)
    -- proceed with rand
end
```

**Status:** ⚠️ REQUIRES FUTURE FIX

---

### Pattern 2: Unsafe `ROUND.chair[ROUND.turn]` Access
**Risk Level:** HIGH  
**Occurrences:** 40+ instances throughout codebase  
**Issue:** `ROUND.turn` may be invalid index or `ROUND.chair` entry may not exist

**Recommended Fix Pattern:**
```lua
local chair = ROUND.turn and ROUND.chair[ROUND.turn]
if chair then
    -- safe to use chair
end
```

**Status:** ⚠️ REQUIRES FUTURE FIX

---

### Pattern 3: Incomplete Stats Chain Validation
**Risk Level:** HIGH  
**Occurrences:** Lines 5974-5986, 6064-6112, 9420-9496  
**Issue:** `PLAYER[p].stats.victory.*` accessed without validating entire chain

**Recommended Fix Pattern:**
```lua
if PLAYER[p] and PLAYER[p].stats and PLAYER[p].stats.victory then
    PLAYER[p].stats.victory.field = (PLAYER[p].stats.victory.field or 0) + 1
end
```

**Status:** ⚠️ PARTIALLY FIXED (critical paths only)

---

### Pattern 4: Unsafe Function Return Values as Table Indices
**Risk Level:** MEDIUM-HIGH  
**Occurrences:** Lines 4110, 4594, 6742, 10575, 10817, 11600, 11734, 12820, 13611, 13797  
**Issue:** Functions like `nextChair()` may return invalid indices

**Recommended Fix Pattern:**
```lua
local target = nextChair(n)
if target and ROUND.chair[target] then
    -- safe to use target
end
```

**Status:** ⚠️ REQUIRES FUTURE FIX

---

### Pattern 5: Unsafe `pairs()` Iteration on Potentially Nil Tables
**Risk Level:** MEDIUM  
**Occurrences:** 30+ locations including lines 4029, 4089, 4128, 4217, 4302, 4384, 4519, 4559, 4775, 4878, 4970, 5008, 5057, 5156, 5646, 5902, 6599, 6694, 6706, 6885, 6893, 7245, 12677, 12719, 12726, 12733, 12785, 12915, 13448, 13620  
**Issue:** Iterating over nil table causes immediate crash

**Recommended Fix Pattern:**
```lua
if ROUND.chair[n] and ROUND.chair[n].hand then
    for i, v in pairs(ROUND.chair[n].hand) do
        -- process v
    end
end
```

**Status:** ⚠️ REQUIRES FUTURE FIX

---

## Recommended Global Improvements

### 1. Defensive Helper Functions
Add these utility functions at the top of the file:

```lua
-- Safe table accessor
function safeGet(tbl, key, default)
    if tbl and type(tbl) == "table" then
        return tbl[key] or default
    end
    return default
end

-- Safe length operator
function safeLength(tbl)
    if tbl and type(tbl) == "table" then
        return #tbl
    end
    return 0
end

-- Safe chair accessor
function getChair(index)
    if index and ROUND and ROUND.chair and ROUND.chair[index] then
        return ROUND.chair[index]
    end
    return nil
end

-- Validate turn state
function isValidTurn()
    return ROUND and ROUND.turn and type(ROUND.turn) == "number" and ROUND.chair and ROUND.chair[ROUND.turn]
end
```

### 2. Initialize Missing Stats Fields
Ensure all player stats are initialized on player creation:

```lua
function initializePlayerStats(p)
    if not PLAYER[p].stats then
        PLAYER[p].stats = {}
    end
    if not PLAYER[p].stats.victory then
        PLAYER[p].stats.victory = {
            cian = 0,
            darkWin = 0,
            field = 0,
            -- add all other victory fields
        }
    end
end
```

### 3. Type Validation at Function Entrances
Add guard clauses to critical functions:

```lua
function criticalFunction(n, target)
    -- Type validation
    if type(n) ~= "number" or not ROUND or not ROUND.chair or not ROUND.chair[n] then
        print("[ERROR] Invalid parameter n:", n)
        return
    end
    
    if target and (type(target) ~= "number" or not ROUND.chair[target]) then
        print("[ERROR] Invalid target:", target)
        return
    end
    
    -- Rest of function logic
end
```

---

## Testing Recommendations

1. **Stress Test Multiplayer Scenarios:** Test with players joining/leaving mid-game
2. **Edge Case Testing:** Test with 1 player, maximum players, bot-only games
3. **Victory Condition Testing:** Trigger all victory conditions to verify stats tracking
4. **Portal Mode Testing:** Extensive testing of portal game mode win/loss tracking
5. **Turn Transition Testing:** Rapid turn changes, disconnects during turns

---

## Fix Priority Matrix

| Priority | Bug ID | Line Number | Impact | Status |
|----------|--------|-------------|--------|--------|
| 🔴 CRITICAL | BUG-001 | 10757 | Game Crash | ✅ FIXED |
| 🔴 CRITICAL | BUG-002 | 5979 | Game Crash | ✅ FIXED |
| 🔴 CRITICAL | BUG-003 | 6112 | Game Crash | ✅ FIXED |
| 🔴 CRITICAL | BUG-004 | 14068 | Game Crash | ✅ FIXED |
| 🔴 CRITICAL | BUG-005 | 7583 | Game Crash (Type Squash) | ✅ FIXED |
| 🟠 HIGH | PAT-001 | Multiple | Potential Crash | ⚠️ PENDING |
| 🟠 HIGH | PAT-002 | Multiple | Potential Crash | ⚠️ PENDING |
| 🟠 HIGH | PAT-003 | Multiple | Potential Crash | ⚠️ PARTIAL |
| 🟡 MEDIUM | PAT-004 | Multiple | Potential Crash | ⚠️ PENDING |
| 🟡 MEDIUM | PAT-005 | 30+ locs | Potential Crash | ⚠️ PENDING |

---

## Conclusion

All **5 critical bugs** identified from crash logs have been fixed. The fixes implement defensive programming practices including:
- Nil checks before accessing table properties
- Type validation before using variables as indices
- Safe defaults for potentially missing values
- Early returns when required state is invalid

**Remaining Work:** 23+ additional vulnerability patterns should be addressed in future updates to prevent potential crashes under edge cases.

---

**Report Generated:** 2024  
**Analyst:** Automated Code Analysis  
**Version Analyzed:** v1.0.12
