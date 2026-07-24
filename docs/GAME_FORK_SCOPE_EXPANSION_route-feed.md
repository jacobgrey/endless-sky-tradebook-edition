# Scope Expansion — In-Game Route Feed

**Read `GAME_FORK_SPEC.md` first.** This document adds one more output to that spec: a **second, separate feeder file** that exposes the player's current in-game plotted course (travel plan) in near real time. All the non-negotiable constraints from the main spec apply unchanged (saves stay byte-for-byte vanilla, same config directory, opt-in, read-only observer, atomic writes).

## Why a *second* file

The main `live-status.json` covers location/cargo/fleet and updates on system-entry, landing, liftoff, and cargo changes. The **plotted course changes far more often** than any of those — the player re-plots on the map constantly. Bundling it into `live-status.json` would force our tool to re-parse the whole status payload on every minor map edit.

So the travel plan gets its own tiny file, `live-route.json`, written **only** when the plan changes. Our tool watches it independently and cheaply, without touching the larger status file or triggering a full game-data reload.

## What the fork must add

A small writer, e.g. `void PlayerInfo::WriteLiveRoute() const;`, invoked whenever the player's travel plan or travel destination changes:

| Trigger | Where |
|--------|-------|
| player plots / extends / clears a course on the map | wherever `TravelPlan()` (the mutable overload) or `SetTravelDestination()` is edited (the map/mission panels) |
| plan shrinks as the player jumps | after `PopTravel()` |
| destination set/cleared | after `SetTravelDestination(...)` |

Relevant `PlayerInfo` API (verified against current `master` — re-verify before wiring):

```cpp
bool HasTravelPlan() const;
const std::vector<const System *> &TravelPlan() const;   // the plotted systems
void PopTravel();                                        // drops the traversed end on jump
const Planet *TravelDestination() const;                 // final landing target, may be null
void SetTravelDestination(const Planet *planet);
```

**Ordering — important:** normalize the output to **travel order**: element `[0]` is the next system the player will jump into from their current system, and the last element is the destination system. The internal `TravelPlan()` vector may be stored reversed (the game pops one end on arrival); confirm the direction against the Engine's jump handling and reverse on write if needed so the file is always in travel order.

## `live-route.json` schema (v1)

Same config directory as `live-status.json` (`%APPDATA%\endless-sky\live-route.json`), atomic write (`.tmp` + rename).

```json
{
  "schemaVersion": 1,
  "timestamp": "2026-07-23T14:05:00Z",
  "saveName": "Jericho Blaze",
  "origin": "Dschubba",
  "travelPlan": ["Lesath", "Alniyat", "Prime"],
  "destination": "Prime Station",
  "hasPlan": true
}
```

- `timestamp` — real-world ISO-8601 UTC (freshness).
- `saveName` — save file stem, to pair the feed to the correct pilot (must match the save filename without extension).
- `origin` — the system the plan starts from (the player's current system). Lets the consumer render the full path `origin → travelPlan…`.
- `travelPlan` — ordered system names in **travel order** (next jump first, destination system last). Empty array when no course is plotted.
- `destination` — the final **planet/station** target if the player set one, else `null` (a course can target just a system with no landing).
- `hasPlan` — `false` when the player has no plotted course (mirrors `HasTravelPlan()`); `travelPlan` is then empty.

Write the file (with `hasPlan: false`, empty `travelPlan`) when the course is cleared, so the consumer can distinguish "no route" from a stale file.

## Checklist for the fork agent

- [ ] Opt-in preference gates both `live-status.json` and `live-route.json` (or a shared "companion feed" toggle).
- [ ] `WriteLiveRoute()` reads only; never mutates game state; never on the save-writing path.
- [ ] Called on every travel-plan / destination change **and** on `PopTravel()`.
- [ ] Output normalized to travel order, atomic write, same config dir.
- [ ] Saves remain byte-for-byte identical to vanilla; the only new artifacts on disk are these two side-channel files.

## Consumer side (our tool — for your context, not your deliverable)

The Trade Reporter will add an **"Import in-game route"** capability to its Route filter, with two modes:

- **Manual** — a button that reads `live-route.json` once and loads `origin + travelPlan` into the route filter (systems become the filter's ordered legs; the table sorts by leg / travel order).
- **Automatic** — a toggle; the tool watches `live-route.json` and re-imports whenever it changes.

Because route edits are frequent, the tool watches this file on its own lightweight path, separate from the main `/api/data` load — which is exactly why it must be a **separate file** from `live-status.json`. No other interface change: same panels, just a new button + toggle in the Route section.
