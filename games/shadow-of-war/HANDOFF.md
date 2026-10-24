# Shadow of War co-op: handoff (2026-10-22, after session aj)

Read this first, then the "Status summary 2026-10-21" and sessions ag to aj at the end of
[NOTES.md](NOTES.md).

**Where we are:** phase 5 (build), first co-op step working one way on one PC. Copy A
streams its player over loopback UDP every frame: position, rotation, health, the 25
inputs of the player's animation graph (stick, speed, turning, camera), its root-motion
speed and its buttons (the non-zero entries of its game-action lists). Copy B shows a
spawned Talion stand-in, AI off, moved by A's inputs through its own animation
(`coop_puppet_move=anim`): it walks, turns and stops with A's stride, feet on the ground, and
with A's buttons its graph free-runs when A does (`A_Button_JustPressed` -> `Sprint`, about
80 ms later), keeping within ~5-8 units on flat free-run legs. It is put back only when it
drifts past 150 units or once A stands still (within about 1.2 s of A letting go of the
stick, creeping included). A's full-stick movement is itself a walk in this game; its run is
the A-button free-run. Not carried or not checked yet: vaults and running jumps, attacks,
anything from B to A, and world state (enemies, Nemesis, quests, time). The owner watched
the free-run and confirmed the stand-in sprints (2026-10-23).

**Build and tools:** game build 3095001 only (Steam app 356190). Our guard DLL
(`bink2w64.dll` proxy) and probe live in the private `local/sow-coop` repo (commit
`846ee90`). Offline tests 42/42, negative control 19/19. The hub branch
`game/shadow-of-war` holds these notes. PR
[#143](https://github.com/cranebeaker/coop-anything/pull/143) (branch
`game/shadow-of-war-2`) has everything up to session ag.

**Run it:**
- `local/launch-sow.ps1 -Copy a -TimeoutSec 7200 -AllowOthers`, then the same with
  `-Copy b`. Each copy loads unattended and reaches the world in about 48 s; B spawns the
  stand-in about 20 s later.
- Probe: `python -I local/sow-coop/tools/probe.py local/test-installs/sow-b-profile remote`
  (or `puppet`, `inputs`, `player`, `teleport x y z`, `pad ...`, `puppet follow anim|teleport|none`,
  `puppet actions on|off`, `puppet speed on|off`, `trace`, `wtrace`). `remote` also shows
  what anim mode's put-back check last saw (`off`, `sender_speed`, `standing_ms`, ...).
- See what the graphs do: `graph_actions.py 0.71,-0.71 0@1.0:1.2 1000@1.0:1.0` (A's and the
  stand-in's states while A walks, then free-runs).
- Measure: `run_compare.py 0.71,-0.71 0:0.8 1000:0.85` (straight free-run, flat) or
  `drift.py [-v] <legs>`; both park B's own player off the path first (the stand-in bumps it
  around). `settle_hunt.py` repeats a run until the stand-in ends far from A. Watch it:
  `view-standin.ps1 -Out <prefix> -Turn` once, then `view-sprint.ps1` or `view-steer.ps1`.
- A crash in game code leaves `crash:` lines with the call stack in the copy's guard log.

**Rules that bit us:**
- Never touch the owner's install or saves; work only in `local/test-installs/`.
- Ask before pushing anything.
- Stop stray background `sh.exe` input loops before restarting a copy.
- A test that runs sender and receiver in one process can hide receive-side bugs.
- The probe pipe takes one client at a time: a script holding it blocks every other.
- Most free-run directions from the test start end in a running jump off a ledge within
  ~800 units; use (0.71, -0.71) after a short walk for flat runs.
- Judge animations from the graph's state names (`graph_actions.py`) or by watching, not
  from distant stills (they misled twice).
- Setting player flags on an AI character: keep its action lists full first, or the game's
  bounds check stops it (NOTES session aj).

**Next steps:**
1. Vaults and running jumps with the buttons carried (the stand-in's graph entered
   `RunningJump_Autopicker_Hold` once): whether it jumps where A jumps. `watch_loop.py`
   repeats free-runs for someone watching copy B.
2. Steadier drift correction on long walks: `DeltaYaw` turns the stand-in; position writes
   made drift worse. A stops can still leave it 15-40 units off for ~1.2 s (A glides after
   letting go of the stick; NOTES session ai).
3. Other buttons the lists now carry (attacks, stealth, interactions): check what the
   stand-in's graph does with them, and what the world needs besides the animation.
4. Make it two-way (B to A), then world state: host authority for enemies and Nemesis
   events, and a real network transport.
5. Show the owner the hub diff and ask before pushing.
