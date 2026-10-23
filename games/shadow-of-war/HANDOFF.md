# Shadow of War co-op: handoff (2026-10-22, after session ah)

Read this first, then the "Status summary 2026-10-21", session ag and session ah at the end
of [NOTES.md](NOTES.md).

**Where we are:** phase 5 (build), first co-op step working one way on one PC. Copy A
streams its player over loopback UDP every frame: position, rotation, health, the 25
inputs of the player's animation graph (stick, speed, turning, camera) and its root-motion
speed. Copy B shows a spawned Talion stand-in, AI off, moved by A's inputs through its own
animation (`coop_puppet_move=anim`): it walks, turns and stops with A's stride, feet on the
ground, and is put back only when it drifts past 150 units or once A stands still. When A
free-runs (A button), the stand-in's graph still only runs, so its root motion is scaled to
A's speed (`coop_anim_speed`, on by default): on flat ground it keeps within ~10 units on
average instead of falling 300 behind. Not carried yet: vaults and running jumps, attacks,
anything from B to A, and world state (enemies, Nemesis, quests, time).

**Build and tools:** game build 3095001 only (Steam app 356190). Our guard DLL
(`bink2w64.dll` proxy) and probe live in the private `local/sow-coop` repo (commit
`cc6b68f`). Offline tests 41/41, negative control 18/18. The hub branch
`game/shadow-of-war` holds these notes, ahead of the fork. PR
[#143](https://github.com/cranebeaker/coop-anything/pull/143) (branch
`game/shadow-of-war-2`) has everything up to session ag.

**Run it:**
- `local/launch-sow.ps1 -Copy a -TimeoutSec 7200 -AllowOthers`, then the same with
  `-Copy b`. Each copy loads unattended and reaches the world in about 48 s; B spawns the
  stand-in about 20 s later.
- Probe: `python -I local/sow-coop/tools/probe.py local/test-installs/sow-b-profile remote`
  (or `puppet`, `inputs`, `player`, `teleport x y z`, `pad ...`, `puppet follow anim|teleport|none`,
  `puppet speed on|off`, `trace`, `wtrace`).
- Measure: move B's own player off A's path first (`teleport 23787.9 43092.5 12989.8` on B),
  then `run_compare.py 0.71,-0.71 0:0.8 1000:0.85` (straight free-run, flat) or
  `drift.py [-v] <legs>`. Watch it: `view-standin.ps1 -Out <prefix> -Turn` once, then
  `view-sprint.ps1` or `view-steer.ps1`.

**Rules that bit us:**
- Never touch the owner's install or saves; work only in `local/test-installs/`.
- Ask before pushing anything.
- Stop stray background `sh.exe` input loops before restarting a copy.
- A test that runs sender and receiver in one process can hide receive-side bugs.
- The probe pipe takes one client at a time: a script holding it blocks every other.
- Most free-run directions from the test start end in a running jump off a ledge within
  ~800 units; use (0.71, -0.71) after a short walk for flat runs.

**Next steps:**
1. Vaults and running jumps during a free-run (the stand-in falls where A jumps). Find what
   starts them on A (`RunningJump_AutoPicker_*` states) and whether the stand-in's graph can
   be made to play them, or move it along A's arc meanwhile.
2. The intermittent missing put-back after a stop (4 of ~35 walking squares; NOTES session ah).
3. Check the stand-in's feet at free-run speed (captures side on), and whether its graph can
   be put into the free-run itself (what the player's graph reads that the inputs do not carry).
4. Steadier drift correction on long walks: `DeltaYaw` turns the stand-in; position writes
   made drift worse.
5. Make it two-way (B to A), then world state: host authority for enemies and Nemesis
   events, and a real network transport.
6. Show the owner the hub diff and ask before pushing.
