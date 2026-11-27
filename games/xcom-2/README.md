# XCOM 2: War of the Chosen (2017)

| | |
|---|---|
| Engine | Unreal Engine 3 (UnrealScript, official mod SDK) |
| Versions tested | Steam, XCOM 2 buildid 8270065 (changelist 699444), War of the Chosen + all DLC, Windows x64 |
| Projects | Team Dragonpunk "Tactical Co-Op" (2016-17, abandoned); WOTC Co-op Campaign: strategy prototype https://github.com/EvilMask587/xcom2-wotc-coop-campaign, extended to missions, up to 4 players and three co-op modes in a fork: https://github.com/Santonus-not-bot/xcom2-wotc-coop-campaign |
| QA level | automated (two to four game copies on one PC, direct IP; three scenarios also on two PCs; see QA) |
| Last updated | 2026-11-25 |

## Where things stand

A full campaign loop runs in automated tests, and the main menu now has three co-op modes. Up to four players
share one Avenger: the host makes the campaign decisions, and each guest manages the soldiers the host gives them.
In missions each player moves their own soldiers in turn (host, then each guest, then the aliens) while the others
watch. Saving and loading mid-mission, Ironman, disconnects and rejoins all work.

- **Co-op Campaign:** the normal campaign, shared.
- **Balanced Co-op Campaign:** the squad grows with the number of players (4 per player at the start, more with the
  squad size upgrades), and every mission gets more and bigger enemy pods plus one enemy from a later stage of the
  game. New campaigns only.
- **Scenario Mode (first version):** one co-op mission without a campaign: 12 operators split among 2-4 players, with
  ranks and gear set by the chosen stage (early, mid, late).

Each mode opens a **lobby** where the host sees who is connected and decides who is player 2, 3 and 4 before
starting. The code is public in the fork above (build and install instructions in its README). Tested on one PC
with up to four game copies; three scenarios also passed on two PCs over direct IP. Steam invites and a
real Steam Workshop subscription for a guest are untested.

## Existing work

- **Team Dragonpunk, "XCOM 2: Tactical Co-Op"** (Steam Workshop 792693587, now removed):
  created 3 Nov 2016, last updated 3 Sep 2017, status "Final". Two players split the squad
  on campaign missions; soldiers can be imported from the friend's game; no Ironman. Only
  the tactical layer, no shared strategy. Probably pre-WotC (last update 5 days after WotC).
  Source: https://github.com/quixotic-cloud/Tactical-Co-Op (GPL-3.0 per the repo, last push
  2017-03-30, vanilla XCOM 2; its `Src/XComGame` folder is the SDK source copied by the old
  ModBuddy template, not the mod's own code), and a Long War 2 variant
  (https://github.com/quixotic-cloud/Long-War-2-Coop, one commit, no license). It uses the
  game's `XComGameStateNetworkManager` over Steam lobbies.
- **robojumper, "WOTC Multiplayer mods"** (Workshop 1571537500): lets mods run in the
  built-in PvP multiplayer; soft-deprecated in favour of the Community Highlander. Not co-op.
- Built-in multiplayer: tactical PvP only.

## How the game works (for co-op)

- Everything that matters is a **game state history** (`XComGameStateHistory`): a list of
  frames, each a set of changed state objects with stable ObjectIDs. Strategy and tactical
  both use it. The mod SDK ships the full UnrealScript source (`Development/SrcOrig`).
- `XComGameStateNetworkManager` (native, script-callable) already moves histories:
  `SendHistory`, `SendPartialHistory`, `SendMergeGameState`, `SendRemoteCommand` with
  typed parameters, receive delegates. Built-in PvP uses "the active player is the
  authority": it runs the ruleset and sends a partial history at the end of its turn.
- Transport: `IpNetDriverSteamworks`, plain IP or Steam sockets. In IP mode the client
  host must be ONE decimal integer (127.0.0.1 -> "2130706433"); a dotted host sends
  nothing. Client and server on one PC need different ports.
- Delegates on the network manager must point at plain Objects: it outlives maps, and a
  delegate bound to an Actor keeps the old World alive (assert on the next map load).
- Every frame submitted on one side is sent to the partner with `SendMergeGameState`.
  A merge state is applied into the receiver's current frame (it does not add a frame), so
  frame counts differ between players; compare content, not indices. Frames without
  objects are refused by SendMergeGameState. Remote commands sent after a merge state are
  handled before it; several frames from one tick can arrive as one merge.
- Merge states carry new and removed objects with their ObjectIDs, and the receiver's ID
  counter moves past them. If BOTH sides create objects, they hand out the same ObjectID
  and the merge silently replaces one object with the other (tested), hence one editor at a
  time.
- **Missions.** WOTC orders turns by initiative groups (`XComGameState_AIGroup.InitiativePriority`),
  and the PvP ruleset (`X2TacticalMPGameRuleset`) already hands control between groups with a
  partial history and a `SwitchTurn` command while the other machine watches a replay. One XCom
  group per player reuses that. Guests connect only to the host, so the host relays a guest's
  frames to the other guests.
- With a connection open, the game treats a campaign mission as multiplayer in several places:
  single-player rules (flank crits, suppression, Shadowbind, pausing) have to be put back, the
  mission load movie never ends by itself, and a dropped connection runs the PvP "opponent left"
  handling, whose asynchronous online-service ping crashes the guest if it leaves the map first.
- Running two instances on one PC: the exe holds a mutex `UnrealEngine3_<n>`; closing that
  handle inside the first instance lets the second start. `-nohomedir` keeps config, logs
  and saves inside the install folder.

## What we changed

Mod: https://github.com/EvilMask587/xcom2-wotc-coop-campaign (code only, no game files) has the
strategy prototype. The campaign and mission co-op below is public in the fork
https://github.com/Santonus-not-bot/xcom2-wotc-coop-campaign; the modes, the menu and the lobby (2026-11-24/25) are
not pushed yet. Design, results per build and every decision: [DESIGN-campaign-coop.md](DESIGN-campaign-coop.md);
every test run: [NOTES.md](NOTES.md).

**Strategy prototype (2026-10):**

- **Client join without post-mission side effects.** A plain `open Avenger_Root` treats a
  received history as a return from a mission and adds 7 frames and new ObjectIDs.
  `CoopHeadquartersGame` (ModClassOverrides) takes the load-game path instead.
- **False desyncs.** The fingerprint is compared when the merge state arrives, not when the
  command does (the command overtook it).
- **ObjectID collisions.** `CoopStrategyRuleset.SubmitGameStates` rejects frames that create
  objects without the baton.
- **Baton handoff race.** The receiver of a grant waits until its fingerprint and highest
  ObjectID match and 1 s passes without frames.
- **Holes in rooms.** The receiver queues a room swap like `XGBase.QueueRoomSwap`.
- **Edits lost on the non-holder.** Every edit screen asks for the baton first.

**Campaign co-op (2026-11):**

- **Up to four players** with slots 0-3; the host assigns a guest's slot on join. A guest
  remembers its slot per host address and asks for it again after restarting its game.
- **Host decides, guests manage their soldiers.** Guests never take the campaign baton; they
  may open the Armory list and their own soldiers' screens, and every other edit screen closes.
- **Soldier ownership.** An "Owner" button on the Armory soldier screen (host only), squad
  select "Slot N" / "Player" buttons for the host, and a colored owner tag next to each
  soldier's flag in missions (HOST gold, P2 green, P3 pink, P4 orange).
- **Mission turns per player.** One XCom initiative group per player, a selection gate so you
  can only select your own soldiers, handover between machines through the host, and the
  single-player rules put back. A banner shows whose turn it is. Watchers can pan the camera.
- **Disconnects.** The host takes over a dropped guest's soldiers; the guest reconnects by
  itself, joins the running mission and gets them back. The host tracks acknowledgements per
  guest slot, so a guest that leaves never blocks the others (found by `guestrestart`).
- **Saves.** Host only. The host saves mid-mission or with Ironman; after loading, it opens
  the server by itself and the guests rejoin.
- **Join and talk.** "Host co-op" on the Avenger HUD, "Join co-op" with an address on the main
  menu, a status line, an in-game mod list check that refuses mismatched guests, tile pings
  (Alt + left click in a mission; a "Ping: player N" marker on the tile for every player) and text chat
  (forwarded to every guest).
- **Crash fixed:** the game's PvP lost-connection handling is skipped in co-op missions (it
  crashed the guest in about one save test in three).
- **Crash fixed:** a guest leaving the Avenger when the host closed the connection died silently in about
  1 departure in 40 (no crash record; Windows' minidumps showed Wwise's event thread calling the game's
  matinee sound handler after the map unload had cleared the world). The guest now pauses the world for a
  second before it leaves.
- **Reconnect after a timeout.** When a guest timed out during a host movie and came back from the
  same game process, the host took it for a duplicate, never sent the campaign, and played on alone. The
  host now counts closed connections and treats a known guest that returns after a close as a
  reconnect with its old slot (found by `soak`).
- **Guest hang on the main menu.** A campaign update that reached a guest at the main menu sent it
  into an endless loop within one frame (a 23 GB log), adding frames that the main menu cannot add.
  History alignment now runs only on the Avenger and stops when a frame isn't added (found by `soak`).
- **Stuns on a guest's soldiers never ended.** Effects that count down on a group's own turn (stun)
  lost their countdown on the machine that plays that group; each machine now registers it again
  for the groups it plays (found by `stunguest`).
- **Joining by network address failed above 127.x.x.x.** The address is passed to the game's network code
  as one number, built in a signed 32-bit int, so every address from 128.0.0.0 up (home networks:
  192.168.x.x) came out negative and failed as a "host name". Only 127.0.0.1 worked, which every one-PC test
  used. Fixed, and a first join now retries for 5 minutes if the host isn't up yet (found by a two-PC
  rehearsal on one PC, guest aimed at the PC's own network address).
- **Enemies that arrive mid-mission never fired overwatch at a guest's soldiers.** A unit the host creates
  during a mission (reinforcements, the Lost, a Chosen arriving) reaches a guest as data only, without the
  event listeners that trigger its abilities, so its overwatch shot never fired when a guest's soldier moved.
  Each machine now registers missing ability listeners again, as loading a save does (found by
  `overwatchguest`; the host-only control `overwatchhost` fired normally).
- **A guest crashed when a mission started right after it rejoined.** If the host launched a mission while a
  guest's campaign download was still unacknowledged, the guest's acknowledgement of the *mission* was taken
  for the campaign's, and the guest opened the Avenger with the mission's data and crashed. A new history now
  replaces any pending one (found by `chosenassault` in the full suite). In the same window a guest could also
  keep a dead connection; a guest that gets its player slot but no history within 150 s now reconnects
  (tested with a host that holds back the campaign: `joinwatchdog`).
- **A guest could control unowned soldiers.** A soldier or VIP joining mid-mission without an owner could be
  selected by any player; it now belongs to the machine that runs its group (the host), and it joins the host's
  turn instead of getting a separate one (`joinxcom`).
- **Mod auto-download, in game.** When the host refuses a guest for different mods, it sends its mod list with
  Workshop ids (written beforehand by its CoopSync); the guest's game saves it, and the guest's CoopSync plans
  the downloads, subscribes with the player's consent, and switches the active mods. Tested end to end with a
  stand-in for Steam: refused, mods fixed, rejoined and playing (`joinsyncapply`); a real Workshop download is untested.
- **A guest never saw its own soldiers act.** On a guest, the game state moved its soldiers but their models
  stayed where they started and none of its actions played (moves, pickups, story moments); the turn just ended.
  A guest loads the mission "in replay" so it can watch, and the game turns visuals for new actions back on only
  for a single-player replay, never for a multiplayer one. The mod now turns them on when a guest takes its turn
  (found by `@pawns`, which compares each soldier's model with its game-state tile; `guestvis`).

**Modes, main menu and lobby (2026-11-24/25):**

- **Main menu.** "Co-op Campaign", "Balanced Co-op Campaign" and "Scenario Mode" sit in the game's own menu row after
  New Campaign (the shell rebuilds its row at times of its own, so the mod checks twice a second). Soldiers from the
  character pool (random ones when it is empty) stand in the menu's 3D scene, placed by a floor trace in front of the
  camera. On backdrops without collision near the camera none are placed (`menushow` records the count).
- **Lobby.** Each entry opens a lobby: host or join, a row per player with its connection state, and buttons for the
  host to make a friend player 2, 3 or 4 (a swap of slots, sent to both guests). Starting a campaign goes through the
  game's own difficulty or load screen with the friends still connected; the host then leaves for the first mission
  and the friends rejoin it in their assigned slots (`lobbycamp`, `lobbyscen3`).
- **Balanced Co-op Campaign.** The squad size and the enemies scale with the number of players. Squad: a hidden sitrep
  per mission adds the normal squad size times (players - 1), so the squad size upgrades count. Enemies: the alert and
  pod-size sitreps never reach the game's enemy picking, so the mod picks the enemies itself
  (`CacheSelectedMissionData` at alert + 1 per extra player, one more unit of the pod's own kind per regular pod per
  extra player, and one follower replaced by an enemy from 4 force levels later). New campaigns only; the choice is
  locked after the first mission (`balanced`, `balancedsquad`). Squads above 6 need robojumper's Squad Select; the
  runner adds it to the test copies for those scenarios.
- **Scenario Mode, first version.** One co-op mission without a campaign, built the way Skirmish mode builds its start
  state: 12 operators of the base classes, ranks and gear by stage (force 3 / 9 / 16, alert 2 / 3 / 4, rank 2 / 4 / 6;
  conventional + Kevlar / magnetic + Predator / beam + Warden), split among the players. The first turn waits until
  every expected player has loaded the map (a guest that joined after the turn began lost it, found by `scenario`).
  At the end the guests go back to the main menu (`scenarioend`). The points shop, saved operators and objective sets
  in the design are not built yet.
- **Host activity in squad select.** The player panel says "Selecting the squad" while the host is in squad select or
  the mission screen before it.
- **Chunked history (no size ceiling).** A whole-history send stops arriving above about 670 KB (see Known issues).
  The engine's `SendRemoteCommand` carries at most one network bunch, but 400-byte chunks with a window of acks arrive
  intact (`chunktest`: a 359 KB campaign history in 898 chunks, byte for byte the history the guest got from a
  normal join). With `bChunkedWholeSend` on, every
  whole send (a join, a mission, the campaign after a mission, the relay of a guest's turn) goes out as the host's
  history base in chunks; each guest takes it as its own base, and the engine's partial history brings it up to date.
  A send that stalls falls back to the engine's whole send after 90 s, which cannot arrive above the ceiling either.
  The long-campaign test with three players (`soakfresh16`) now passes; still off by default until a two-PC test.

## QA

- Two to four copies of the game on one PC (127.0.0.1), each with its own folder. A test bot
  inside the mod (`CoopHarness`, `@` steps) drives each copy; a runner script starts the
  copies, asserts on every copy's log (expected lines, no crash, no desync, equal campaign
  fingerprints) and checks afterwards that the real install and saves are unchanged. The
  runner scripts are not published yet. The copies also take screenshots, checked by eye.
- Last results. "11-27" rows ran on 2026-11-27 on the latest build; "11-25" rows ran on 2026-11-25, on the builds with the main menu, the lobby, the modes and
  chunked sends (the full regression and every content test ran that day; failures were fixed and rerun, see
  NOTES.md); "earlier" rows are from 2026-11-13 to 2026-11-18 and were not rerun. The real install was unchanged in
  every run.

  | Scenario | Players | What it covers | Result | When |
  |---|---|---|---|---|
  | `squad`, `exp12` | 2 | ownership tags, squad slot rules, group split | 13/13, 11/11 | earlier |
  | `join`, `joinmismatch` | 2 | join with matching mods, refusal on a mismatch | 7/7, 5/5 | 11-25 |
  | `join3` | 3 | two guests join one host | 12/12 | 11-27 |
  | `buttons` | 2 | the co-op entries on the main menu, "Join a friend" in the lobby, the host's player panel | 15/15 | 11-25 |
  | `features` | 2 | Owner and slot buttons, guest rights, chat | 22/22 | 11-25 |
  | `guestrestart` | 3 | both guests restart, get their slots back | 19/19 | 11-25 |
  | `mission` | 2 | full rounds with enemies, pings, chat, camera, flags, banner | 54/54 | 11-25 |
  | `dropguest` | 2 | guest killed in its turn, host takes over, guest rejoins | 15/15 | 11-25 |
  | `missionend` | 2 | mission end and return to the Avenger | 11/11 | 11-25 |
  | `mission3`, `mission4` | 3, 4 | guest-to-guest handover through the host | 27/27, 33/33 | 11-27, earlier |
  | `ironman` | 2 | Ironman autosave, load, rejoin | 18/18 | 11-25 |
  | `saveload` | 2 | save, load, second mission | 18/18 | 11-25 |
  | `midsave` | 2 | save inside a mission, load, rejoin | 16/16 | 11-25 |
  | `mindcontrol`, `mcall` | 2 | mind control of and by a guest's soldiers, freeing them | 24/24, 16/16 | 11-25 |
  | `lostchosen` | 2 | the Lost; a Chosen kidnaps a guest's soldier, mission end | 21/21 | 11-25 |
  | `retaliation`, `blacksite` | 2 | Chosen retaliation (resistance, civilians); a story mission | 14/14, 14/14 | 11-25 |
  | `avengerdefense`, `avdefreinf` | 2 | Avenger defense; its reinforcements go to their owner | 16/16, 16/16 | 11-25 |
  | `flankcrit` | 2 | flank crits use the single-player bonus (shot breakdown, host and guest) | 10/10 | 11-25 |
  | `bigsquad` | 2 | 12 soldiers on one mission, split between two players | 13/13 | 11-25 |
  | `stronghold`, `chosenassault` | 2 | a Chosen stronghold; the Chosen attack on the Avenger | 13/13, 13/13 | 11-25 |
  | `finalmission` | 2 | the final mission's map, three rounds | 13/13 | 11-25 |
  | `avatarmc` | 2 | an Avatar mind-controls a guest's soldier; control ends on time | 17/17 | 11-25 |
  | `relayleave` | 3 | a guest leaves while the host relays another guest's turn, rejoins | 16/16 | 11-27 |
  | `soak` | 2 | four missions in a row with the Avenger in between; the guest reconnects after the host's long movie | 13/13 | 11-25 |
  | `soakstory` | 2 | four missions with their objectives completed: the story moves on past the Blacksite | 14/14 | 11-25 |
  | `soakstory8` | 2 | eight missions the same way, a Chosen retaliation among them | 22/22 (twice) | earlier |
  | `guestaction` | 2 | a guest's soldier moves and shoots with the game's normal actions; every unit the same on both machines | 13/13 | 11-25 |
  | `guestaction3` | 3 | the same with three players: one guest's move and shot reach the host and the other guest | 11/11 | 11-25 |
  | `aiturnguest` | 2 | the enemy AI takes its turn against the squad; every unit the same on both machines afterwards | 8/8 | 11-25 |
  | `corruptguest` | 2 | a Chosen Warlock's Corrupt kills a guest's soldier, which rises as an enemy zombie on both machines | 11/11 | 11-25 |
  | `joinxcom` | 2 | a soldier joins XCOM mid-mission: the host controls it in its own turn, the guest cannot select it | 12/12 | 11-25 |
  | `joinwatchdog` | 2 | a guest that gets no campaign after joining reconnects by itself | 9/9 | 11-25 |
  | `joinsync` | 2 | the host's mod list reaches a refused guest; CoopSync on the guest plans the change | 9/9 | 11-25 |
  | `joinsyncapply` | 2 | the whole auto-download path with a stand-in for Steam: refused, CoopSync fetches the host's mod and switches the guest's mods, the guest restarts and joins | 17/17 | 11-25 |
  | `soakstory16` | 2 | sixteen missions with whole histories: the guest misses missions 14 and 15 (history size limit, see Known issues) | 34/38 | earlier |
  | `overwatchhost`, `overwatchguest` | 2 | an enemy on overwatch fires at a soldier walking into view: the host's, then a guest's | 8/8, 8/8 (once 5/8: no shot, cause unknown) | 11-25 |
  | `stunguest` | 2 | a Stun Lancer stuns a guest's soldier; it loses one action and the stun ends | 14/14 | 11-25 |
  | `dazeguest` | 2 | a Chosen Warlock's Mind Scorch dazes a guest's soldier: two turns lost, free on the third | 15/15 | 11-25 |
  | `vialwalk` | 2 | Blacksite: a guest's soldier walks to the vial and takes it with the game's own action; both machines play the story moment; the objective is used on both machines; every soldier's model on its tile | 18/18 | 11-25 |
  | `soakpartial` | 2 | four missions where every history after the first goes as a partial history (experiment, off by default) | 17/17 | 11-25 |
  | `soakpartial16` | 2 | sixteen missions with partial histories: the guest plays all 16, histories up to 757 KB | 41/41 | earlier |
  | `guestvis` | 2 | a guest's soldiers' models follow their moves on the guest; the ping key binding | 17/17 | 11-25 |
  | `crashhunt` | 2 | enemy overwatch before each guest turn, to catch the rare guest crash | 6/6 (14 times, no crash) | earlier |
  | `avengerfeed`, `presenceshow` | 2 | what the other player does, on the Avenger and in a mission (player panel, presence) | 13/13, 11/11 | 11-25 |
  | `loadshow`, `overlaytest` | 2 | the guest's mission loading screen step by step; that it draws on the main menu and the Avenger | 16/16, 4/4 | 11-25 |
  | `pingshow` | 2 | pings drawn on screen, in either player's turn | 11/11 | 11-25 |
  | `squadshow` | 2 | guests watch the host pick the squad | 7/7 | 11-25 |
  | `gatelaunch` | 2 | a mission launch with the mod check on | 7/7 | 11-25 |
  | `controltest` | 2 | control on the Avenger: ask (no, then yes), the host takes it back, a guest edits its own soldier, Barracks owner tags | 17/17 | 11-25 |
  | `menushow` | 1 | the main menu row and the soldiers in the menu scene | 5/5 | 11-25 |
  | `lobbycamp`, `lobbyscen3` | 2, 3 | the lobby: friends join, the host starts a new balanced campaign or a scenario, the friends follow | 12/12, 27/27 | 11-25 |
  | `lobbyleave` | 2 | a friend leaves the lobby and joins again; the host closes the lobby | 9/9 | 11-25 |
  | `lobbyload` | 2 | the lobby's Load: the friends in the lobby get the loaded campaign; the host quits, the friend does not crash | 8/8 | 11-25 |
  | `lobbynewreal` | 2 | a new campaign through the game's own difficulty screen (tutorial off); the friend joins the first mission | 10/10 | 11-25 |
  | `balanced`, `balancedsquad`, `balanced3` | 2, 2, 3 | balanced campaign: squad size per player, scaled enemies; 8 slots with robojumper's squad select; three players | 15/15, 9/9, 16/16 | 11-25 |
  | `scenario`, `scenarioend`, `scenariomid` | 2 | scenario mode: early stage; the end, back to the menu; mid stage (force level, rank, magnetic weapons) | 15/15, 9/9, 16/16 | 11-25 |
  | `chunktest`, `joinchunk`, `missionchunk` | 2 | chunked history sends: the format check; a join; a mission and its end, chunked only | 12/12, 10/10, 14/14 | 11-25 |
  | `soakfresh16` | 3 | sixteen missions with chunked sends; a new guest joins after mission 13 | 47/48 (48/48 with the fixed check, see NOTES.md) | 11-27 |
  | `mission3chunk` | 3 | mission3 with chunked sends: the mission send and every relay of a guest's turn go in chunks | 22/22 | 11-27 |

- Two-PC test kit (not published): the runner can run the host alone and grade a guest log from another PC;
  a script on the guest PC makes its own test copy and runs the guest bot. Rehearsed on one PC over its network
  address: `join` 7/7, `mission` 54/54.
- **Two PCs (2026-11-15):** `join` 7/7, `mission` 54/54 and `missionend` 11/11 over a home network, direct IP, one Steam account (guest PC in
  Steam offline mode).
- Not covered: long sessions on two PCs, Steam sockets and invites, a real Workshop download for a guest
  (the rest of the path is tested with a stand-in for Steam), a full campaign played to the end, sessions longer than sixteen missions.

## How to play

Not playable yet: the campaign code has to be published first. Developers: the prototype's build
and test instructions are in the mod repo README.

## Known issues

- Two PCs tested over direct IP only (join, mission, mission end); untested over Steam (invites and Steam
  sockets).
- A guest who drops mid-turn loses the actions of that turn; the host plays on from there.
- The ping key (Alt + left click) is bound and checked in the test copies, but no real key press has been tested:
  test scripts cannot press keys, so the tests send pings through the console command.
- While the host has any edit screen open (the Geoscape included), a guest waits to edit
  its own soldiers.
- Other mods that replace `UIUnitFlag` conflict with the owner tags, and mods that replace
  `XComHeadquartersController` or `XComShellController` with the guest's handling of a lost host (on the
  Avenger, on the main menu). robojumper's squad
  select screen has not been tested with the slot buttons.
- Test copies show the game's own debug "red screens", still with Steam's `-noRedScreens -review` added; every
  message is from the base game (maps, DLC sounds), none from the mod. Whether a plain Steam launch shows them: unknown.
- When a visualization on a guest times out while it waits for its turn, the mod finishes it so the turn
  can start; the root cause (an AI reveal timing out on the guest) is unknown.
- **Long campaigns: with whole-history sends, a guest can no longer join after the history passes about 670 KB.**
  Whole-history sends up to 661 KB arrive; from 685 KB they never do (16-mission test: around mission 14).
  Compacting the history with the game's `ArchiveHistory` crashed the host. Workaround, tested but off by default:
  `bPartialResend=true` sends a returning guest only the frames it lacks; in a 16-mission test the guest played
  every mission with histories up to 757 KB (`soakpartial16`, 41/41). A second fix, also off by default:
  `CoopChunks.bChunkedWholeSend=true` sends the host's history in 400-byte chunks and then a delta, with no size
  limit. In a 16-mission test (`soakfresh16`, 2026-11-25) every one of 36 sends went that way (bases up to 765 KB
  in about 0.4 s), and a new guest joined after mission 13 and entered the Avenger. That run found a slot bug (the
  new guest took the slot of a guest coming back), fixed; see QA for the rerun. Untested with either fix: more than
  sixteen missions, two PCs.
- Story moments: one tested (a guest takes the Blacksite vial; both machines play the cinematic and agree).
  Other story objectives and a whole campaign are untested.
- A rare guest crash as its turn begins (`XCom2.exe+0xb158a7`, 2 of many runs; not reproduced in 14 runs of the
  scenario built from its setup, nor in a 16-mission soak). From the crash dump and the disassembly: the game's
  visibility update read a unit's cached visibility interface and found it empty. The mod now logs any object whose
  cache is empty after each handoff; it found none in those runs. Root cause unknown. A second address of the same
  kind (`XCom2.exe+0xae6b20`, a lookup of an object by id in the newest history came back empty) crashed a guest 3 s
  into its turn in mission 12 of one 16-mission soak (2026-11-26); the host took over its soldiers and went on.
- A second guest crash, seen once (`XCom2.exe+0x4ce279`, before the guest-visuals fix): a lookup in a history
  frame, the same function as an earlier crash while the history was written. Not seen since.
- Timed effects lost their countdown after a handoff and never ended. They are now registered again
  (`CoopListenerRepair`): by the host for the player's turn (mind control), and by each machine for
  the groups it plays (stun, daze). Tested: mind control, stun, daze. Other group-turn effects use the same
  repair but are untested.
- The host sends nothing while it plays a long movie (98 s after the first mission). Guests time out
  after 60 s and come back by themselves through the main menu (checked in `soak`).
- The lobby's "Load" handles a save made on the Avenger (the friends in the lobby get the campaign); a save made
  inside a mission is not handled.
- A new co-op campaign starts without the tutorial: with the tutorial the game loads the tutorial's own save and
  builds the campaign only after that mission, so it could not become co-op. The mod unticks and locks the box.
- The main menu's 3D scene shows no soldiers on some of the game's menu backdrops (seen, cause not looked into).
- The host crashed once in the game's sound engine (the same Wwise race the guest fix covers) when
  the test bot launched a mission straight from the Avenger; the bot now pauses for a second first.
  Whether a player's normal launch, after the Skyranger flight, can hit it is unknown.

## Next steps

- Publish the campaign code (a fork or a PR to the prototype's repo).
- Two PCs: direct IP, then Steam invites; auto-subscribe a guest to the host's mods.
- Play a whole campaign.
- Decide whether chunked history sends become the default, after a test on two PCs.
- The modes on two PCs: the lobby, the balanced campaign and scenario mode are tested on one PC only.
- Scenario mode: points to spend on abilities, stats, special suits and hero classes, and several objectives to
  split up for, are still to do. Plan and open questions in
  [DESIGN-campaign-coop.md](DESIGN-campaign-coop.md#game-modes-decided-2026-11-13-scenario-mode-replaced-horde-defense-2026-11-16).
- Loading a mid-mission save from the lobby.
