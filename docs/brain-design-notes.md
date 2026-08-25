# ArtificialWilson Brain — Design Study Notes

Notes from reading through `scripts/brains/` and `scripts/behaviours/` in this repo, plus one piece
of git-history archaeology on `upstream/feature/FixStuckWilson`. Written for studying the design,
not as documentation of guaranteed-current behavior — file/line references may drift as the mod
changes.

## Table of contents

1. [Root PriorityNode](#1-root-prioritynode)
2. [Not chopping without an axe](#2-not-chopping-without-an-axe)
3. [Targeting trees/bushes and avoiding re-targeting](#3-targeting-treesbushes-and-avoiding-re-targeting)
4. [Exploration / wandering](#4-exploration--wandering)
5. [Prioritizer component](#5-prioritizer-component)
6. [Cartographer component (unused)](#6-cartographer-component-unused)
7. [Tool-equip check: findtreeorrock vs findresourcetoharvest](#7-tool-equip-check-findtreeorrock-vs-findresourcetoharvest)
8. [findresourcetoharvest.lua: avoiding re-targeting](#8-findresourcetoharvestlua-avoiding-re-targeting)
9. [Light and hunger thresholds, and nightfall lookahead](#9-light-and-hunger-thresholds-and-nightfall-lookahead)
10. [git history: upstream/feature/FixStuckWilson](#10-git-history-upstreamfeaturefixstuckwilson)

---

## 1. Root PriorityNode

Source: `scripts/brains/artificialwilson.lua:801-878`

A `PriorityNode` re-checks its children from the top every tick (here, every 0.25s) and runs the
**first one whose guard is currently true**. List order *is* priority order — item 1 always wins
over item 14 if both are eligible at once.

| # | Behaviour | Guard condition |
|---|-----------|------------------|
| 1 | Fix stuck state (`FixStuckWilson`) | `inst:HasTag("IsStuck")` — set when pathfinding fails |
| 2 | Drop the "active" inventory item | `inventory.activeitem ~= nil` (leftover overflow item) |
| 3 | `DontBeOnFire` | always eligible (checks fire internally) |
| 4 | `MaintainLightSource(30)` | `clock.isnight` |
| 5 | `ChaseAndAttack` ("go for the eyes") | `not clock.isnight and GoForTheEyes(self.inst)` |
| 6 | `RunAway` from dangerous mobs | `ShouldRunAway(guy)` true for something within 5 units |
| 7 | `ManageHealth(.75)` | `not IsBusy(self.inst)` |
| 8 | `ManageSanity(.9, .75)` | `not IsBusy(self.inst)` |
| 9 | `DoScience` (prototype/build queued item) | always eligible (no-ops internally if nothing queued) |
| 10 | Nested `PriorityNode { MasterChef, ManageInventory }` | throttled — runs at most every 2.5s |
| 11 | `day` | `clock.isday` |
| 12 | `dusk` | `clock.isdusk and (not MidwayThroughDusk() or not HasValidHome)` |
| 13 | `dusk2` | `clock.isdusk and MidwayThroughDusk() and HasValidHome` |
| 14 | `night` | `clock.isnight` |

**Plain-language summary:** survival reflexes come first (unstuck → don't burn → stay lit →
fight/flee threats), then upkeep (health, sanity, crafting, cooking/inventory on a slower cadence),
and only at the bottom does it fall into whichever time-of-day routine matches the clock — day/dusk
are gathering-focused, dusk2/night shift toward going home and cooking. Because it's a priority
list, a monster attack or catching fire will always interrupt wood-chopping, but wood-chopping will
never interrupt an "am I on fire" check.

---

## 2. Not chopping without an axe

Source: `scripts/behaviours/findtreeorrock.lua:79-98` and `:196-239`

```lua
function FindTreeOrRock:FindAndEquipRightTool()
    local equiped = self.inst.components.inventory:GetEquippedItem(EQUIPSLOTS.HANDS)
    if equiped and equiped.components.tool and equiped.components.tool:CanDoAction(self.actionType) then
        return true, equiped
    else
        tool = self.inst.components.inventory:FindItem(function(item)
            return item.components.equippable and item.components.tool
                and item.components.tool:CanDoAction(self.actionType) end)
    end
    if tool then
        self.inst.components.inventory:Equip(tool)
        return true, tool
    end
    return false, nil
end
```

1. Checks what's equipped in the hands slot; reuses it if it's a valid tool for `self.actionType`
   (axe for `CHOP`, pickaxe for `MINE`).
2. Otherwise searches the whole inventory for an equippable tool that can do this action, and
   equips it.
3. If nothing found, returns `false` — the caller falls into a **craft** branch instead of
   swinging bare-handed:

```lua
if haveRightTool then
    ...
else
    local thingToBuild = nil
    if self.actionType == ACTIONS.CHOP then thingToBuild = "axe"
    elseif self.actionType == ACTIONS.MINE then thingToBuild = "pickaxe" end
    ...
    if thingToBuild and self.inst.components.builder:CanBuild(thingToBuild) then
        -- queue a BUILD action for the axe/pickaxe instead
    else
        self.status = FAILED  -- can't get a tool, give up on this target
    end
end
```

No tool → try to craft one → if it can't even craft one, the node just fails rather than attacking
bare-handed. It also re-checks tool validity every tick while chopping is in progress
(lines 270, 293-294), and treats a tool becoming invalid mid-action as `SUCCESS` (assumes the
target died).

---

## 3. Targeting trees/bushes and avoiding re-targeting

Sources: `findtreeorrock.lua:100-168` (trees/rocks), `findresourcetoharvest.lua:46-82` (pickables)

Both call `FindEntity(inst, searchDistance, filterFn)` — the engine's nearest-entity search. The
filter rejects:

- anything not currently workable/pickable right now (`workable:CanBeWorked()` /
  `pickable:CanBePicked()`)
- anything on the prioritizer's ignore list (by prefab or GUID)
- anything near a hostile mob (`HostileMobNearInst`)
- anything whose loot it doesn't actually want (already has a full stack, etc.)

**Avoiding re-targeting the thing it just harvested is essentially free — it relies on game state,
not extra bookkeeping.** Once a tree is chopped down or a bush is picked, its own
`workable`/`pickable` component reports it can't be worked/picked again (chopped tree becomes a
stump/falls, picked bush needs to regrow), so the very next `GetTarget()` scan simply skips it.

The separate **ignore list** (`components/prioritizer.lua`) is for a different failure mode: things
that are workable/pickable but unreachable or otherwise bad. Populated via `AddToIgnoreList(prefab,
pos)`, notably from the `noPathFound` handler (`artificialwilson.lua:315-326`) — if pathfinding
fails, that target's GUID gets ignored *near the current position* (`OnIgnoreList` checks distance
against `try_ignore_dist = 30`, so it's "don't retry within 30 units of here," not a global
blacklist unless `always` is set). A handful of prefabs (seeds, evil petals, marsh plants, pinecones,
etc.) are permanently ignored at startup (`artificialwilson.lua:645-656`).

---

## 4. Exploration / wandering

**This is the weakest part of the design — there's no actual "pick a direction and walk" logic.**
Two mechanisms exist, both inside the `day`/`dusk` `SelectorNode`s
(`artificialwilson.lua:683-702`):

**a) Grow the search radius.** If pickup → harvest → chop → mine → burn all fail, the last
fallback is:

```lua
IfNode( function() return not IsBusy(self.inst) end, "nothing_to_do",
    NotDecorator(ActionNode(function() return self:IncreaseSearchDistance() end))),
```

`IncreaseSearchDistance` (`artificialwilson.lua:306-309`) bumps `CurrentSearchDistance` by
`SEARCH_SIZE_STEP` (10), capped at `MAX_SEARCH_DISTANCE` (100). Wrapped in `NotDecorator` so this
node always reports "failure." Wilson doesn't move here — he just widens the radius `FindEntity`
searches next tick, from wherever he's currently standing. `ResetSearchDistance` (back to
`MIN_SEARCH_DISTANCE = 10`) fires whenever something *is* found.

**b) Once maxed out, cheat toward a wormhole.**

```lua
local function FindSomewhereNewToGo(inst)
    local wormhole = FindEntity(inst,200,function(thing) return thing.prefab and thing.prefab == "wormhole" end)
    if wormhole then
        inst.components.locomotor:GoToEntity(wormhole,nil,true)
    end
end
```

The comment above it in the source is honest: *"Cheating for now. Find the closest wormhole and go
there... he'll hopefully find something else to do."* If no wormhole is within 200 units either,
this does nothing and Wilson just stands there re-running the maxed-out search every tick. A
`--Wander(self.inst, nil, 20)` line is commented out right above the `night` block, and there's a
`-- TODO: Need a good wander function for when searchdistance is at max` comment — a real
random-walk/explore behaviour was planned but never implemented.

---

## 5. Prioritizer component

Source: `scripts/components/prioritizer.lua`

Despite the name, **it doesn't compute "what's needed next."** It's a passive memory other code
reads from or writes to — not a decision-maker.

### `ignore_list` — actually wired up

- `AddToIgnoreList(prefab, fromPos)` — no position → permanently ignore that prefab type
  (`always = true`). With a position → records "the last attempt near here failed."
- `OnIgnoreList(prefab)` — true if `always` is set, **or** the character is within
  `try_ignore_dist` (30 units) of any position where it previously failed on that prefab.

Purely a negative/veto filter: never says what to do next, only excludes recently-bad candidates
near the current position (plus a permanent list: seeds, evil petals, cave entrances, etc., set up
in `artificialwilson.lua:645-656`).

### `build_order` / `build_parameters` — dead code

The shape is a real FIFO priority queue: `AddToBuildList(toBuild, build_params, topOfList)` /
`RemoveFromBuildList`. But the only caller anywhere in the codebase is commented out, in
`doscience.lua:112`:

```lua
--local build_info = {pos=self.buildTable.pos, onsuccess=self.buildTable.onsuccess, onfail=self.buildTable.onfail}
--self.inst.components.prioritizer:AddToBuildList(toBuild,build_info)
```

Nothing populates `build_order`; nothing reads it back. `DoScience` (the thing that actually decides
what to build) uses its own hardcoded `BUILD_PRIORITY` table instead
(`doscience.lua:61-66`: spear → backpack → firepit → cookpot).

### `OnUpdate` — also a no-op

```lua
function Prioritizer:OnUpdate(dt)
   if not self.inst.brain.HostileMobNearInst then return end
end
```

Runs every tick but only checks whether the function *exists* (always true), does nothing when
true, and there's no code after it either way. Leftover scaffolding.

**Bottom line:** `Prioritizer` is a blacklist with distance-based expiry, used to stop the character
repeatedly retrying the same unreachable/unwanted target. The build-queue half was scaffolded for a
real planner but never finished — `DoScience` bypassed it with a hardcoded list.

---

## 6. Cartographer component (unused)

Source: `scripts/components/cartographer.lua`

**Not actually active.** The only references to `cartographer` anywhere in the codebase are
commented out in `artificialwilson.lua`:

```lua
--self.inst:AddComponent("cartographer")
...
--self.inst:RemoveComponent("cartographer")
```

Same dead/unused pattern as `build_order` above. With that caveat, here's the design:

### What it remembers

A **graph of "circles"** — ~15-unit-radius disks (`circleRadius = 15`) dropped as the character
walks through unexplored ground. `circleMap` is a connectivity graph of explored territory. Per
circle:

- **`pos`** — world coordinate where dropped
- **`tiles`** — frequency count of ground types sampled at 16 perimeter points ("what terrain is
  this, mostly")
- **`linkedCircles`** — indices of connected circles (adjacency list — this is what makes it a
  graph)
- **`exitPoints`** — valid, walkable, non-overlapping points on the circle's edge — candidate
  directions toward unexplored space
- **`circle`** — the actual spawned `range_indicator` game entity for this node (doubles as the
  spatial index, see below)

`lastCircle` is a single pointer to whichever circle the character is currently/most-recently
standing in. `utilityMap` is declared and passed through save/load but never populated anywhere in
this file — an unused placeholder.

### How it's stored

Plain Lua tables keyed by integer index, persisted via `OnSave`/`OnLoad`. Since a `range_indicator`
entity can't be serialized, `OnSave` strips the `circle` field before writing; `OnLoad` re-spawns
each circle from its saved `pos`.

### How it's queried

**It doesn't query its own table by coordinates.** The physical `range_indicator` entities double
as the spatial index — queries go through the game engine's own spatial search
(`TheSim:FindEntities` / `FindEntity`) asking "what circle entities are near this point right now":

- `OnUpdate` (every 0.5s) — searches for a circle within `2*circleRadius`. If standing inside one,
  marks it `lastCircle`. If none found → new territory → samples 16 perimeter points for
  water/impassable tiles, and if valid, drops a new circle, links it to `lastCircle`, computes exit
  points.
- `GetIndexFromCircle(circle)` — reverse lookup, linear scan.
- `FindExitNodesFrom(index, exitPoints)` — searches for circles within `3*circleRadius`, checks
  actual walkability via `FindWalkableOffset` (won't link circles separated by a river), links them
  via `ConnectCircles`, prunes neighboring exit points that now fall inside the new circle.
- `CanPlaceCircleAtPoint` — validates a candidate spot against live world state before recording it.

**Plain-language summary:** a frontier-exploration graph — waypoints roughly every 15 units, each
remembering nearby terrain type and which other waypoints connect to it, tracking open "exit"
directions where the graph hasn't been extended. A plausible foundation for the exploration/wander
gap noted in section 4 — but currently disconnected; never added to the brain, so none of it runs.

---

## 7. Tool-equip check: findtreeorrock vs findresourcetoharvest

**`findresourcetoharvest.lua` doesn't check for a tool at all — because picking doesn't need one.**

### findtreeorrock.lua — full check (lines 79-98)

See section 2 above — checks equipped hands slot, searches inventory, equips if found, falls back
to crafting if not.

### findresourcetoharvest.lua — no check

No reference anywhere to `EQUIPSLOTS`, `tool`, or `Equip`. Goes straight from finding a target to
queueing the action:

```lua
local action = BufferedAction(self.inst,target,ACTIONS.PICK)
```

`ACTIONS.PICK` in Don't Starve is always bare-handed — no tool gates it, so there's nothing to
check.

**The difference reflects the underlying game action**, not a gap in one file — `findtreeorrock.lua`
gates on tool availability because `CHOP`/`MINE` genuinely require one (with a craft fallback);
`findresourcetoharvest.lua` has no equivalent because `PICK` never needed a tool. Both share the
same target-filtering pattern (ignore list, hostile-mob check, "do I want this loot" check) — only
the tool-gating step differs.

---

## 8. findresourcetoharvest.lua: avoiding re-targeting

Source: `findresourcetoharvest.lua:46-82`

It doesn't do anything special itself — leans entirely on the plant's own game state:

```lua
if item.components.pickable and item.components.pickable:CanBePicked() and item.components.pickable.caninteractwith then
```

`CanBePicked()` is the game's built-in `pickable` component (not written by this mod) — it tracks
whether that specific plant has already been harvested and is waiting to regrow. Once picked, it
flips to "can't be picked" until its regrowth timer elapses. The next `GetTarget()` scan fails this
check immediately and skips it — no bookkeeping needed on the AI's side.

Everything else in the filter is a different kind of check:
- `prioritizer:OnIgnoreList(...)` — unreachable/unwanted targets, not "recently harvested"
- `HostileMobNearInst` — danger avoidance, unrelated
- inventory/stack checks — stop re-targeting a *different, still-pickable* bush of a type it
  already has enough of

No explicit "remember what I just picked" logic exists anywhere in this file — exclusion is a side
effect of asking the plant itself, fresh, every scan.

---

## 9. Light and hunger thresholds, and nightfall lookahead

**Short answer: no lookahead in either file — both are purely reactive to the current instant, not
planned ahead of nightfall.**

### findormakelight.lua thresholds

| Check | Threshold | Where |
|---|---|---|
| "Too dark, run to light NOW" | `LightWatcher:GetLightValue() < TUNING.SANITY_LOW_LIGHT` | line 72 |
| "Stop running, safe now" | `LightWatcher:GetLightValue() >= max(TUNING.SANITY_LOW_LIGHT, TUNING.SANITY_HIGH_LIGHT/3)` | line 144 |
| "Fire needs fuel" (trigger) | `fueled:GetPercent() < .25` | lines 109, 233 |
| "Fire fully fueled again" | `fueled:GetPercent() > .25` | line 276 |
| "Use non-log emergency fuel" | only if `fueled:GetPercent() < .15` | line 258 |
| "Still OK, stop pestering the fire" | `fueled:GetPercent() > .05` (fail out) | line 269 |

`SANITY_LOW_LIGHT`/`SANITY_HIGH_LIGHT` are engine `TUNING` constants — this node borrows the game's
own sanity-lighting cutoffs rather than defining its own concept of "dark."

**Lookahead: none.** The node only runs at all because of a guard back in `artificialwilson.lua`:

```lua
WhileNode(function() return clock and clock.isnight end, "StayInTheLight", MaintainLightSource(self.inst, 30))
```

Gated strictly to `clock.isnight` — nothing prepares ahead of dusk (no stockpiling fuel, no
building a campfire before sunset). The first moment it can even check "am I in danger" is once
night has already fully arrived.

### managehunger.lua thresholds

The main threshold, `hungerPercent`, is a parameter, not hardcoded in this file — only acts when
`hunger:GetPercent() <= self.percent` (line 29). Call sites (`artificialwilson.lua`) set it to:
- **.5** during `day`, `dusk`, `dusk2`
- **.9** during `night`

Hardcoded thresholds inside this file, independent of that parameter:

| Check | Threshold | Where |
|---|---|---|
| Opportunistically eat lesser food about to go stale | fresh and `perishable:GetPercent() < .6` | line 67 |
| Opportunistically eat lesser food about to fully spoil | stale and `perishable:GetPercent() < .3` | line 71 |
| "Desperate enough to eat harmful food" | `hunger:GetPercent() <= .15` | line 87 |

**Lookahead: none, same pattern.** The 90% night threshold isn't anticipation of "no gathering
happens at night" — it's simply a different fixed number the root tree hands the node once
`clock.isnight` is already true. Nothing checks "how much daylight is left" to eat/stockpile early;
both files react only to instantaneous state (`LightWatcher` value, `fueled` percent, `hunger`
percent) at the moment they're ticked.

---

## 10. git history: upstream/feature/FixStuckWilson

`feature/FixStuckWilson` isn't a diverged branch — its tip commit is a direct ancestor of `master`
(`master` simply kept going afterward with ~29 more commits;
`git merge-base upstream/master upstream/feature/FixStuckWilson` returns the branch's own tip). Its
tip is a single commit:

> **f6d4be0** — "Have a pathfinder listener going now...if he can't find a path, it will trigger the
> listener to restart the brain. Also ads that prefab GUID to an ignore list as well as his current
> position."

### The problem it was solving

Wilson would get physically stuck (walking into water, pathing to an unreachable target) with no
signal from the game engine that anything was wrong — there was no built-in event for "pathfinding
failed," so he'd just keep running in place indefinitely.

### What the commit changes

**1. `modmain.lua` (+204 lines)** — monkey-patches the locomotor's internal update loop
(`RoGOnUpdate`) to watch the engine's own pathfinder status (`Pathfinder:GetSearchStatus`). On an
empty/failed search result, it fires a brand-new custom event that didn't exist in the base game:

```lua
self.inst:PushEvent("noPathFound", {inst=self.inst, target=self.dest.inst, pos=...})
```

**2. `scripts/brains/artificalwilson.lua`** (same file as the current `artificialwilson.lua`, just
an earlier spelling) — adds the listener/handler for that event. This is the same `OnPathFinder`
function and `noPathFound` listener still present in the current codebase:

```lua
local function OnPathFinder(self,data)
    print("Pathfinder has failed!")
    if data.target then
        self.brain:AddToIgnoreList(data.target.entity:GetGUID(), Vector3(self.Transform:GetWorldPosition()))
    end
    ...
    self:AddTag("IsStuck")
end
```

On failure: says a line ("I'm too dumb to walk around this..."), forcibly stops the locomotor, tags
Wilson `"IsStuck"` — caught by the root `PriorityNode`'s first child (`FixStuckWilson`, section 1),
which resets the whole behaviour tree.

**3. The ignore-list got position-awareness in this same commit.** Before: `AddToIgnoreList(prefab)`
blacklisted a prefab *type* forever — too blunt (one unreachable rock shouldn't ban all rocks
globally). This commit added an optional position:

```lua
function ArtificalBrain:AddToIgnoreList(prefab, fromPos)
    if not fromPos then
        IGNORE_LIST[prefab].always = true  -- ban this type everywhere
    else
        table.insert(IGNORE_LIST[prefab].posTable, fromPos)  -- ban only near this spot
    end
end
```

`OnIgnoreList` was updated to check distance against remembered positions
(`TRY_AGAIN_DIST = 15` in this commit — later renamed `try_ignore_dist` and bumped to 30 in
`prioritizer.lua`). A target that fails pathfinding gets blacklisted *only near where it failed*,
not everywhere.

**In short:** this branch is the origin of the entire stuck-recovery system traced through the rest
of these notes — the `noPathFound` event, the `OnPathFinder` handler, the `"IsStuck"` tag,
`FixStuckWilson`, and the position-aware ignore list are all first introduced here, later relocated
into `components/prioritizer.lua`.
