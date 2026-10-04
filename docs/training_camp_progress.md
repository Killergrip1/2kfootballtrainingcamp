# Training Camp — Progress Log

Upstream baseline: `cruuz/2k-football-mod-tools` @ `5232f391` (Beta 76.4 / RC109,
2026-10-03). All addresses are USA retail `default.xbe` VAs. Labels are the
upstream repo's own (PROVED / PROVED OFFLINE / WITNESSED / HYPOTHESIS /
EXPERIMENTAL-UNWITNESSED); nothing below is witnessed by us yet.

---

## Session 1 — Orientation (2026-10-04, no code)

### What was done
- Added the design spec (`docs/training_camp_spec.md`) and brief
  (`docs/CLAUDE_CODE_PROMPT.md`) to this repo.
- Added **APF 2K8-style timed kicking** as the last feature (spec §15, brief
  backlog).
- Read upstream CONTRIBUTING, STATUS, roadmap, changelog and the feature
  modules/reports listed below.
- **Not done:** building the preset or booting it in xemu. The retail ISO is on
  the user's Windows PC, and this cloud session can't reach it. That step has
  to run on the user's machine (see "Next step").

### Where the current notes live
`BETA_RELEASE_NOTES.md` and the root `CHANGELOG.md` are stale (they stop around
beta-63). The current release notes are
`docs/mod_editor/2k5_mod_studio_changelog.md`. `STATUS.md` jumps from 76.4
straight back to 63.

### Proof ladder (CONTRIBUTING.md)
`unknown → read-only-mapped → extract-only → offline-writer-proved → runtime-proved`.
- Every writer ships with an **independent verifier** that re-derives the
  container instead of importing the writer's parser.
- Never write to the user's original; fail closed; no game data, ever. The
  retail-free gate enforces the last rule.
- If you edit a pinned module: `python3 packaging/repin.py --apply`.
- A new native feature must be experimental and off in every preset. It must
  pass the retail byte pins, the memory-write and cave-reference gates, and
  pairwise owner composition in both orders. It also needs a real
  Experimental build and an all-opt-ins build.
- PR template sections: "What I proved, and how" and "What I did NOT prove".
- Capability registry: `mod_editor/capabilities/ROADMAP.md`, checked by
  `bash mod_editor/capabilities/validate.sh`.

### State of the building blocks
| Block | State at 76.4 | Where |
|---|---|---|
| Free Practice in Franchise (`franchise_practice`) | Return to Desk **WITNESSED** (Noah). On in Advanced and Experimental. Module still says `runtime_verified: False`. | `mod_editor/core/nfl2k5_franchise_practice.py` |
| Free Practice stats/injuries | Practice runs as mode 1; stat, clock and injury paths are gated on mode ≥ 4 (docstring claim, not separately PROVED). Season block never written. | same |
| Coach's Desk | Descriptor `0x522190`, 11 rows × 0x34, terminator `0x52215C`, **no spare row slot**. Practice row `0x521FBC`. Practice Squad screen already replaces The Crib row. | same, `nfl2k5_practice_squad_screen.py` |
| 7-on-7 (`seven_on_seven`) | v2 released as opt-in, off in every preset, **UNWITNESSED** (huddle break still on the witness list). Practice Type 4, flag `0xA69970`, loader hook `0x62D0C`. | `nfl2k5_seven_on_seven.py` |
| MyCareer | Off in presets. First cut WITNESSED 9/8 (route, creation, a real game). Overall EXPERIMENTAL. 20,480 RX. | `nfl2k5_my_career*.py`, `tools/mycareer_mode/` |
| Owned space (allocator) | 106,496 RX / 86,016 RW / 20,480 RO. With the full owner union: **192 RX / 0 RW / 584 RO free.** Loader is formally UNWITNESSED, though MyCareer code in owned pages has run in xemu (inferred). | `nfl2k5_xbe_space.py`, `ASTRA_ALLOCATOR_SCALEOUT_REPORT.md` |
| Franchise calendar | Stage `0xE576A4` (1 retire … 4 Combine, 5 draft, 6 signing, **7 preseason**, 8 regular, 9 post). Stage advance `FUN_002480B0`; Signing→Preseason case at `0x2484E7` → preseason gen `0x2BEC20`. Week advance `0x247D40`. | `nfl2k5_franchise_save.py`, `nfl2k5_calendar_engine.py` |
| Roster gate | Stage 7 runs the retail roster/cap gate (54-player message). Native cutdown `0x2BFAA0`, release `0x2BD900`. | `tools/practice_squad/AUDIT.md` |
| Depth-chart locks | Player +0x52 bits 0..4, honored by compactor `0x243790`. On in Experimental, UNWITNESSED. | `nfl2k5_depth_locks.py` |
| Reserves 53+12 | Gate-proved, UNWITNESSED. | `nfl2k5_practice_squad.py`, `nfl2k5_practice_reserves.py` |
| Franchise 2026 rules | Host-side kernel only, `RUNTIME_READY=False`; blocked on owned save storage. | `nfl2k5_franchise_2026.py` |
| Senior Bowl | **Neither playable nor simulated** (`NATIVE_EVENT_AVAILABLE=False`); dormant reserve 4K RX + 64K RW; no prospects on field. | `nfl2k5_senior_bowl.py` |
| Weekly Preparation (retail) | Retail 7-day plans, 173 drills, temporary bonuses. Upstream patch PROVED OFFLINE, off. **Closest thing to Game Prep.** | `nfl2k5_weekly_prep.py` |
| Practice playbooks | Side setters `0x77AE0` / `0x77B20` install each side's book. | `nfl2k5_franchise_practice.py`, `nfl2k5_playbook_pair.py` |
| Stats | Commit `0x1356D0` (from `0x10B9C0`); MyCareer hooks `0xC5D9E`. | `docs/mod_editor/career_stats.md` |
| Injuries | Player +0x28 & 0x3E0, duration +0x20 bits 22..29; CPU injury/IR `0x2BE020` (PROVED). | `ASTRA_B66_2K5_GAME_REPORT.md` |
| Ratings | Player bytes +0x36..+0x51. Only +0x53 bits 6–7 are free. | `nfl2k5_roster_records.py` |
| Uniforms | Builder `FUN_000615A0`; Basic Training already uses practice kits 31h0/31a0. | `nfl2k5_uniform_choice.py`, `nfl2k5_team_logo_swap.py` |
| Save extension precedent | MyCareer appends a 128-byte signed `MCPL0001` footer after the native save. | `nfl2k5_my_career_save.py` |

### Not upstream (ours to build)
Training camp, camp tracking, position battles, a playable combine,
opponent-playbook Game Prep, practice jerseys in Franchise, targeting penalty,
timed kicking.

### Template to copy: how Free Practice was added
1. Pure-Python patcher module in `mod_editor/core/`. The docstring states the
   contract, and the module has `status()`.
2. `BuildPlan` flag in `mod_build.py`, a GUI row in
   `gui/build_panel_qt.py`, and preset membership.
3. Code goes in **dead retail caves** (352 bytes at `0x1D82D0`), not the
   allocator. Only one retail instruction changed (the call at `0x2C0E7C`).
4. Verifiers in `tests/mod_editor/test_nfl2k5_franchise_practice*.py`: pins,
   unreferenced cave, Unicorn stubs, order independence. Plus the global
   memory-write, cave-reference and pairwise gates.

### Build + xemu workflow (to run on the user's PC)
- GUI: `2K5-Mod-Studio.bat` → Open game disc (accepts the `.iso` directly,
  identifies retail/repack and refuses pre-modded images) → ★ Build & Share →
  Build → **Experimental** preset → Save disc copy as → **Make my disc** →
  Play latest disc in xemu. xemu needs `mem_limit = '128'`.
- Headless:
  ```python
  from mod_editor.core import mod_build; import dataclasses
  plan = mod_build.apply_preset(mod_build.BuildPlan(source="RETAIL.iso", target="OUT.xiso.iso"), "softdrink_experimental")
  # plan = dataclasses.replace(plan, seven_on_seven=True)   # extra opt-ins
  mod_build.build(plan)
  ```
- Debug: `xemu -dvd_path OUT.xiso.iso -gdb tcp:127.0.0.1:1234`, then
  `gdb -q -nx -ex 'target remote 127.0.0.1:1234'`. Bind localhost only:
  connecting halts the VM.

### Proof status (ours)
- PROVED: nothing yet. Everything above is upstream's claim, read from their repo.
- TO VERIFY: the baseline Experimental build boots in xemu from the user's dump.

### Risks / blockers
1. **No free owned state space (0 RW bytes).** Camp needs about 1 KB RW. Options:
   - (a) Ask the maintainers to split off part of the dormant Senior Bowl 64K RW
     reserve.
   - (b) Use dead retail caves for code (like Free Practice) and find dead
     `.data` for state.
   - (c) Use the K128 path (xemu only).

   **Needs a decision from the user and the maintainers** (brief rule 10).
2. **No spare Coach's Desk row.** Camp must either take over the Practice row
   during stage 7 or relocate the row table.
3. **Camp state persistence:** no owned save storage yet. Either keep Phase 1
   within one sitting, or follow MyCareer's signed save-footer precedent.
4. Practice is mode 1, so stat accumulation won't fire. Camp tracking needs
   its own hook on the play-result path.

### Next step
User runs a session **on their own PC** (Claude Desktop app, or
`claude remote-control` in a terminal), builds the `softdrink_experimental`
preset from their ISO, and boots it in xemu. Witness: Franchise → Coach's Desk
→ Practice → run plays → quit back to the Desk. Record the result and the build
receipt here.
