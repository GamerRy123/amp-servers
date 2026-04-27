# AMP Stardew Valley Template (Generic Module)

A custom AMP Generic Module template that wraps the [SMAPIDedicatedServerMod](https://github.com/ObjectManagerManager/SMAPIDedicatedServerMod) approach to give you a Stardew Valley dedicated server you can manage from your AMP panel.

## ⚠️ Read this first — important caveats

This is **experimental** and stitched together from public docs. It will likely need tweaking on first run. Specifically:

1. **Stardew has no official dedicated server.** This template orchestrates the community SMAPI + dedicated-server-mod stack. Updates from SDV, SMAPI, or the mod can break it.
2. **Steam login is required.** The template uses `SteamForceLoginPrompt=True` and `SteamUpdateAnonymousLogin=False` because Stardew Valley cannot be downloaded anonymously via SteamCMD. You will be prompted on first update.
3. **Steam Guard.** If 2FA is enabled, AMP's SteamCMD prompt may struggle. Easiest path: dedicated Steam account with Guard disabled.
4. **Multiplayer is Steam-friend-based.** Players still need to be Steam friends with the server's Steam account, or use the invite code that the SMAPI mod surfaces.
5. **Console regexes are best-effort.** `AppReadyRegex`, `UserJoinRegex`, `UserLeaveRegex`, `ServerInfoRegex` may need adjusting once you see actual log output. Edit `stardewvalley.kvp` and re-fetch the repo if needed.
6. **The SMAPI installer download/extract step is fragile.** SMAPI ships an installer (`install on Linux.sh`) that normally needs interactive input. The template grabs the installer ZIP from GitHub releases but does NOT run the installer — you may need to manually extract `SMAPI X.X.X installer/internal/linux/` contents into the game folder on first run if the auto-update doesn't fully install SMAPI. See "Manual SMAPI install" below.

If any of this turns out to be a dealbreaker, run [JunimoServer](https://github.com/stardew-valley-dedicated-server/server) in Docker instead — it solves all of this in 15 minutes.

---

## How this template works

When AMP creates an instance from this template, it:

1. Logs into Steam and downloads Stardew Valley (App ID `413150`) into `./413150/`
2. Downloads the latest SMAPI release ZIP from GitHub
3. Downloads `SMAPIDedicatedServerMod` from GitHub into `./413150/Mods/`
4. Marks `StardewModdingAPI` executable on Linux
5. Launches `StardewModdingAPI --server --no-terminal` which boots Stardew in headless dedicated mode

The `--server` flag is added by the dedicated server mod, not vanilla SMAPI. Without the mod installed correctly, Stardew will try to start a window and fail on a headless box.

---

## Files

| File | Purpose |
| --- | --- |
| `manifest.json` | Repo metadata — id, prefix, origin URL |
| `stardewvalley.kvp` | The Generic Module config — start cmd, ports, update sources, console regexes |
| `stardewvalleyconfig.json` | Settings UI shown in AMP panel (farm type, max players, etc.) |
| `stardewvalleyports.json` | Port allocation hints (TCP+UDP 24642) |

---

## Installation

### 1. Fork or upload this repo

You need this on your own GitHub account so AMP can fetch it.

```bash
# In this folder, on your machine
git init
git remote add origin https://github.com/buchwald-io/amp-servers.git
git add .
git commit -m "Initial Stardew template"
git push -u origin main
```

### 2. Edit `manifest.json` for your fork

If you're forking this rather than using buchwald-io's published version, update:
- `id` — generate a new UUID (`uuidgen` on Linux/Mac, or any online generator)
- `origin` — your repo's `.git` URL
- `url` — your repo's web URL
- `prefix` — what shows up before the template name in AMP's instance creation dropdown (e.g. `Custom`, `Buchwald`, `Personal`)

### 3. Add the repo to AMP

In your AMP panel:

1. Go to **Configuration** → **Instance Deployment**
2. Click **Add Configuration Repository**
3. Enter: `buchwald-io/amp-servers:main`
4. Click **Fetch**
5. **Refresh your browser** (this is critical — modules don't appear without a hard refresh)

### 4. Create the instance

1. **Instance Management** → **Create Instance**
2. In the Module dropdown, find `[BIO] Stardew Valley (SMAPI Dedicated Server)` (or whatever prefix you chose)
3. Pick a name, port (default 24642), and create
4. **Don't start it yet** — go to **Configuration** first to set Steam credentials

### 5. Configure Steam credentials

1. Click into the new instance → **Configuration**
2. Find the Steam settings section (under the SteamCMD update plugin)
3. Enter your Steam username and password
4. Save

### 6. First update

1. Click **Update** on the instance
2. Watch the console — SteamCMD will log in, may prompt for a Steam Guard code
3. Once Stardew is downloaded, the SMAPI and dedicated-server-mod stages run
4. If any stage fails, check console output and fall back to the manual install steps below

### 7. Start the server

1. Click **Start**
2. Watch the console for the "ready" message and any invite code the mod surfaces
3. Connect from your Stardew client (must be Steam friends with the server account, or use the invite code)

---

## Manual SMAPI install (if auto-update doesn't fully install it)

SMAPI's installer ZIP contains the actual game-modified files in a subdirectory; the auto-extract may put them in the wrong place. If `StardewModdingAPI` is missing from `./413150/` after update:

```bash
# As the amp user, in the instance's working directory
cd /home/amp/.ampdata/instances/<InstanceName>/413150/

# After downloading SMAPI installer ZIP and extracting it nearby:
# Copy the Linux internal files into the game folder
cp -r ../SMAPI-*-installer/internal/linux/unix-launcher.sh ./StardewValley
cp -r ../SMAPI-*-installer/internal/linux/SMAPI*.dll ./
cp -r ../SMAPI-*-installer/internal/linux/StardewModdingAPI ./
chmod +x StardewModdingAPI
```

The exact paths depend on the SMAPI release layout — check the SMAPI GitHub for the current installer structure.

---

## Tuning the console regexes

After your first successful run, watch the AMP console output. You'll likely need to tune these in `stardewvalley.kvp`:

- `Console.AppReadyRegex` — flips instance state from "Starting" to "Running". Look for the actual line SMAPI prints when it's ready, e.g. something containing `"started"` or `"now listening"`.
- `Console.UserJoinRegex` — needs `(?<username>...)` capture group. Match whatever SMAPI/the dedicated server mod prints when a player connects.
- `Console.ServerInfoRegex` — capture invite codes or join URLs the mod prints so AMP shows them on the dashboard.

After editing, push to your repo, click **Fetch** in **Instance Deployment** again, then **Update Configuration** on the instance.

---

## Honest assessment

This template gets you AMP-managed lifecycle (start/stop/restart, log viewing, settings UI, monitoring) for a game AMP doesn't natively support. It will not be as polished as a first-party module:

- No RCON — Stardew has no remote console, so the in-game "console" is whatever SMAPI logs to stdout
- No live player list — AMP's player tracker depends on the regex matching, which is best-effort
- Updates are fragile — SDV, SMAPI, and the mod update independently and can desync

If polish matters, run JunimoServer in Docker on the same host. If "everything in AMP" matters more, this is your path.

---

## License

MIT. Use it, fork it, fix it.
