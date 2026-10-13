# Yeeps 1.31.0-blaze (self-host kit)

Run your own Yeeps 1.31.0-blaze server on your own PC, with your own Photon apps.
Your players install an APK made for your server. Nothing here talks to anyone
else's server.

What you get:

- **The 1.31.0-blaze APK** (in [Releases](../../releases)): GameAnalytics tracking
  blocked, the BLAZE blocks, the version shown as "1.31.0-blaze", and a news-board
  Discord button.
- **A server** that runs in a console window on your PC (`start.bat` / `start.sh`):
  logins, the world, maps, private worlds, moderation, and a web editor at `/editor`.
- **The 1.31 world**: Starter House, Playground, Blue House, the Wilds, the tutorial,
  the Build-A-Map rooms and more, loaded into your database the first time it starts.

Not included: community worlds, live player lookups from real Yeeps, a Discord bot,
or a Photon bot.

Join the Discord: https://discord.gg/VhFmdAyXGk

> **Status (kit1):** the server is tested end to end (`tools/smoke_test.py`). The APK's
> settings file and Photon app swap haven't been tested on a headset yet: please report
> how it goes in the Discord.

Thanks to @Mixid for the aws api!

## What you need

- A PC that stays on while people play (Windows, Linux or macOS).
- [Python 3.11+](https://www.python.org/downloads/) (on Windows tick "Add to PATH").
- A database: either [MongoDB Community Server](https://www.mongodb.com/try/download/community)
  on the same PC, or a free [MongoDB Atlas](https://www.mongodb.com/cloud/atlas) cluster.
- A free [Photon](https://dashboard.photonengine.com) account.
- To make your APK: Java (a JDK) and Android build-tools (`zipalign`, `apksigner`).
- Quest headsets with developer mode, and [SideQuest](https://sidequestvr.com) or `adb`.

## Setup

### 1. Make two Photon apps

At https://dashboard.photonengine.com, create:

- a **PUN** app (Multiplayer Game, Photon PUN), and
- a **Voice** app (Photon Voice).

Copy both App IDs. No custom authentication is needed.

### 2. Start the server

Windows: double-click `start.bat`. Linux/macOS: `./start.sh`.

The first run installs the Python packages and creates `server/.env` with a random
editor password (shown in the window). Then edit `server/.env`:

- `MONGO_URI`: leave the default for a local MongoDB, or paste your Atlas connection string.
- `PUBLIC_BASE_URL`: the address players' headsets will use (see step 3).

Close the window and start it again after editing `.env`. Keep the window open while
people play.

Check it: `python tools/smoke_test.py http://127.0.0.1:8000 --editor-password <password>`
should end with `ALL PASS`.

### 3. Pick your server's address

The headsets need to reach your PC. Pick one:

- **Same Wi-Fi only:** `http://<your PC's local IP>:8000`, for example `http://192.168.1.20:8000`.
  Your PC's firewall must allow incoming connections on that port (Windows asks the
  first time; on Linux with ufw: `sudo ufw allow 8000/tcp`).
- **Over the internet, fixed address (best):** a Cloudflare named tunnel with your own
  domain, for example `https://yeeps.your-domain.com`. See Cloudflare's
  "Create a tunnel" guide and point it at `http://127.0.0.1:8000`.
- **Quick test over the internet:** `start.bat --tunnel` / `./start.sh --tunnel` opens a
  free `https://....trycloudflare.com` address. It changes every time, so only use it
  for testing (see "Override file" below).

Put the same address in `PUBLIC_BASE_URL` in `server/.env`.

### 4. Make your APK

Download `Yeeps-1.31.0-blaze-kit1.apk` from Releases, then:

```
python tools/configure_apk.py --apk Yeeps-1.31.0-blaze-kit1.apk --server http://192.168.1.20:8000 --photon-app-id <PUN app id> --photon-voice-app-id <Voice app id>
```

It writes your settings into the APK and signs it with your own key (created in
`tools/.keys/` the first time; keep that folder, or later APKs won't install over
earlier ones). The output is `Yeeps-1.31.0-blaze-<your server>.apk`.

### 5. Install it on each headset

With SideQuest, or `adb install -r <your apk>`. If another Yeeps 1.31.0-blaze build is
already installed, uninstall it first: a different signing key can't update it.

### 6. Use the editor

Open `<your server>/editor` and sign in with `EDITOR_PASSWORD` from `server/.env`.
From there you can edit players, ban, kick, change the news board, edit maps and rooms,
and more.

## Override file (testing)

A headset can use a different server without a new APK. Put a file named
`blaze-config.txt` at `/sdcard/Android/data/com.TrassGames.Yeeps/files/` with:

```
server=https://whatever.trycloudflare.com
photon_app_id=<PUN app id>
photon_voice_app_id=<Voice app id>
```

`python tools/configure_apk.py ... --settings-only` prints this file for you.

## Using Claude

This repo is set up for [Claude Code](https://claude.com/claude-code): open it and
ask, for example, "set up my server and make my APK". `CLAUDE.md` explains everything
to Claude.

## Disclaimer

A fan-made private server project. Not affiliated with or endorsed by Trass Games or
Yeeps. Don't use it to cheat in, or harm, the real game.
