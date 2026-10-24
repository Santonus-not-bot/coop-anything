# Shadow of War co-op: handoff (2026-10-23, after session ak)

Read this first, then the "Status summary 2026-10-21" and sessions ag to ak at the end of
[NOTES.md](NOTES.md).

**Where we are:** phase 5 (build), co-op movement working both ways on one PC. Each test
copy streams its player over loopback UDP every frame (position, rotation, health, the 25
inputs of the player's animation graph, its root-motion speed and its buttons, the non-zero
entries of its game-action lists) and shows the other player as a spawned Talion stand-in,
AI off, moved by those inputs through its own animation (`coop_puppet_move=anim`): it walks,
turns, stops and free-runs (`Sprint`, about 80 ms later) with the other player's stride. It is
put back when it drifts past 150 units or once the other player stands still. One way, it
kept within ~5-8 units on flat free-run legs (the owner watched it sprint, 2026-10-23).
Both ways, tracking is worse (gaps 22-56) because each player reacts to the other's stand-in
as to a nearby character: the combat targeting counts it as a valid RB target and the
player's graph goes into `Hand_Holding`, which changes its own walk; the other copy cannot
mirror that (NOTES session ak). A's full-stick movement is itself a walk; its run is the
A-button free-run. Not carried or not checked yet: vaults and running jumps, attacks, and
world state (enemies, Nemesis, quests, time).

**Build and tools:** game build 3095001 only (Steam app 356190). Our guard DLL
(`bink2w64.dll` proxy) and probe live in the private `local/sow-coop` repo (commit
`7b0917d`). Offline tests 42/42, negative control 19/19. The test copies' ini files are set
up both ways (A sends to 47011 and listens on 47012; B the reverse). The hub branch
`game/shadow-of-war` holds these notes. PR
[#143](https://github.com/cranebeaker/coop-anything/pull/143) (branch
`game/shadow-of-war-2`) has everything up to session ag.

**Run it:**
- `local/launch-sow.ps1 -Copy a -TimeoutSec 7200 -AllowOthers`, then the same with
  `-Copy b`. Each copy loads unattended and reaches the world in about 48 s; each spawns its
  stand-in of the other about 20 s later.
- Probe: `python -I local/sow-coop/tools/probe.py local/test-installs/sow-<a|b>-profile remote`
  (or `puppet`, `inputs`, `player`, `teleport x y z`, `pad ...`, `puppet follow anim|teleport|none`,
  `puppet actions on|off`, `puppet speed on|off`, `trace`, `wtrace`). `remote` also shows
  what anim mode's put-back check last saw (`off`, `sender_speed`, `standing_ms`, ...).
- Tools drive A and watch B's stand-in; `SOW_SENDER=b` swaps the roles.
- See what the graphs do: `graph_actions.py 0.71,-0.71 0@1.0:1.2 1000@1.0:1.0` (the sender's
  and the stand-in's states while the sender walks, then free-runs).
- Measure: `run_compare.py 0.71,-0.71 0:0.8 1000:0.85` (straight free-run, flat) or
  `drift.py [-v] <legs>`; both park the receiver's own player off the path first.
  `settle_hunt.py` repeats a run until the stand-in ends far from the sender. Watch it:
  `view-standin.ps1 -Out <prefix> -Turn` once, then `view-sprint.ps1` or `view-steer.ps1`,
  or `watch_loop.py` while someone watches copy B.
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
- With both copies sending, a sender within ~300 units of its stand-in of the other player
  moves differently (`Hand_Holding`); measure one direction with the other player parked
  far away, or fix the targeting first.

**Next steps:**
1. Keep the stand-in out of the local player's combat targeting (`TargetValid_ActivateRB`
   from `17B35A8`, `TargetNativePredicate`) and check that `Hand_Holding` goes away near it;
   then re-measure two-way tracking. Also check what the player can still do to the
   stand-in (attack it, RB on it).
2. Vaults and running jumps with the buttons carried (the stand-in's graph entered
   `RunningJump_Autopicker_Hold` once): whether it jumps where the sender jumps.
3. Steadier drift correction on long walks: `DeltaYaw` turns the stand-in; position writes
   made drift worse. A stops can still leave it 15-40 units off for ~1.2 s (NOTES session ai).
4. Other buttons the lists now carry (attacks, stealth, interactions), then world state:
   host authority for enemies and Nemesis events, and a real network transport.
5. Show the owner the hub diff and ask before pushing.
