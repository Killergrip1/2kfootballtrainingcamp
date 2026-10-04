# Claude Code Brief: ESPN NFL 2K5 — Training Camp and Beyond

## Who I am and what I want

I'm a modder working on ESPN NFL 2K5 (original Xbox) using the open-source
`cruuz/2k-football-mod-tools` project (2K5 Mod Studio by SoftDrinkTV). I want to
add features that make 2K5 the most immersive football sim possible, starting
with a playable **Training Camp** and **weekly interactive practice** inside
Franchise mode. The full design is in `training_camp_spec.md` in the repo root.
Treat that spec as the plan, but treat the repo and latest release notes as the
truth when they disagree.

## Ground rules (always follow)

1. **Follow CONTRIBUTING.md and the project's proof ladder.** Every change that
   writes to the game needs an independent verifier. New features are
   experimental and default off until witnessed in xemu.
2. **Label every claim** PROVED, HYPOTHESIS, or TO VERIFY, like the project does.
   Never present a guess as fact. If you don't know, say so and propose how to
   find out.
3. **Never ship or commit game data.** Work only from my own retail dump. Never
   copy retail bytes, assets, or decompiled retail code into the repo beyond
   what the project already allows (addresses, pins, hashes, offsets).
4. **Reuse before building.** Check whether a feature or building block already
   exists before writing new code. The project moves fast: Free Practice,
   MyCareer, Senior Bowl, 18-week schedule, 7-seed playoffs, new OT rules,
   dynamic kickoff, and Superstar Abilities already exist in recent betas.
5. **Follow existing patterns.** Study how MyCareer and Free Practice were added
   (mode state, menu hooks, screen transitions, owned code space, verifiers) and
   structure my code the same way so it can be merged upstream.
6. **Small steps.** Each step should be buildable and testable in xemu the same
   day. Don't touch unrelated features.
7. **I run the game.** You can't see xemu. When a step needs an in-game test,
   give me exact steps (what to build, what to click, what to look for, what
   memory addresses to check in the gdb stub), then wait for my report.
8. **Work on a branch:** `feature/training-camp`.
9. **Keep a log.** Maintain `docs/training_camp_progress.md`: what was done,
   what was proved, what failed, open questions, next step. Update it at the end
   of every session.
10. **Stop and ask** before large design decisions, anything that grows shared
    owned code/state space significantly, or anything that changes existing
    behavior when my feature is turned off.

## Session 1: Orientation (do this first, no code)

1. Read CONTRIBUTING.md, STATUS.md, the capability roadmap, the latest release
   notes, and `training_camp_spec.md`.
2. Report:
   - What's PROVED about: Free Practice, the Coach's Desk, the hidden 7-on-7
     mode, the owned code/state space (capacity, free space, witness status),
     franchise calendar hooks, depth-chart locks, reserves, Senior Bowl.
   - How MyCareer and Free Practice were implemented (files, tools, patch
     points, verifiers) as a template for my feature.
   - Which parts of my spec already exist upstream, so I don't rebuild them.
   - The exact build + xemu test workflow for an experimental feature.
3. Help me build the experimental preset from my own dump and boot it in xemu
   before anything else.

## Session 2: Research (Phase 0 of the spec, no patches yet)

Using the repo's Ghidra scripts and headless Ghidra on my `default.xbe`, find
and report with addresses, evidence, and confidence:

1. The franchise calendar state and the offseason → preseason transition.
2. The Free Practice lifecycle: entry, per-play loop, return-to-desk path.
3. Where per-player stats are accumulated in real games and why Free Practice
   skips them.
4. Whether retail enforces the 53-man limit before week 1.
5. The in-memory rating fields and whether in-game changes persist through a
   normal save.
6. Where a practice session assigns each side's playbook.

## Weekend milestone: playable camp loop

1. A **Camp** entry on the Coach's Desk during the preseason window.
2. Selecting it launches a Free Practice session.
3. A camp day counter (default 8 days) advances when a session returns.
4. After the last day, camp ends and Franchise continues to the preseason.
5. Feature flag default off; verifier written; full loop witnessed in xemu.

Fallbacks: if the owned code space isn't witnessed yet, proving it is the
first job. If the stat hook is hard, ship the loop without grades first.

## Next, in order (see spec for details)

1. Per-player camp tracking, grades, and a Camp Report.
2. Position battles: detect close battles, rotate the depth chart so both
   players get reps, compare grades per snap, "Coach's decision" at camp end.
3. Rating changes from camp (bounded), depth chart suggestions, cut deadline,
   CPU teams make cuts.
4. Weekly Game Prep: practice against the next opponent's offensive and
   defensive playbooks with a scout team; graded; small temporary game bonus
   that is always reverted (after the game, quit-to-menu, and reload).
5. Practice schedule periods: on-air, 1-on-1 WR vs DB, 7-on-7, team period.
   Intensity (walkthrough / shells / full pads). Practice injuries (opt-in,
   slider). Weekly injury report (DNP / limited / full).
6. Practice uniforms: practice jerseys, red no-contact QB jerseys, Guardian
   caps where possible. (Shorts only if 2K5 tints skin per player.)
7. Combine: watch mode first (results = hidden ratings + noise), do mode later
   if prospects can be placed on the field. Check the existing Senior Bowl
   first: if it puts prospects on the field, that answers this.
8. Senior Bowl extensions only if missing upstream: graded practice week,
   performance moves draft stock.

## Backlog (later projects, research first)

- **Targeting as hit-stick risk/reward:** big hits on defenseless players can
  force fumbles but may draw targeting. 15 yards + automatic first down.
  First-half foul: out for the rest of the game. Second-half foul: rest of game
  plus first half of the next game (needs saved state + kickoff/halftime
  hooks). CPU defenders follow the same rule. Settings: on/off, frequency,
  college vs NFL rules. Optional instant replay of the hit before the call.
- **Crowd behavior:** crowd builds continuously on big runs and peaks at the
  TD; realistic reactions when the away team scores (may be an audio swap).
- **SportsCenter:** more than 1–3 games selected and more highlight clips per
  week; find the selection logic and whether the counts are simple constants.
- **Stadiums:** unlock the 15 hidden stadiums (82 exist, 67 selectable); a
  general same-size geometry writer with compression-fit checks so real
  stadiums can be shrinkwrapped onto existing meshes in Blender.
- **Animations:** research per-QB throwing motions and importing APF 2K8 /
  2K6-era animations, starting with one throw replacing one existing slot.
- **Targeted decomp:** reverse-engineer only the systems blocking these
  features (model loading, animation selection, crowd logic, SportsCenter
  selection, penalties), rewrite in C, and inject into owned code space.

## How to report each session

End every session with:
1. What was done.
2. What is PROVED vs HYPOTHESIS vs TO VERIFY.
3. Exact xemu test steps for me, if any.
4. Risks or blockers.
5. The single next step.
