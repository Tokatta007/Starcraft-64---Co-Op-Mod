# StarCraft 64 Campaign Co-Op — Technical Notes

Reverse-engineering notes and patch documentation for the StarCraft 64 (U)
campaign Co-Op mod. This file is for anyone who wants to understand how the mod
works or extend it further (especially: getting **Dark Origins** into the scenario
menu, which I could not complete cleanly — see the final section).

Base ROM: **StarCraft 64 (U) [.z64, big-endian], 32 MB.**
Unmodified source ROM MD5: `559f71b861f639b6376d891e3023414b` (USA, `.z64`,
first bytes `80 37 12 40`).
Internal name at header `0x20`: `STARCRAFT 64` (vanilla) → set to `STARCRAFT CO-OP V1`
in the released build so emulators/flash carts display the mod name.

---

## Address / offset conventions

- **Static ROM code** (the main executable): file offset = `RAM − 0x80000000 + 0xC00`.
  (Some analysis was done in Ghidra with N64LoaderWV using `MIPS:BE:64:64-32addr`.)
- **Boot segment** (low code, e.g. the file loader at `0x800074C0`):
  file offset = `RAM − 0x7FFFF400`.
- **Menu overlay** (the `0x8012xxxx` scenario/menu code, DMA-loaded at runtime):
  file offset = `RAM − 0x800495B0`. **Important:** the overlay is loaded in
  *fragmented segments* and this delta is only locally correct — it drifts across
  the overlay. This drift is what makes free-space code injection into the overlay unsafe
  (see the Dark Origins section).

---

## Key RAM addresses (runtime)

| Address | Meaning |
|---|---|
| `0x800AFEFC` | `second_player_active` — gates ALL P2 UI (cursor/viewport/HUD). The engine also treats this as "is multiplayer" in ~44 end-game branches. Load-bearing during gameplay **and** input handling; cannot be cleared mid-session without glitching/freezing. |
| `0x800B00FC` | P1 game slot |
| `0x800B0100` | P2 game slot |
| `0x800B0104` / `0x800B0108` | P1 / P2 selection contexts |
| `0x800AFF54` | Master player table, stride `0x24`. +0 control (`02`=human occupied, `06`=open), +1 race, +2 local player #. |
| `0x800D13C4` | Progression counters (6 bytes, one per episode). RAM copy loaded at boot. |
| `0x800D0154` | Default progression template (static data, loaded to `0x800D13C4` by `FUN_800d8080`). |
| `0x800D13D0` | Cheat-unlock flags word (secret code ORs `0x781E` here). |
| `0x800D118C` / `0x800D118D` | Secret-scenario unlock flags (Resurrection IV etc.). |
| `0x800D13F8` | Current map index (u16). Level-select via `800D13F9 00XX`. |

---

## The five gameplay patches (co-op core)

All found in the **static ROM**. These make P2 share P1's forces with independent
cursor/selection. Derived per-mission automatically.

1. **P2 shares P1's slot** — RAM `0x8003DF34`, in `FUN_8003de50`.
   `30 42 00 FF 14 57 00 05` → `30 42 00 FF 14 46 00 05` (`bne v0,s7` → `bne v0,a2`).

2. **P2 own selection context** — RAM `0x8003DF48`.
   `AC D0 00 04 AF D0 00 04` → `AC D0 00 04 AF D7 00 04` (`sw s0` → `sw s7`).

3. **P2 viewport enable** — RAM `0x8007A454`, in `FUN_8007a3e0`.
   `24 03 00 01 A0 40 D3 B8` → `24 03 00 01 A0 43 D3 B8`.

4. **Enable second player on campaign load** — RAM `0x800229B4`, in `FUN_80022970`
   (file ~`0x000235B0`).
   `3C 12 80 0B 92 42 FE FC 10 40 00 02 24 04 00 01 24 04 00 02`
   → `3C 12 80 0B 24 02 00 01 A2 42 FE FC 24 04 00 01 24 04 00 02`.
   **Timing-critical:** the flag MUST be set before `FUN_80022970` runs, or black-screen crash.

5. **Menu text** — file offset ~`0x000D1B20`: "Single-Player"→"Story Co-Op",
   "Two-Player"→"Team Melee". Strings uncompressed; shorter is OK (pad with `00`).

---

## Unlock / data patches

**Unlock all 60 missions.** Default progression template at `0x800D0154`
(file `0x000D0D54`). First 6 bytes `03 01 01 01 01 01` → `0C 0A 0A 0A 0A 0A`
(per-episode unlock counts; `0C`=Terran incl. tutorials, `0A` each other episode).

The copy function `FUN_800d8080` publishes this template into `0x800D13C4` at boot,
so editing the template makes the unlock permanent regardless of save state.

**Unlock all cheats.** Template +0xC (`0x800D0160`, file `0x000D0D60`).
Set to `00 00 78 1E` — the value the secret cheat-code ORs into `0x800D13D0`.

**Unlock secret-scenario flags (Resurrection IV, etc.).** `0x800D118C`/`0x800D118D`
are runtime flags checked by the scenario menu. They are set by the secret button-code
handlers (`FUN_800d837c`, `FUN_800d84b8`). To make them permanent, I hooked the boot
progression-copy function `FUN_800d8080` to write both at boot:

- At the function's tail (file ~`0x000D8D04`):
  `24 05 01 01 A4 45 FD B8` → `24 05 01 01 A4 45 FD 98`
  (loads `0x0101` and stores it as a halfword to `0x800D118C`, setting both flags).
  The offset `-0x268` (`0xFD98`) is correct here — an earlier attempt used `-0x248`
  (`0xFDB8`) which hit `0x800D11AC` by mistake. Verify `0x800D118C`/`118D` read
  `01 01` after boot.

---

## Scenario menu: Mass Hysteria

The Two-Player Scenario menu is built by `FUN_8012404C` (RAM `0x8012404C`).

- The **visible entry count** is a literal `li v0, 0x08` at RAM `0x8012412C`
  (file `0x000DAB7C`), stored into the menu descriptor at `sp+0x1C4`.
- The menu computes each slot's **map index** as `slot + 0x57`
  (`addiu v0,v0,0x57` at RAM `0x801243B0`). So:
  slot 0 → map `0x57` (Pro Bowl) … slot 8 → map `0x5F` (**Mass Hysteria**).
- The map's **file number** = map index + 8 (Pro Bowl file `0x5F`, Mass Hysteria
  file `0x67`). Files live in a **compressed BOLT archive** (folder `008`).

Mass Hysteria (and its `*Mass Hysteria*` label + table entry) was left half-wired in
the ROM by the developers but never displayed, because the count was `8`.

**Patch — reveal Mass Hysteria (count 8 → 9):**
```
Find:    24 06 00 1B 24 02 00 08 AF A6 01 C0
Replace: 24 06 00 1B 24 02 00 09 AF A6 01 C0
```
(file `0x000DAB78` region; the count byte at `0x000DAB7C` goes `08` → `09`.)

Setting the count to **`0x0A`** (10) instead reveals a 10th slot which computes map
`0x60` (file `0x68`) — a **nonexistent file**, so the game hangs in the map-load DMA
wait loop (`0x80004994`/`0x80004998` polling `s1+0xE040`) and then faults to the
exception vector `0x80000180`. Keep the count at `0x09`.

---

## Campaign menu crash fix (Scenario/Load Saved navigation)

**Symptom.** After the co-op patches, navigating the **Story Co-Op (campaign)**
mission list left/right into the "Scenario" or "Load Saved" tabs hung the game (DMA
wait loop at `0x80004994`/`0x80004998`, then exception vector). Vanilla single-player
never hit this because it builds a different (single-player) scenario list; my co-op
patches made the campaign side attempt a menu setup whose asset load
(`jal 0x800074C0` at RAM `0x80126D2C`, params a2=`0xE2`/a3=`0x122`) never completes in
that context. The crash was reached by *falling through* the campaign frame handler,
not via the menu-state jump table (`0x800D0088`) — a breakpoint on that dispatch never
fired on the right-press, which is why several table/count-based fixes missed.

**Root of the fix.** The campaign mission-list frame handler is `FUN_80125E7C`. At
`0x80126500` it calls the shared menu updater `FUN_80122DC4` → nav handler
`FUN_80122BF4`. That nav handler does two things at once: (1) essential per-frame menu
state upkeep, and (2) returns a value in `v0` whose **sign** encodes the requested tab
move (`>0` = right, `<0` = left, `0` = no move). Immediately after the call:

```
0x80126504  nop
0x80126508  addu v1, v0, zero      ; v1 = nav result (the requested tab move)
0x8012650C  blez v1, ...           ; sign of v1 drives left/right/none
```

NOP-ing the whole call (an earlier attempt) destroyed the state upkeep and made the
menu auto-navigate and crash on open. The working fix keeps the call intact and simply
**forces the directional result to zero** so the tab never moves:

```
0x80126508   addu v1, v0, zero   (00401821)   ->   addu v1, zero, zero   (00001821)
```

**Patch (one instruction, file offset `0x000DCF58`):**
```
Find:    0C 04 8B 71 00 00 00 00 00 40 18 21
Replace: 0C 04 8B 71 00 00 00 00 00 00 18 21
```
(The `0C 04 8B 71` is `jal FUN_80122DC4`; the following `00 00 00 00` is its delay-slot
nop; only the final `00 40 18 21` → `00 00 18 21` changes. The 12-byte context makes
the location unique.)

**Why it's clean and contained.** The edit lives in the campaign-specific frame handler
`FUN_80125E7C`. The **Two-Player** menu drives its tab navigation through a *different*
caller (`0x80125418`), so Two-Player Scenario navigation (including Mass Hysteria and
Resurrection IV) is untouched. Because the tab index can no longer move on the campaign
menu, the game's own "tab unavailable" state kicks in: the Scenario / Load Saved tabs
render **greyed out** and left/right play the invalid-input beep. This looks like an
intentional design choice, not a workaround. Verified: crash gone, menu opens cleanly,
Two-Player / Encyclopedia / chapter select all unaffected.

---

## Progression / saving in co-op — why it's disabled

Extensively traced via R4300i CPU logs (vanilla win, co-op win, co-op quit). Summary:

- Vanilla mission-complete increments `0x800D13C4[episode]` (write at RAM `0x800337A4`,
  in the handler reached via `FUN_8001BCB4 → FUN_8001FA58 → FUN_80038D1C →`
  campaign-advance). The RAM value and the `.fla` save (offset `0xC007`) both go
  `03 → 04` on beating the first Terran mission.
- In **co-op**, that entire victory chain **never executes.** `second_player_active`
  causes the engine to classify the session as multiplayer from launch, so the
  "campaign mission won" event/message is never generated — the session ends through
  a separate multiplayer results path (`0x80128xxx`) instead.
- The divergence is upstream of the dispatcher and is **runtime function-pointer
  driven**, so it can't be cleanly re-routed with a branch patch. Clearing
  `second_player_active` at the mission-complete popup was tested and breaks input
  (the popup's own input handling reads the flag) — so there's no window to clear it.

**Decision:** progression is not fixable without re-engineering the end-of-mission
flow. I shipped **unlock-all** instead (see above), which is why every mission is
selectable from the start.

---

## Dark Origins — unfinished; notes for a future attempt

**Goal:** make **Dark Origins** (map index `0x3B`, file `0x43`, `008\043.chk`) a real
entry in the Two-Player Scenario menu (as slot 10, right after Mass Hysteria).

**Why it's hard.** The menu computes `map = slot + 0x57`. Slot 10 therefore computes
map `0x60`, not `0x3B`. Dark Origins is *outside* the consecutive scenario block, so
no count value reaches it. Making slot 10 load Dark Origins requires **substituting**
its map index and every avenue I tried is blocked:

1. **Code injection (hook).** I built a working hook: intercept the shared map-index
   read at `0x80122DF4` (`lhu v0,0xA(a1)`), and when the value is `0x60` (slot 10),
   substitute `0x3B`. The routine and jump encode correctly. **Problem:** there is no
   free space to host the routine. The menu overlay is loaded in fragmented segments
   and is **100% packed** — every resident byte is used. Free ROM space (e.g. the big
   zero region at file `0x01FB1A72`) maps to RAM addresses that are **not resident**
   when the menu runs, so jumping there executes garbage. My first attempt placed the
   routine at RAM `0x8011DC20` (below the resident region, start `0x8011FBF8`) and it
   corrupted the boot. **A hook needs free space inside a correctly-mapped resident
   overlay segment, which does not exist here.**

2. **In-place arithmetic edit.** `map = slot + 0x57` is linear; a single instruction
   can't special-case one slot without a conditional, which needs space we don't have.

3. **Repoint a map→file table.** There is **no** `map→file` lookup table. The mapping
   is arithmetic (`file = map_index + 8`), computed in code. (Several apparent "tables"
   found by scanning were coincidental ascending byte sequences / compressed map data.)

4. **Rename / reinsert the map file (most promising).** Idea: place Dark Origins'
   `043.chk` data at archive index `0x68`, so slot 10's computed map `0x60` → file
   `0x68` finds Dark Origins there. **Blocker:** maps are stored **compressed in a
   BOLT archive**, indexed by number (all in folder `008`). `BOLTextract` is
   **extract-only** and there is no repacker. Building one from the BOLT format is a
   real software project (reverse-engineer the compression + directory), not a patch.

**Cleanest path for a future effort:** write (or find) a **BOLT repacker**, then
insert Dark Origins' map at the archive index the scenario menu computes for slot 10
(file `0x68`), and set the scenario count to `0x0A`. That makes Dark Origins a menu
entry with **no code changes at all** — purely a data/archive edit — which sidesteps
the overlay's no-free-space problem entirely. This is the recommended approach.

**Shipped workaround.** Dark Origins is fully playable via level-select:
set `0x800D13F8` (u16) to `0x003B` — GameShark `800D13F9 003B`, then select any
level. It boots and plays normally. (This was verified in-game).

---

## Tools & workflow

- **BOLTextract** (github.com/heinermann/BOLTextract) — extracted 96 `.chk` maps.
- **Ghidra** + **N64LoaderWV** (github.com/zeroKilo/N64LoaderWV), language
  `MIPS:BE:64:64-32addr`. For runtime overlay code not in the static ROM, import an
  8 MB RDRAM dump, set image base `0x80000000` (Memory Map → Set Image Base), and
  auto-analyze.
- **Project64** debugger: set CPU Core = Interpreter for reliable breakpoints;
  R4300i → Command Log → CPU logging for execution traces (slow; keep captures short).
  Memory breakpoints report an imprecise PC (often off by a delay slot or two).
- **HxD** for hex edits; **rn64crc -u** (or a CIC-6102 CRC routine) to fix the checksum.
- Distribute as a **BPS/xdelta patch** (Floating IPS / xdelta), never the ROM. Record
  the source ROM's MD5/CRC in the release.

---

## Scenario / secret map reference

| Name | Map index | CHK file | Notes |
|---|---|---|---|
| Pro Bowl | `0x57` | `05F.chk` | scenario slot 0 |
| Round Up | `0x58` | `060.chk` | scenario slot 1 |
| King of the Hill | `0x59` | `061.chk` | scenario slot 2 |
| Old Faithful | `0x5A` | `062.chk` | scenario slot 3 |
| Guardians | `0x5B` | `063.chk` | scenario slot 4 |
| Zerg Troopers | `0x5C` | `064.chk` | scenario slot 5 |
| Resurrection IV | `0x5D` | `065.chk` | scenario slot 6 (secret; unlocked via flag) |
| Rage | `0x5E` | `066.chk` | scenario slot 7 |
| **Mass Hysteria** | `0x5F` | `067.chk` | scenario slot 8 (**revealed by count 8→9**) |
| **Dark Origins** | `0x3B` | `043.chk` | folder `008`; not in the consecutive block. Playable via `800D13F9 003B`. |

Map index → CHK file number = **index + 8**. All maps are in BOLT folder `008`.
