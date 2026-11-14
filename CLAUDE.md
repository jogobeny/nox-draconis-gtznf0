# CLAUDE.md — Nox Draconis modpack

packwiz modpack for a private NeoForge 21.1.252 / MC 1.21.1 server (minecrafthost.cz) played by 4 friends.
Clients use MultiMC/UltimMC with `packwiz-installer-bootstrap` as pre-launch command (see README.md), so every push reaches players on their next launch.

## Rules from the user
- Talk Czech, short and direct. Give one recommendation, not variants.
- **Never commit or push.** Leave changes in the working tree; the user commits.
- **Don't download/install anything without asking** (mods, tools, packages).
- Gameplay must stay the same: no mod or config that changes behaviour without asking. Performance only if invisible.
- No suspicious shell stuff (no `bash -c`, no `cd` into downloaded dirs). Downloads go to their own empty dir; run Python on them with `-I`.
- Hosting credentials: `~/.config/mc-server/sftp.env` (HOST, PORT, USER, PASS, RCON_PORT, RCON_PASS). Scripts read them from the file; never print or copy them.

## Layout
- `mods/*.pw.toml` — Modrinth / CurseForge metafiles, `side = "client" | "server" | "both"`.
- `mods/*.jar` — 4 mods not fetchable via API (CF distribution disabled / unknown fingerprint). Update by hand. Where to check for new versions:
  - Graveyard: GitHub releases of `MapleSugar365/the-graveyard-1.21-fix` (fork of Finallion's; not the CF "Unofficial Port").
  - CreateColonies (CF 1332665), Create Colony Logistics (CF 1466643), Undergarden Delight (CF 960662): in a temp packwiz pack, `echo 1 | packwiz curseforge add -y "<name>"` and compare the filename.
- `config/paxi/datapacks`, `config/paxi/resourcepacks` + `*_load_order.json` — loaded by Paxi. Plain files, **packwiz does not update them.** Each folder has a README; `* fixed.zip` and `IAFCE` are our modified versions. Upstream sources (our files are modified, so hash lookup won't match):
  - Cataclysm Compat, Quark Compat, Supplementaries Compat, FD Cutting Compat: Modrinth user `CyberRat2` (`/v2/user/CyberRat2/projects`).
  - IAFCE - XaeroMap Support: CF project 1443034. Icon Xaero's, Visual Titles: Modrinth hash lookup works (unmodified).
- `.packwizignore` keeps docs out of the pack.

Server only, not in the pack:
- `perfpatch-0.1.0.jar` — our own server mod, only in hosting `mods/` and `~/mc-server-test/mods`. Keep it when syncing mods.
- `config/paxi/datapacks/Modpack Data Fixes.zip` — our data fixes (namespaces aethersdelight, dungeons_arise(_seven_seas), farmersdelight, idas). The build scripts (`build_data_fixes.py`, `build_fixes.py` for the Modpack Fixes resource pack, formerly in `~/Downloads/novy-modpack`) **no longer exist** (gone by 2026-11-06). The zips are the source of truth now: after an update, compare each patched file against the new jar of a mod that changed and edit the zip by hand.
- `config/Icl/ICL.json` — ItemClearLag, `"Delay": 300` (default 80 s regenerates if the file is missing).
- Server configs. `config/servercore/config.yml` is all defaults; deleting it is safe (regenerates). ServerCore silently falls back to defaults for the WHOLE file if one value is invalid (e.g. a mod renamed its MobCategory) — look for `[ServerCore] One of the values in config.yml is not valid` in the log.

## Local test server
`~/mc-server-test` — copy of the hosting server (same mods incl. perfpatch, world copy in `world/`).
- Start in background: `java @user_jvm_args.txt @libraries/net/neoforged/neoforge/21.1.252/unix_args.txt nogui > server-console-X.log 2>&1` (cwd = the server dir), wait for `For help, type`.
- Local RCON 127.0.0.1:25576 (password in its server.properties): `python3 -I tools/rcon.py server.properties 25576 "save-all flush" stop`.
- It often hangs on `stop` after `All dimensions are saved` → kill the java PID (identify by `readlink /proc/PID/cwd`; `pkill -f` matches your own shell). Exit code 143 of the background job is then expected.
- Startup time is bimodal and depends on machine load, not mods: datapack load ~6 s / server ~29 s, or ~12 s / ~53 s. Don't blame an update for it.
- After a mod update, NeoForge backs up corrected configs as `config/<name>-1.toml.bak`; diff against it to see which new options a mod added (and their defaults).
- Test on a world copy: `cp -r world world-upd`, `level-name=world-upd`, restore afterwards.

## Hosting
- SFTP (play2.minecrafthost.cz:2022). paramiko is **no longer in the uv cache**; use system OpenSSH via `~/mc-server-test/tools/hsftp.sh <batchfile>` (sftp `-b` batch of `ls -l`/`get`/`put "local" "remote"`/`rm`). It feeds the password through `SSH_ASKPASS` (`tools/askpass.sh` reads `sftp.env`), never printed. `-o BatchMode=no` is required, otherwise `-b` disables password auth. Don't run it with `-v` (prints the username).
- RCON on port 25691 (extra port allocated in the hosting panel). Times out when the server is stopped.
- The user stops/starts the server in the panel. Only touch `mods/` / configs while it's stopped. To verify it's stopped: `get logs/latest.log` → `All dimensions are saved` and no new lines (the hosting also hangs on stop with "Stuck Thread" lines; the panel kills it).

## Update procedure ("updatuj")
1. `packwiz update --all -y` (to only preview: run it on a copy of the repo in the scratchpad). Check the list: alpha/beta builds (pin to latest release: `packwiz modrinth install --project-id X --version-id Y -y`), major version jumps (read changelog), renamed mods/modids.
   - Known channels: JEI on Modrinth is mostly beta — stay on the newest *release*. Pinning it re-adds MezzConfig → `packwiz remove mezzconfig`. C2ME is alpha-only, MineColonies/Structurize/Domum snapshots are beta-only, Cataclysm: Spellbooks is beta — those are fine.
   - Changelogs: Modrinth `/v2/project/<id>/version?loaders=["neoforge"]&game_versions=["1.21.1"]`; CurseForge via the website API (no key needed): `https://www.curseforge.com/api/v1/mods/<pid>/files?pageIndex=0&pageSize=50&sort=dateCreated&sortDescending=true` and `.../files/<fileId>/change-log`.
   - packwiz's CF API lags behind fresh uploads by hours. Cross-check CF mods with the website API; if newer, `packwiz curseforge add -y --addon-id <pid> --file-id <fid>`.
   - Also check NeoForge: latest in `https://maven.neoforged.net/releases/net/neoforged/neoforge/maven-metadata.xml`, changelog in `.../<ver>/neoforge-<ver>-changelog.txt`. Updating it needs the installer for `~/mc-server-test` (ask before downloading), likely resets the hosting G1 flags (see History), and the UltimMC instances' NeoForge version. 2026-11-06: 21.1.265 available (network packet-limit security fixes), postponed by the user.
2. Check required deps of new versions via Modrinth API. Its metadata lies sometimes (Patchouli for Apotheosis, MezzConfig for JEI are not needed) — the server test decides.
3. Check every file in `config/paxi`: hash lookup on Modrinth, CurseForge via a temp packwiz pack + `packwiz curseforge add <slug>`. Verify the content really is 1.21: `recipe/` not `recipes/`, `tags/item|block` not `items|blocks`, `neoforge:conditions` not `forge:conditions`, recipe files under `data/<ns>/recipe/`. Many "1.21.1" datapacks are really 1.20 layouts. Fix them, make tag entries for mods we don't have `{"id": ..., "required": false}`, name them `* fixed.zip`, update load order + README.
4. Fix packs (scripts are gone, see Layout): for every updated mod whose namespace is patched in `Modpack Fixes.zip` / `Modpack Data Fixes.zip`, compare the patched file with the new jar — fix still needed, or drop it. Check IAFCE Xaero icon definitions still point to existing Ice and Fire textures: the `textures/entity/...` refs must exist in the jar (the old `textures/models/...` keys are the original pack's and are expected to be missing).
5. Server test: `packwiz serve --port 8089` from the repo; in `~/mc-server-test` move `mods` aside, `java -jar packwiz-installer-bootstrap.jar -g -s server http://localhost:8089/pack.toml`, copy perfpatch back, copy paxi files. Boot on a world copy, run a minute, `save-all flush`, stop. Diff ERROR/WARN lines against the previous log (strip timestamps). Check `Loaded N recipes`, `/datapack list enabled`, ServerCore.
6. Client: install `-s client` into a scratch dir and diff against the user's UltimMC instance mods; the user launches the game for the final check (`logs/latest.log`, magenta squares = missing texture).
7. Tell the user what changes (new content, risks). User commits, pushes, stops the server.
8. Deploy over SFTP: make hosting `mods/` equal to `~/mc-server-test/mods` (delete extra, upload new/changed, verify sizes), upload changed paxi files/load order/configs. If anything changed after the server test (e.g. a late CF release), install it into `~/mc-server-test` and re-boot first — the deploy copies that dir. User starts the server only after the push; pull `logs/latest.log` and check it.

## History
- Lag spikes = chunk loading (main thread waits in `getChunkBlocking`), worst after teleport/death. view-distance 18 → 12, C2ME added after a worldgen comparison. Next step was a spark profile with `--thread *` to find who requests chunks off-thread.
- 2026-10-21: TPS 11–14 caused by hosting GC (Shenandoah `compact`, 29 % of server thread in `ShenandoahPacer`). Support switched startup to G1 + Aikar flags. Changing version/software in the hosting client zone reverts it — check spark JVM args after any such change.
- 2026-10-21: ItemClearLag added at the user's request (clears ground items every 5 min, keeps no-despawn/no-pickup items).
- Rejected: AI-Improvements (changes look AI, negligible gain), unattended auto-updates, Chunky pregeneration, Paper/Folia hybrids.
- 2026-11-06: 28 mods updated (Ice and Fire CE 2.2, AzureLib 3.2, Sophisticated, Amendments 2.2 …). The user OK'd Amendments' new default-on options (pets follow teleports, offhand container pickup, fence/wall connections, bubble columns through grates).
- Friends&Foes 5 renamed MobCategories `GLARES/RASCALS` → `FRIENDSANDFOES_*` (broke ServerCore config → deleted it).
