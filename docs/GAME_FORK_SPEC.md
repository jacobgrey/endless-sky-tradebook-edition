# Live-Status Feed — Game Fork Spec

**Audience:** an agent tasked with modifying the Endless Sky game (a fork, or ideally an upstream contribution) so that companion tools can read the player's *current* state in real time.

**Why:** Endless Sky writes its save file only on **takeoff** (so a death-reload returns the player to the planet pre-departure). Every companion tool that reads the save therefore sees "where the player just left," never "where they are now." A tiny always-current status file eliminates that lag. This is the missing piece for the Endless Sky Trade Reporter's live tracking (see repo `CLAUDE.md`).

## Non-negotiable constraint: save files stay byte-for-byte vanilla

The modified game **must produce save files identical to the unmodified game**. Saves must remain fully interchangeable in both directions — a save written by the fork loads cleanly in vanilla and vice versa, with no differences.

Therefore:

- **Do not change the save format, contents, or write cadence.** No new fields, no extra sections, no writing the save more often or at different moments. Leave the existing save serialization path completely untouched.
- **Use the exact same config/save directory as vanilla** (`%APPDATA%\endless-sky\` on Windows, and the platform equivalents). Do not rename the config directory, namespace it, or create a separate profile for the fork. Saves, `preferences.txt`, `plugins/`, `recent.txt`, etc. are shared, so the player can start either build and continue exactly where they left off. **From the player's standpoint there is zero difference** between launching the modified and the unmodified game.
- **All new or higher-frequency information goes only to the separate `live-status.json` file**, whose sole purpose is communicating with this tool. It is never part of, and never affects, the save.
- The live file is purely additive and side-channel: the *only* observable difference between the two builds is the presence of that one file in the config directory. Deleting it, or running vanilla, changes nothing about saves or gameplay.

Anything already practical to read from the save (visited systems, the economy supply table, applied events) can keep being read from the save by the tool — the fork does not need to duplicate it. The live file exists only to provide *current, real-time* state that the save cannot (because the save is written only on takeoff).

## Goal

While the game runs, keep a small JSON file on disk — **entirely separate from the save** — that always reflects the player's current location, cargo, and fleet state. Rewrite it (atomically) whenever any of these change:

1. **Space-system entry** — the player jumps/enters a new system. `planet` becomes `null`, `landed` false.
2. **Landing** — the player lands on a planet/station. `planet` set, `landed` true.
3. **Liftoff (takeoff)** — the player departs a planet. `planet` becomes `null`, `landed` false.
4. **Any cargo change** — buying/selling commodities, and mission/job cargo being added or removed.

Keep it **opt-in** via a preference (e.g. a `"live status file"` preference, default off) so vanilla players are unaffected and the change is upstream-friendly.

## Where to hook (verified against current `master`)

All of these are reachable from `PlayerInfo` (`source/PlayerInfo.h`), which owns the authoritative state. The clean design is a single writer method, e.g. `void PlayerInfo::WriteLiveStatus(const char *event) const;`, invoked from:

| Event | Call site |
|-------|-----------|
| system entry | after `PlayerInfo::SetSystem(const System &)` (and/or `Engine`'s enter-system path) |
| landing | end of `PlayerInfo::Land(UI &)` |
| liftoff | end of `PlayerInfo::TakeOff(UI &, bool)` |
| cargo change | `TradingPanel` buy/sell (which mutate `CargoHold` via `Add`/`Remove`/`Transfer`), and the mission-cargo paths in `PlayerInfo` (accept/abort/complete) |

The writer is a **read-only observer** of game state: it reads `PlayerInfo` and emits the live file. It must not mutate any game state, and must not sit on the save-writing code path. State it reads, all already on `PlayerInfo`:

- `GetSystem()`, `GetPlanet()` — current system / planet (planet is `nullptr` in flight).
- `GetDate()` — in-game date.
- `Accounts()` — credits.
- `Cargo()` → `CargoHold`: `Commodities()` (the `map<string,int>` of tradeable goods), and mission cargo is stored separately (`missionCargo`, keyed by `const Mission *`) — sum it for `missionCargoTons`.
- `Flagship()` / the active fleet — for total cargo capacity and fuel/jump range.

> Verify each signature in the source before wiring; the game evolves. As of writing: `void Land(UI&)`, `bool TakeOff(UI&, bool)`, `void SetSystem(const System&)`, `const System *GetSystem() const`, `const Planet *GetPlanet() const`, `CargoHold &Cargo()`, `const Account &Accounts() const`, `const Date &GetDate() const`, `const Ship *Flagship() const`; `CargoHold::Commodities()`, `Size()`, `Free()`, `Used()`.

## Output file

- **Path:** the game's config directory (same folder as `saves/`), file name `live-status.json`. On Windows that resolves to `%APPDATA%\endless-sky\live-status.json`.
- **Atomic write:** write to `live-status.json.tmp` then rename over the target, so a reader never sees a half-written file.
- **On exit:** optionally delete the file (or leave it; consumers treat a stale file as "no live data" — see Consumer contract).

## Data schema (v1)

```json
{
  "schemaVersion": 1,
  "event": "landing",
  "timestamp": "2026-07-23T14:05:00Z",
  "pilot": "Jericho Blaze",
  "saveName": "Jericho Blaze",
  "date": [23, 1, 3015],
  "system": "Dschubba",
  "planet": "Sundrinker",
  "landed": true,
  "credits": 1220479,
  "fleet": {
    "cargoCapacity": 100,
    "jumpRange": 4
  },
  "cargo": {
    "Food": 40,
    "Medical": 10
  },
  "missionCargoTons": 56,
  "outfitCargoTons": 0
}
```

Field notes:

- `event` — one of `"system-entry" | "landing" | "liftoff" | "cargo-change"`.
- `timestamp` — real-world ISO-8601 UTC; lets a consumer judge freshness.
- `saveName` — the save file stem, so a consumer can pair the feed to the correct pilot/save (must match the save filename without extension).
- `date` — in-game `[day, month, year]`.
- `system` — current system name (always present while playing).
- `planet` — current planet/station name, or `null` when in flight.
- `landed` — convenience boolean (`planet != null && actually docked`).
- `fleet.cargoCapacity` — total tradeable cargo tons across the active (non-parked) fleet.
- `fleet.jumpRange` — minimum consecutive jumps the fleet can make on a full tank (fuel ÷ cheapest installed drive's jump fuel, taken over the weakest active ship). Optional but valuable; omit if not readily computed.
- `cargo` — tradeable commodities only, commodity → tons.
- `missionCargoTons` — total non-tradeable mission/job cargo tonnage.
- `outfitCargoTons` — outfits being carried as cargo (tonnage); `0` if none.

These field names deliberately mirror the Trade Reporter's `/api/data` `player` object so the consumer can adopt them with minimal mapping.

## Consumer contract (Trade Reporter side — the planned next step here)

The Trade Reporter will add an **optional live-status input**:

- If `live-status.json` exists **and** its `saveName` matches the active save **and** its `timestamp` is recent, use its `system`/`planet`/`landed`/`cargo`/`credits`/`date`/`fleet` as the *current* player state — replacing the save-derived, last-departed values.
- Otherwise fall back to the current behavior (parse the save). No regressions when the file is absent or stale.
- Prices still come from the save's `economy` block (the live file intentionally does not carry the full economy supply table); the economy only meaningfully changes on save/day-advance anyway.
- **No interface change** beyond the data being fresher: same panels, same controls. Live tracking simply reflects landings/cargo the instant they happen instead of on the next takeoff.
- Because saves are identical whether written by the fork or vanilla, the tool keeps reading the save exactly as it does today; which game wrote the save is irrelevant. The live file only ever *adds* freshness on top.

## Upstreaming vs. fork

Prefer proposing this to the Endless Sky project (https://github.com/endless-sky/endless-sky) as an opt-in preference — it benefits every companion-tool author and avoids maintaining a private build. If declined, a local fork applying the same writer + preference is straightforward (CMake + vcpkg on Windows); rebuild after game updates. Reading process memory is explicitly rejected: it breaks on every game update and is permanent reverse-engineering maintenance.
