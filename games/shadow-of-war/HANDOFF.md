# Shadow of War co-op: handoff (2026-10-24, after session am)

Read this first, then the "Status summary 2026-10-21" and sessions ag to am at the end of
[NOTES.md](NOTES.md).

**Where we are:** phase 5 (build), co-op movement working both ways on one PC. Each test
copy streams its player over loopback UDP every frame (position, rotation, health, the 25
inputs of the player's animation graph, its root-motion speed and its buttons, the non-zero
entries of its game-action lists) and shows the other player as a spawned Talion stand-in,
AI off, moved by those inputs through its own animation (`coop_puppet_move=anim`): it walks,
turns, stops and free-runs (`Sprint`, about 80 ms later) with the other player's stride. It is
put back when it drifts past 150 units or once the other player stands still. The stand-in is
marked as the local player's follower, so the local player no longer goes into combat stance
beside it (`Hand_Holding` gone). Both ways at once, flat free-run legs keep within ~5-8 units
again (session ak, before the follower mark: 22-56). The owner watched the stand-in sprint
(2026-10-23). Attacks no longer aim at the stand-in or hurt it: two stubs make the game's
attack target check reject it and drop every hit on it (24 strikes beside it, untouched). If
it is ever freed or killed anyway, the guard spawns a new one within 10 s. A's full-stick
movement is itself a walk; its run is the A-button free-run. Not carried or not checked yet:
vaults and running jumps, attacks, and world state (enemies, Nemesis, quests, time).

**Build and tools:** game build 3095001 only (Steam app 356190). Our guard DLL
(`bink2w64.dll` proxy) and probe live in the private `local/sow-coop` repo (commit
`b4f8d6a`). Offline tests 42/42, negative control 19/19. The test copies' ini files are set
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
  `puppet actions on|off`, `puppet follower on|off`, `puppet targetable on|off`,
  `puppet speed on|off`, `trace <addr> <ms> [names|args]`, `wtrace`).
  `remote` also shows what anim mode's put-back check last saw (`off`, `sender_speed`,
  `standing_ms`, ...).
- Tools drive A and watch B's stand-in; `SOW_SENDER=b` swaps the roles.
- See what the graphs do: `graph_actions.py 0.71,-0.71 0@1.0:1.2 1000@1.0:1.0` (the sender's
  and the stand-in's states while the sender walks, then free-runs).
- Measure: `run_compare.py 0.71,-0.71 0:0.8 1000:0.85` (straight free-run, flat) or
  `drift.py [-v] <legs>`; both park the receiver's own player off the path first.
  `settle_hunt.py` repeats a run until the stand-in ends far from the sender. Watch it:
  `view-standin.ps1 -Out <prefix> -Turn` once, then `view-sprint.ps1` or `view-steer.ps1`,
  or `watch_loop.py` while someone watches copy B.
- The game's combat log: `combat_log.py <B x y z>` (A walks with the combat channels on);
  `attack_try.py [presses] [on|off]` (A strikes beside its stand-in, health before and after).
- A crash in game code leaves `crash:` lines with the call stack in the copy's guard log.

**Rules that bit us:**
- Never touch the owner's install or saves; work only in `local/test-installs/`.
- Ask before pushing anything.
- Stop stray background `sh.exe` input loops before restarting a copy.
- A test that runs sender and receiver in one process can hide receive-side bugs.
- The probe pipe takes one client at a time: a script holding it blocks every other.
- Most free-run directions from the test start end in a running jump off a ledge within
  ~800 units; use (0.71, -0.71) after a short walk for flat runs. The start itself is a tower
  ledge: a stand-in knocked there falls.
- Judge animations from the graph's state names (`graph_actions.py`) or by watching, not
  from distant stills (they misled twice).
- Setting player flags on an AI character: keep its action lists full first, or the game's
  bounds check stops it (NOTES session aj).
- Do not teleport a copy's own player thousands of units from its stand-in to test "far": the
  game freed the stand-in once (the guard now replaces it). Re-read the stand-in's pointer
  before any probe write to it.
- `attack_try.py` leaves the stand-in alone now; with `puppet targetable on` it kills it, and
  anything measured right after runs on a new one.
- A static reading of which register holds what can be wrong: check with `trace ... args`
  before writing a stub (the hit's target was in r9, then rdx, never rcx).
- Both copies have twice quit together about 12 minutes after launch (exit 0, no crash);
  relaunch them and carry on.

**Next steps:**
1. Check what enemies do with the stand-in now that no attack can target or hit it (they
   should ignore it); then show the other player's health on it.
2. Vaults and running jumps with the buttons carried (the stand-in's graph entered
   `RunningJump_Autopicker_Hold` once): whether it jumps where the sender jumps.
3. Steadier drift correction on long walks: `DeltaYaw` turns the stand-in; position writes
   made drift worse. A stops can still leave it 15-40 units off for ~1.2 s (NOTES session ai).
4. Other buttons the lists now carry (attacks, stealth, interactions), then world state:
   host authority for enemies and Nemesis events, and a real network transport.
5. Show the owner the hub diff and ask before pushing.
