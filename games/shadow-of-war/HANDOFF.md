# Shadow of War co-op: handoff (2026-10-21)

Read this first, then the "Status summary 2026-10-21" at the end of [NOTES.md](NOTES.md).

**Where we are:** phase 5 (build), first co-op step working one way on one PC. Copy A
streams its player over loopback UDP every frame. Copy B shows a spawned Talion stand-in
that follows A (50 ms behind), with its AI off, walking with the player's own animations.
Nothing goes from B to A yet; enemies, Nemesis, quests and time are not shared.

**Build and tools:** game build 3095001 only (Steam app 356190). Our guard DLL
(`bink2w64.dll` proxy) and probe live in the private `local/sow-coop` repo (`fb752b4`).
Offline tests 40/40, negative control 17/17; the spawn path is verified live only. The hub
branch `game/shadow-of-war` (`fdb75fd`) is 15 commits ahead of PR #99, not pushed.

**Run it:**
- `local/launch-sow.ps1 -Copy a -TimeoutSec 7200 -AllowOthers`, then the same with
  `-Copy b`. Each copy loads unattended and reaches the world in about 48 s.
- Probe: `python -I local/sow-coop/tools/probe.py local/test-installs/sow-b-profile puppet`
  (or `remote`, `player`, `teleport x y z`, `pad ...`).

**Rules that bit us:**
- Never touch the owner's install or saves; work only in `local/test-installs/`.
- Ask before pushing anything.
- Stop stray background `sh.exe` input loops before restarting a copy: one steered copy A
  into the DLC3 campaign.

**Next steps:**
1. Move the helper scripts left in the agent's temporary scratchpad into
   `local/sow-coop/tools/`.
2. Tune the stand-in's walk/run (foot sliding) and check turning and stopping.
3. Add an offline test for the spawn path.
4. Make it two-way (B to A).
5. Then share world state (host authority, Nemesis events) and add a real network transport.
