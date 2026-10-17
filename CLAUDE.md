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
- `mods/*.jar` — 4 mods not fetchable via API (CF distribution disabled / unknown fingerprint): Graveyard, CreateColonies, Create Colony Logistics, Undergarden Delight. Update by hand.
- `config/paxi/datapacks`, `config/paxi/resourcepacks` + `*_load_order.json` — loaded by Paxi. Plain files, **packwiz does not update them.** Each folder has a README; `* fixed.zip` and `IAFCE` are our modified versions.
- `.packwizignore` keeps docs out of the pack.

Server only, not in the pack:
- `perfpatch-0.1.0.jar` — our own server mod, only in hosting `mods/` and `~/mc-server-test/mods`. Keep it when syncing mods.
- `config/paxi/datapacks/Modpack Data Fixes.zip` — built by `~/Downloads/novy-modpack/build_data_fixes.py` (`build_fixes.py` there builds the Modpack Fixes resource pack).
- Server configs. `config/servercore/config.yml` is all defaults; deleting it is safe (regenerates). ServerCore silently falls back to defaults for the WHOLE file if one value is invalid (e.g. a mod renamed its MobCategory) — look for `[ServerCore] One of the values in config.yml is not valid` in the log.

## Local test server
`~/mc-server-test` — copy of the hosting server (same mods incl. perfpatch, world copy in `world/`).
- Start in background: `java @user_jvm_args.txt @libraries/net/neoforged/neoforge/21.1.252/unix_args.txt nogui > server-console-X.log 2>&1` (cwd = the server dir), wait for `For help, type`.
- Local RCON 127.0.0.1:25576 (password in its server.properties).
- It often hangs on `stop` after `All dimensions are saved` → kill the java PID (identify by `readlink /proc/PID/cwd`; `pkill -f` matches your own shell).
- Test on a world copy: `cp -r world world-upd`, `level-name=world-upd`, restore afterwards.

## Hosting
- SFTP (play2.minecrafthost.cz:2022) with paramiko: `uv run --offline --no-project --with paramiko python script.py` (paramiko only in uv cache).
- RCON on port 25691 (extra port allocated in the hosting panel).
- The user stops/starts the server in the panel. Only touch `mods/` / configs while it's stopped.

## Update procedure ("updatuj")
1. `packwiz update --all -y`. Check the list: alpha/beta builds (pin to latest release: `packwiz modrinth install --project-id X --version-id Y -y`), major version jumps (read changelog), renamed mods/modids.
2. Check required deps of new versions via Modrinth API. Its metadata lies sometimes (Patchouli for Apotheosis, MezzConfig for JEI are not needed) — the server test decides.
3. Check every file in `config/paxi`: hash lookup on Modrinth, CurseForge via a temp packwiz pack + `packwiz curseforge add <slug>`. Verify the content really is 1.21: `recipe/` not `recipes/`, `tags/item|block` not `items|blocks`, `neoforge:conditions` not `forge:conditions`, recipe files under `data/<ns>/recipe/`. Many "1.21.1" datapacks are really 1.20 layouts. Fix them, make tag entries for mods we don't have `{"id": ..., "required": false}`, name them `* fixed.zip`, update load order + README.
4. Re-run `build_fixes.py` / `build_data_fixes.py` against the new jars (it prints `skip` for fixes no longer needed). Check IAFCE Xaero icon definitions still point to existing Ice and Fire textures.
5. Server test: `packwiz serve --port 8089` from the repo; in `~/mc-server-test` move `mods` aside, `java -jar packwiz-installer-bootstrap.jar -g -s server http://localhost:8089/pack.toml`, copy perfpatch back, copy paxi files. Boot on a world copy, run a minute, `save-all flush`, stop. Diff ERROR/WARN lines against the previous log (strip timestamps). Check `Loaded N recipes`, `/datapack list enabled`, ServerCore.
6. Client: install `-s client` into a scratch dir and diff against the user's UltimMC instance mods; the user launches the game for the final check (`logs/latest.log`, magenta squares = missing texture).
7. Tell the user what changes (new content, risks). User commits, pushes, stops the server.
8. Deploy over SFTP: make hosting `mods/` equal to `~/mc-server-test/mods` (delete extra, upload new/changed, verify sizes), upload changed paxi files/load order/configs. User starts the server; pull `logs/latest.log` and check it.

## History
- Lag spikes = chunk loading (main thread waits in `getChunkBlocking`), worst after teleport/death. view-distance 18 → 12, C2ME added after a worldgen comparison. Next step was a spark profile with `--thread *` to find who requests chunks off-thread.
- Rejected: AI-Improvements (changes look AI, negligible gain), unattended auto-updates, Chunky pregeneration, Paper/Folia hybrids.
- Friends&Foes 5 renamed MobCategories `GLARES/RASCALS` → `FRIENDSANDFOES_*` (broke ServerCore config → deleted it).
