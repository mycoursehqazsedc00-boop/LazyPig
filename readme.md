# _LazyPig

A quality-of-life automation addon — auto-accept, auto-roll, auto-dismount, chat/spam filtering, and related conveniences.

## Client mods used

**None.** Checked two candidates specifically for this addon and neither applies:

- **ClassicAPI's `UnitIsInMyGuild`** — looked like a natural replacement for the addon's own `GetGuildMemberInfo`-based roster scan, but its own documentation confirms that for a **literal name string** (not a resolved unit token) it falls back to walking the same engine guild-roster array `GetGuildRosterInfo` already uses. LazyPig checks guild membership from parsed chat/invite names, not from live unit tokens, so this wouldn't have been faster — not used.
- **WeirdUtils' `string.find`/`gsub`/`gfind` literal prefilter** — plausibly helps, since this addon does a fair amount of chat-text pattern matching, but it's an engine-level optimization that works transparently if a user has `weirdperformance.dll` installed — there's nothing for LazyPig's own code to call or depend on either way. Not a dependency, not integrated, just complementary if present.

No feature in this update requires any client mod to be installed.

## Features

- Auto Dismount
- Auto Stance
- Extended camera distance (optional)
- Auto-accept invites — separately toggleable for guild members, friends, and strangers; can be disabled while in a battleground or its queue
- Auto-accept summons
- Auto battleground: auto-join, auto-leave, auto-queue, auto-release spirit in BG
- Auto-decline quest sharing while in a battleground (optional)
- Announce when queueing for a battleground as party leader (optional)
- Position the loot frame at the cursor (optional)
- Improved right-click (optional)
- Auto-roll: green items, ZG coins/bijou, MC mats, AQ scarabs/idols, corrupted sand, Dream Shards, Naxx scraps — each independently configurable
- Auto-roll chat messages (optional)
- Auto-cancel duels (optional)
- Special key-combination actions
- Mute World chat — while in dungeons, while in a raid, while in a battleground, or always (each independently toggleable)
- Hide player spam messages (duplicate-text filter across Say/Channel/Yell/Emote)
- Hide green/blue item-roll messages (independently toggleable)
- Improved stack-splitting with Shift
- Auto-accept resurrection in raid/dungeon/BG, if the resurrecter is out of combat
- Auto-process gossip dialogs
- Auto-remove Blessing of Salvation (configurable: off, always, or tank-only)
- Auto-remove mana buffs (Blessing of Wisdom, Arcane Intellect, Prayer of Spirit)
- Dedicated keybinds: Logout, Unstuck, Reload UI, target WSG EFC/cancel duel, drop WSG flag/remove Slow Fall, open the LazyPig menu
- `/lp` for in-game options

## New features in this update

**None.** This update is fixes and internal performance improvements only — no new user-facing feature was added. (If you'd like one added, say the word — happy to scope something out.)

## What changed — performance

1. **`IsGuildMate()` recomputed `string.lower(name)` on every single roster entry** instead of once before the scan loop. Hoisted out.
2. **The invite-accept check could call `IsGuildMate()`/`IsFriend()` up to twice each** for the same name within one expression — both full list scans. Now cached in locals, computed once.
3. **Two full buff-bar scans ran on every `PLAYER_AURAS_CHANGED` event** (frequent in combat) — `LazyPig_CheckSalvation()` and `LazyPig_CheckManaBuffs()` each independently walked the whole player buff bar. Merged into a single `LazyPig_CheckBuffs()` that scans once and checks both buff lists per entry. Verified against 7 mock-buff-bar scenarios (including an `untilCancelled` buff that must be skipped): identical cancel/message outcome in every case, roughly half the API calls when both features are active.

## What changed — bug fix

**Spam filter was only ever exempting guildmates/friends from `CHAT_MSG_EMOTE`, never Say/Channel/Yell.** Lua's `and` binds tighter than `or`, so:
```lua
... (event == "CHAT_MSG_SAY" or event == "CHAT_MSG_CHANNEL" or event == "CHAT_MSG_YELL" or event == "CHAT_MSG_EMOTE" and not (IsGuildMate(arg2) or IsFriend(arg2)))
```
only ever applied the `not (IsGuildMate or IsFriend)` exemption to the last clause (`CHAT_MSG_EMOTE`). A guildmate's or friend's Say/Channel/Yell message could get hidden as spam if someone else happened to post identical text within 70 seconds. Fixed by parenthesizing so the exemption applies consistently across all four message types:
```lua
... (event == "CHAT_MSG_SAY" or event == "CHAT_MSG_CHANNEL" or event == "CHAT_MSG_YELL" or event == "CHAT_MSG_EMOTE") and not (IsGuildMate(arg2) or IsFriend(arg2))
```
Verified across every event × guildmate × friend combination: `CHAT_MSG_EMOTE` behavior is unchanged (it was already correct); only Say/Channel/Yell from a guildmate-or-friend changes, now correctly exempted to match Emote's existing behavior.

## Checked and found clean (full regression pass)

- Every `OnUpdate` across all three Lua files — the main automation loop is throttled to 10Hz and all its queued-action tables are small and user-action-bounded, not combat/raid-scaling.
- `ChatSpamClean()` — a properly double-buffered, bounded cleanup, not a leak.
- Searched systematically for other instances of the same operator-precedence bug — none found; it was an isolated case.
- One item flagged but **not** changed, pending your confirmation of intent: `ScheduleButtonClick`'s queue processor silently drops (rather than defers) a pending scheduled click if a shared cooldown happens to be active when it runs. Could be an oversight, could be deliberate — Lua's grammar doesn't make this one unambiguous the way the spam-filter bug was.

## Verification

- Syntax-checked (`luac5.1 -p`) — clean.
- All behavioral claims above verified by simulation against the actual extracted/patched code, not just reasoning by eye.
- Patch reapplies to a clean copy of the original and reproduces the shipped file exactly, byte for byte.
- Not tested in a live client.

## Files changed

```
LazyPig.lua   (only file touched)
```
