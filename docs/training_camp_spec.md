# Training Camp Mode for ESPN NFL 2K5 — Design Spec

Target project: `cruuz/2k-football-mod-tools` (2K5 Mod Studio), built as an
experimental, default-off feature that follows the project's CONTRIBUTING.md
proof ladder. This brief uses the project's own labels:

- **PROVED (per release notes):** stated as shipped or proved in a named beta.
- **TO VERIFY:** must be confirmed against the current code/binary first.
- **HYPOTHESIS:** a proposed design that needs an in-game witness.

Read CONTRIBUTING.md and the latest release notes before starting. Do not
treat anything below as more proved than its label says.

---

## 1. Goal

A playable preseason training camp inside Franchise mode. Between the offseason
and the preseason, the user runs a series of camp sessions with the real
roster. Performance in camp is tracked per player, turned into camp grades,
nudges ratings and the depth chart, and feeds a roster cut deadline before the
season. The feel to aim for is a "training camp documentary" season opener:
position battles, surprise rookies, hard cuts.

## 2. Design principles

1. **Reuse retail systems first.** Practice, rosters, depth chart, release,
   and Coach's Desk already exist. New code should connect them, not replace them.
2. **Experimental and default off**, like every recent native feature, with a
   witness list in the report.
3. **Changes happen inside the running game.** Ratings and roster changes are
   made in memory and saved by the game's own save routine, which avoids the
   unsolved external save-signing problem.
4. **Small state footprint.** Owned executable space is limited (see §4), so
   camp state must be compact and bounded.
5. **Ship in phases**, each independently playable and witnessable.

## 3. Player experience (target flow)

1. After the draft and free agency, Franchise enters **Training Camp** instead
   of going straight to preseason.
2. The Coach's Desk shows a **Camp** entry with the current day (e.g. "Camp
   Day 3 of 8") and the cut deadline.
3. Each camp day the user runs one session:
   - Phase 1: a team scrimmage (existing Free Practice).
   - Later phases: 7-on-7, position drills, 1-on-1 periods.
4. After each session, a **Camp Report** lists standout and struggling players
   by position group and current position battles.
5. At camp end, grades apply small rating changes and depth chart suggestions.
6. **Cut deadline:** the season cannot begin until the active roster meets the
   limit. The user releases players or moves them to reserves.
7. CPU teams make their own cuts, which creates a pool of newly released players.

## 4. Known building blocks

| Block | Status | Source |
|---|---|---|
| Free Practice inside Franchise: team scrimmages itself with exact roster, ratings and depth chart, returns to the desk | PROVED (shipped) | beta-58 |
| Free Practice does **not** record stats or injuries | PROVED (stated limitation) | beta-58 |
| Coach's Desk order Schedule, Practice; reserves drawn into the practice roster | PROVED (shipped) | beta-61 |
| Reserves: 53 active, 12 reserve, 65 total, atomic Promote/Demote in the Studio | PROVED (shipped, Studio side) | beta-61 |
| Depth-chart locks honored by the weekly sort | PROVED (shipped, experimental) | beta-61 |
| Owned executable space: two code pages + one writable page; about 6,501 bytes of owner code and 3,242 bytes of state at that release; allocator flags default off pending witness | PROVED offline, in-game witness pending at beta-61 | beta-61 |
| 7-on-7 practice mode built and debugged through xemu gdb stub, kept hidden until witnessed | PROVED built, hidden | beta-57 / beta-61 |
| Basic Training tutorial with drill-style events exists in 2K5 | PROVED as retail content (per APF lineage research) | phase2 docs |
| Franchise 2026 rules memo: real cutdown and trade deadline need native code plus new storage; team record cannot widen in place | PROVED (research memo) | beta-61 |
| MyCareer mode exists in later betas | PROVED (shipped) | later betas; check current state |

Check current values (capacity, flags, witness status) against the latest
release. The allocator and owned pages may have been witnessed since beta-61.

## 5. Phase 0 — Research (TO VERIFY before writing code)

Answer each with an address, a Ghidra trace, and a reproducible check:

1. **Franchise calendar state:** where the current phase/week is stored, and
   where the offseason → preseason transition happens. This is the camp entry hook.
2. **Free Practice lifecycle:** entry point, per-play loop, and the return-to-
   desk path. The end-of-session hook goes on the return path.
3. **Per-play stat accumulation:** in regular games, where per-player stats
   (catches, targets, rush yards, sacks, tackles, drops, completions) are
   accumulated, and why Free Practice skips it. Goal: redirect or mirror those
   events into a camp buffer without touching season stats.
4. **Roster limit enforcement:** does retail 2K5 already require 53 actives
   before week 1? If yes, camp only adds evaluation. If no, the cut deadline is
   new gating code.
5. **Rating write path:** the in-memory player record fields for position-
   relevant ratings, and whether changes there persist through a normal save.
6. **UI surface for the Camp Report:** is there an existing list/sheet screen
   (e.g. a Team Rosters sheet clone, as the practice squad memo suggests) that can
   show text rows? If not, Phase 1 can show the report through an existing
   news/ticker/text surface.
7. **CPU roster moves:** where CPU teams trim rosters today (if at all).

## 6. Phase 1 — MVP (camp sessions + tracking + report)

- **Camp state** (in owned writable space):
  - `camp_active` (1 byte), `camp_day` (1 byte), `camp_days_total` (1 byte, default 8)
  - Per-player camp counters for the user's team only.
- **Session:** launching Camp runs the existing Free Practice scrimmage.
- **Tracking:** hook the stat accumulation identified in Phase 0 so events
  during a camp session increment the player's camp counters.
- **End of session:** increment `camp_day`, compute running grades, update the
  Camp Report.
- **Exit:** after the last day, set `camp_active = 0` and continue to the
  existing preseason flow.

### State budget (HYPOTHESIS, to confirm against current free space)

65 players x 12 bytes = 780 bytes, for example:
- 8 x 1-byte saturating event counters (position-agnostic slots, meaning set
  per position group)
- 1 x 2-byte running grade (fixed point)
- 1 x 1-byte sessions played
- 1 x 1-byte flags (standout / struggling / position battle)

Plus about 16 bytes of global camp state. Well under the beta-61 writable
capacity, but that space is shared with other owners (e.g. the scorebug
runtime), so request a named allocation through the allocator.

**Persistence caveat:** if camp spans multiple sessions with saves in between,
camp state must survive save/load. Phase 1 may keep camp within one sitting,
or store it in spare fields the Phase 0 research finds. Mark this explicitly.

## 7. Phase 2 — Consequences

- **Grades → ratings:** at camp end, apply bounded deltas to position-relevant
  ratings, e.g. -2 to +3, weighted toward young players, never above the
  retail max. HYPOTHESIS: tune after play testing.
- **Depth chart:** produce suggestions; never override depth-chart locks (beta-61).
- **Report text:** "Rookie WR [name] won the slot job", "[name] fell behind in
  the LB battle".

## 7b. Position battles (part of camp, Phase 2)

**Detection at camp start (HYPOTHESIS):** for each starting depth-chart slot,
flag a battle when a backup is within a small overall gap of the starter, or
is a high-potential rookie. Cap at about 8 battles per team.

**Rep sharing (key technical piece, TO VERIFY):** Free Practice uses the depth
chart, so backups may never play. Rotate the depth chart between sessions or
periods (swap starter and challenger) so both get snaps, and restore it
afterwards. Must respect or temporarily lift depth-chart locks (beta-61).

**Scoring:** compare grades per snap, not totals, so fewer reps are not
penalized. For QB battles on the user's team, the user's own play decides it.

**Resolution at camp end:**
- User team: a "Coach's decision" step showing each battle, the scores, and a
  recommendation; the user confirms the starter.
- CPU teams: winner moved to starter automatically.
- Camp Report shows battles all camp ("[name] leads the RB battle").
- Optional: a small rating bump for the winner; the loser flagged "on the
  bubble" if near the cut line.

**State (HYPOTHESIS):** per battle, two player IDs, two running scores, and
snap counts; about 8 battles x ~12 bytes, under 100 bytes.

**Witnesses:** both players get snaps in camp sessions; the depth chart is
restored after each session; the confirmed winner starts week 1.

## 8. Phase 3 — Cut deadline

- Gate the transition out of camp until the active roster meets the limit,
  reusing retail release and reserve operations.
- Show players "on the bubble" from camp grades.
- CPU teams: trim by rating plus camp grade, sending released players to free agency.
- Note the beta-61 memo: modern cutdown rules need native code and new
  storage. Keep Phase 3 to the retail roster structure (53 active / 12 reserve).

## 9. Phase 4 — Drills and periods

- 7-on-7 period using the hidden 7-on-7 mode (after it is witnessed).
- Position drills built on the Basic Training event framework, if it is
  reachable and adaptable (TO VERIFY).
- Each drill feeds the same camp counters.

## 10. Phase 5 — Flavor

- Camp news items in existing text surfaces.
- Reuse existing commentary or presentation audio where a fitting cue exists.
- Optional: name the mode "Training Camp" (avoid third-party show names).

## 10b. Phase 6 — Weekly Game Prep (in-season extension)

Reuses the camp tracking and grading code during the regular season.

**Flow:** Coach's Desk shows **Game Prep** before each game. The user picks
an Offense period (their offense vs a scout defense running the next
opponent's defensive book) or a Defense period (their defense vs a scout
offense running the opponent's offensive book). The scout team is the user's
own backups and reserves, as in a real NFL practice week.

**Mechanism (HYPOTHESIS):**
- Session is Free Practice with the scout side's playbook swapped to the next
  opponent's book (2K5 has 32 team books plus GEN, per beta-61).
- Opponent comes from the franchise schedule for the current week.

**Grading per play (HYPOTHESIS, tune in play testing):**
- Offense: success rate (≥40% of yards-to-go on 1st down, ≥60% on 2nd, 100% on
  3rd/4th), explosive plays, sacks, turnovers.
- Defense: stops by the same success rule, pressures/sacks, takeaways,
  explosive plays allowed.
- Result: letter grade per side plus notes ("struggled vs Cover 2 looks").

**Reward (HYPOTHESIS):** a small, temporary game-prep bonus for the next game
only, scaled by grade (e.g. +1 to +3 awareness/reaction-type ratings for the
starters involved). Applied at kickoff, reverted at game end.

**Phase 0 additions (TO VERIFY):**
1. Where a practice session assigns each side's playbook, and whether the scout
   side can take another team's book index.
2. Where the CPU practice side picks plays (does it use the book's own
   call logic or random selection?).
3. Game start and game end hooks for applying and reverting the bonus.
4. The revert must also happen if the user quits mid-game or a save/load
   occurs; otherwise bonuses stack permanently. This is the main risk.

**Witnesses:**
- Scout side visibly runs formations from the opponent's book.
- Grade changes with results; bonus applied at kickoff (debugger check),
  reverted after the game, and not present after quit-to-menu or reload.
- Regular-season stats unaffected by prep sessions.

## 10c. Phase 7 — Practice schedule, periods, intensity, and injuries

Turns camp days and game-prep days into structured practices, like a real
NFL practice script.

**Daily practice script (user-configurable, defaults per day type):**
1. Walkthrough (optional, no grading, no injury risk)
2. On-air: offense runs plays with no defense; grades timing, completions,
   ball placement (HYPOTHESIS: needs a defense-less practice setup, TO VERIFY)
3. 1-on-1 periods: WR vs DB (QB + 1 WR + 1 DB); later OL vs DL pass rush
4. 7-on-7: built on the hidden 7-on-7 mode (after it is witnessed)
5. Team period: full 11-on-11 (existing Free Practice)

Each period is a short set of plays (e.g. 6–10) feeding the same camp or
game-prep counters. MVP: the day's script is a checklist on the Coach's Desk
and each period launches separately. Later: periods chain automatically.

**Intensity per day:** Walkthrough / Shells / Full pads.
- Higher intensity = larger grade weight and development gains, higher
  injury risk. Realistic trade-off: full pads in camp, lighter in season.

**Practice injuries (opt-in, low rates, HYPOTHESIS):**
- Option A: let the game's own injury logic run during practice, if Free
  Practice suppresses it with a flag or branch (TO VERIFY; beta-58 says Free
  Practice does not record injuries).
- Option B: a separate end-of-period roll per participating player, scaled by
  intensity and contact type, writing a normal retail injury so the existing
  injury and substitution systems handle it.
- Injuries feed the weekly injury report (did not practice / limited / full).
- Must have a slider or off switch; default off.

**New Phase 0 items (TO VERIFY):**
1. How the hidden 7-on-7 mode reduces the player count, and whether the same
   mechanism can produce 1-on-1 (3 players) and on-air (offense only) setups.
2. Where Free Practice skips injuries, and the retail injury write path
   (status, duration) so practice injuries are real, bounded injuries.
3. Whether a practice session can be restricted to a play subset (routes only,
   pass plays only) for on-air and 1-on-1 periods.

**Witnesses:**
- Each period type launches, shows the right number of players, and returns.
- Period results feed grades correctly.
- Practice injury at a forced high rate: appears in the injury system, player
  is unavailable per the duration, and the rate returns to the slider value.

## 10d. Phase 8 — Scouting Combine (watch or do)

**Starting point:** the beta-61 Senior Bowl memo states the offseason already
has scouting and combine hooks (PROVED as a research finding). What retail
shows today, and how prospect data is stored, is TO VERIFY.

**Watch mode (first):**
- Combine days run event by event: 40-yard dash, bench, vertical, broad jump,
  3-cone, shuttle, plus position drills.
- Results reveal as a live results feed or board, with standouts and busts.
- Measured values = true hidden ratings + bounded noise, so the combine
  informs the draft but does not reveal ratings exactly.

**Do mode (later):**
- Playable events only where the game has fitting motion: 40-yard dash and
  shuttle (running), WR/TE gauntlet and DB drills via the 1-on-1 practice setup.
- Times should come mostly from the prospect's ratings through the game's own
  movement physics; the user's input matters only at the margins (start,
  cuts), so player skill does not distort draft grades.
- Bench, vertical, broad jump: simulated, since there are no animations.

**Phase 0 items (TO VERIFY):**
1. How retail 2K5 represents draft prospects before the draft, and whether they
   are full player records that can be put on a field.
2. Existing combine data fields and where retail generates them.
3. Whether a practice session can load non-roster (prospect) players.

**Witnesses:**
- Watch mode: results shown for every prospect, consistent with their
  ratings plus noise, and carried into draft screens.
- Do mode: a prospect appears on the field, runs the 40, and gets a time that
  tracks his speed rating across repeated runs.

## 10e. Practice uniforms

**Practice jerseys (achievable):**
- Uniform textures are already writable (jersey, pants, helmet colors, per
  earlier betas). Author practice jerseys: offense in one color, defense in
  another, mesh-style texture, red no-contact jersey for QBs.
- Needs a practice uniform slot per team. Options: repurpose one alternate or
  throwback uniform slot per team, or a new slot if archive growth allows.
- Hook: practice, camp, and game-prep sessions select the practice slot
  (TO VERIFY: where Free Practice chooses uniforms; beta-58 added home/away
  jersey at any stadium, so a selector exists somewhere).

**Guardian caps:** beta-61 shipped an experimental cap-shaped helmet (sculpted
within fixed spans). Per-player or practice-only caps fit here once the
per-player route exists.

**Shorts (hard):**
- Texture-only: paint the pants lower half as skin and socks. Problem: one
  texture cannot match every player's skin tone (TO VERIFY: whether 2K5 tints
  exposed skin per player on pants or sock regions).
- Model route: shorter pants shape plus exposed legs means geometry changes,
  which are limited to same-footprint sculpting today.
- Recommendation: jerseys first, shorts only if a skin-tone channel exists.

## 10f. Phase 9 — Senior Bowl extensions

**Update:** SoftDrinkTV announced on Sep 5, 2026 that a Senior Bowl has been
added to ESPN NFL 2K5 (alongside dynamic kickoff, an 18-week schedule,
7-seed playoffs, new OT rules, new playbooks, and Superstar Abilities). Do not
rebuild it. First find out what his version does:

**TO VERIFY in the current repo/release:**
1. Is it playable or simulated?
2. Are prospects put on the field as full player records? If yes, this also
   answers the combine "do mode" blocker (§10d item 1).
3. Does it replace the Pro Bowl slot, and does it affect draft stock?

**Possible extensions (only if not already present):**
- A graded Senior Bowl practice week using the Phase 7 practice periods
  (1-on-1s, 7-on-7), since real scouts learn the most in those practices.
- Performances raise or lower draft stock and narrow uncertainty on
  prospect ratings, shared with the combine.

## 11. Verification and witness list

Offline gates (must pass, per project rules):
- Retail byte pins, memory-write gate, cave-reference gate, owner composition.
- Independent verifier for every new writer.
- Real image build of the experimental preset, plus a build with all opt-ins on.

In-game witnesses (xemu), each recorded in the report:
1. Franchise enters Camp after the offseason; Coach's Desk shows the Camp entry.
2. A camp session launches, plays, and returns to the desk.
3. Camp counters change for players who made plays (debugger memory check).
4. Camp Report shows sensible standouts after two sessions.
5. After the final day, the game proceeds to preseason with no crash.
6. Phase 2: ratings changed within bounds and persisted through save and reload.
7. Phase 3: season start is blocked until roster compliance; CPU teams comply.
8. Regular-season stats are unaffected by camp sessions.

## 12. Out of scope (for now)

- New animations, cutscenes, interviews, or voice lines.
- Modern 2026 roster rules (47/48 actives, 16-man practice squad, IR returns),
  already queued in the project's "Franchise 2026 rules" memo.
- APF 2K8.

## 13. Open questions for the maintainers

1. Current state of the allocator and owned-page witness.
2. Whether the "Franchise 2026 rules" work would own the cut-deadline logic, so
   camp can plug into it instead of duplicating it.
3. Preferred UI surface for new text screens.
4. Whether camp should also work for MyCareer (player-only camp view).

## 14. Suggested first prompt for Claude Code

> Read CONTRIBUTING.md, the latest release notes, and this spec. Do Phase 0
> only. For items 1–3, find the franchise calendar transition, the Free
> Practice lifecycle, and the per-play stat accumulation path in the 2K5 XBE
> using the repo's Ghidra scripts. Report addresses, evidence, and confidence
> using PROVED / HYPOTHESIS labels. Do not write any patch yet.

## 15. Phase 10 — APF 2K8-style timed kicking (added last)

Replace 2K5's fill-and-stop kick meter with a kick you have to **time**, in
the style of All-Pro Football 2K8. Applies to field goals, PATs, punts and
kickoffs. Experimental and default off, like every other feature here. (§12
puts "APF 2K8" out of scope. That means porting the APF game itself. This
section only re-creates one APF mechanic natively in 2K5.)

**Target feel (HYPOTHESIS: confirm against APF 2K8 in Xenia before building):**
- A timing window, not a "stop the bar at max" meter. Power comes from the
  swing and accuracy from timing. A perfect press in the sweet spot gives a
  straight kick at the kicker's rated distance. Early or late presses hook or
  slice, and the error grows with distance from the sweet spot.
- Optional analog mode (APF-style swing): pull the right stick back to start
  the run-up, then flick it forward. Power comes from the flick and accuracy
  from how straight it is.
- Kicker ratings set the size of the window: KAC (accuracy) widens the sweet
  spot, KPW (power) raises max distance. Pressure moments (late game, long
  attempts, icing) can shrink the window. Options: off, on, user-only.
- The CPU keeps its current range logic, so CPU kickers are unaffected.

**What upstream already has (Beta 76.4; from repo research, labels are the repo's):**

| Block | Status | Where |
|---|---|---|
| Meter fill → distance: human launch `FUN_003147a0` reads curve `0x50B4F0` (fill→yards) × kicker curve `0x50B514` at `KPW - 0.2*(1-KAC)`, minus rand×4 yd | PROVED offline (unicorn), not witnessed | `mod_editor/core/nfl2k5_kick_rules.py` |
| Kickoff curve `0x50B980`, punt curve `0x50C0F8`; CPU range `FUN_0018b120` reads the same tables | PROVED offline | same |
| Kick HUD init `FUN_000ba940` (KickArrow / KickMeter / windmeter scenes), draw `FUN_000bb2c0`, curve sampler `0x2F010` | PROVED offline | `nfl2k5_kick_meter_2026.py`, `nfl2k5_hud_layout.py`, `reports/b76_km/` |
| Place-kick flow: `2F15C0` waits → meter prep `2EE950` → run-up `2F0DD0` (fed by latched meter completion) | PROVED offline, but the meter is supplied externally in tests | `ASTRA_KICKOFF_V4_REPORT.md` |
| Ball launch `0x222CA0`, release call `0x222D01`, aim hook `0x222E67` | used by dynamic kickoff (experimental, unwitnessed) | `nfl2k5_dynamic_kickoff.py` |
| Controller: command context 2 = kick meter (commands 0x37..0x3B, 0x3D/0x3E); held-command reader `0x120960`; XInput decoder `0x39480` (analog at +0x24) | bytes PROVED, command meanings HYPOTHESIS | `FABLE_MYCAREER_REPORT.md`, `nfl2k5_read_option_runtime.py` |
| APF 2K8 kick mechanic research | **none in repo** | — |

**Phase 0 items (TO VERIFY):**
1. The address and state of the meter's value, and the per-frame code that
   sweeps it and latches it on button press (the consumer of the context-2
   commands). This is the main unknown.
2. Where the aim/accuracy result is applied at launch (`FUN_003147a0`,
   `0x222E67`). Today accuracy appears to be only the KAC term in distance,
   with no lateral error model documented.
3. Record the APF 2K8 kick behaviour in Xenia (video plus notes): button
   sequence, window size and how ratings and pressure change it. No XEX reverse
   engineering is needed for a feel-alike.
4. Whether the existing KickMeter scene (13 submeshes and MAX window) can be
   driven to show a timing marker, or whether a new HUD runtime is needed
   (the scorebug runtime is an allocator-based pattern).

**Mechanism (HYPOTHESIS):**
- Hook the meter latch: when the button is pressed, record the frame offset
  from the sweet spot instead of (or alongside) the fill value.
- Map timing error to (a) a distance multiplier through the existing curve
  `0x50B4F0` and (b) a lateral launch-angle offset applied at the aim hook,
  scaled by KAC.
- State: under 16 bytes (mode flag, sweet-spot frame, latched error, pressure
  scale), in a named allocator block.

**Witnesses:**
- A perfect-timing FG from 30 yd goes straight; early and late presses miss
  left and right consistently.
- Max distance still tracks KPW across kickers; a low-KAC kicker has a visibly
  narrower window.
- Punts and kickoffs use the same timing; the dynamic kickoff still works
  when both are on.
- With the flag off, kicking is byte-identical to retail.
