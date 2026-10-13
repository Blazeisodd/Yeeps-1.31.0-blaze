# CLAUDE.md

Guide for Claude working in this repo. Read this first.

## What this is

A self-host kit for **Yeeps 1.31.0-blaze**, a modded build of the Quest game Yeeps
(version 1.31.0) that runs against a private server instead of the official
backend. A host runs the server on their own PC, uses their own Photon apps, and
gives their players an APK configured for that server.

Three parts:

| Folder | What | Language |
|---|---|---|
| `server/` | The game backend + web editor (`/editor`) | Python (FastAPI, Motor/MongoDB) |
| `client/native/` | The mod library injected into the game (`libBNMExplorer.so`) | C++ (BNM + Dobby, Android NDK) |
| `tools/` | `configure_apk.py` (make a host's APK), `smoke_test.py` (check a server) | Python |

The APK itself is not in git (it's ~380 MB): it's the GitHub release asset
`Yeeps-1.31.0-blaze-kit1.apk`, a template with no server or Photon app set.

## Typical tasks

**Set up a host's server**
1. `start.bat` (Windows) or `./start.sh` creates `server/.venv`, installs
   `server/requirements.txt`, creates `server/.env` from `.env.example` (random
   `EDITOR_PASSWORD` + `EDITOR_SECRET`), and runs uvicorn on `0.0.0.0:$PORT`.
2. Set `MONGO_URI` (local `mongodb://127.0.0.1:27017` or an Atlas string) and
   `PUBLIC_BASE_URL` (the address headsets use) in `server/.env`, restart.
3. `python tools/smoke_test.py <server> --editor-password <pw>` must end `ALL PASS`.

**Make a host's APK**

    python tools/configure_apk.py --apk Yeeps-1.31.0-blaze-kit1.apk --server <url> \
        --photon-app-id <PUN id> --photon-voice-app-id <Voice id> [--photon-region us]

Needs Java (`keytool`) and Android build-tools (`zipalign`, `apksigner`) on PATH or
under `$ANDROID_HOME/build-tools/<ver>/`. It rewrites one zip entry
(`lib/arm64-v8a/libblazeconfig.so`, a text file), keeps every other entry's
compression (`assets/bin/Data/data.unity3d` MUST stay Stored, never deflate it),
zipaligns and signs with `tools/.keys/yeeps-selfhost.keystore` (created on first use;
its password is in the `.pass` file next to it). Both are gitignored: never commit them.

**Change the mod library (C++)**
1. Edit `client/native/native-lib.cpp`.
2. Build with Android NDK r27c (any recent NDK should work), CMake and Ninja:

       cmake -S client/native -B build/native -G Ninja \
         -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK/build/cmake/android.toolchain.cmake \
         -DANDROID_ABI=arm64-v8a -DANDROID_PLATFORM=android-27 -DCMAKE_BUILD_TYPE=Debug
       ninja -C build/native

3. Put it in the APK: `tools/configure_apk.py ... --lib build/native/libBNMExplorer.so`.
4. If you change `kBuildId`, the server's build gate only matters when
   `ENFORCE_CLIENT_BUILD=1` (off by default).

## How the game talks to the server

The game thinks it talks to AWS. The mod (`InstallBackendRedirect`) rewrites every
`*.execute-api.*`, `lambda.*` and `dynamodb.*` amazonaws.com URL to
`<server>/xxxxxxxxxxxxxxxxxx[/k/<headset key>][/c/<build tag>]/<original path>`.
`server/main.py` strips those prefixes, then routes:

- **API Gateway** (direct paths) e.g. `POST .../questLogIntoAccount` -> `handlers/login.py`.
- **Lambda**: `POST /2015-03-31/functions/g2_<name>/invocations` -> `routers/lambda_router.py`
  (name table there) -> `handlers/*.py`.
- **DynamoDB**: `POST /` with `X-Amz-Target: DynamoDB_20120810.<Op>` ->
  `services/dynamo_ops.py` (GetItem/BatchGetItem/UpdateItem/... on tables G2_Accounts,
  G2_Rooms, G2_RoomMap, G2_Maps, G2_Globals, G2_Bundles, G2_PlayerStatuses), backed by
  Mongo collections (`services/db.py` maps tables to collections).
- **Photon custom auth** (only if the host configures it in the Photon dashboard):
  `GET /?accountID=...&dk=<headset key>` -> `main._photon_auth_denied`.
- **Mod polling**: `GET /client/feed.txt` (announcements, disk audio, picture screens)
  and `GET /client/video.txt` -> `routers/client_router.py`.
- **Editor**: `/editor` (page `server/static/editor_ui.html`, API `routers/editor_router.py`).
  Sign-in = any name + `EDITOR_PASSWORD`. Everyone signed in is the host (full access).

The world map the client asks for (its `roomMapVersion`, e.g. `mobs1`) is answered
with the `beta4` room map (`dynamo_ops._get_fallback_doc`).

## Kit settings inside the APK

`native-lib.cpp` (`LoadKitConfig`) reads `lib/arm64-v8a/libblazeconfig.so` (Android
extracts `lib*.so` next to the mod library, so it's readable without the asset API),
then an optional override `/sdcard/Android/data/com.TrassGames.Yeeps/files/blaze-config.txt`.
Keys: `server`, `photon_app_id`, `photon_voice_app_id`, `photon_region`, `discord`.

- `server` -> backend base (`KitServer()`), feed and video URLs.
- Photon ids: `hook_PN_GetSettings` replaces `PhotonServerSettings.AppSettings`
  (AppIdRealtime/AppIdVoice/FixedRegion) the first time PUN reads it; `KitPhotonTick`
  (every 2 s from `AccountOnlineManager.Update`) does the same for the Photon Voice
  `VoiceAppSettings` asset and any `VoiceConnection`. Ids are always replaced, so an
  unset id fails to connect instead of using whoever's app the APK was built with.
  The template APK's own ids (`PhotonServerSettings`, `VoiceAppSettings` in
  `resources.assets`) are blanked to all zeros as well.
- `discord` -> the news board's Discord button (default `https://discord.gg/VhFmdAyXGk`).
- The APK allows plain HTTP (`res/xml/network_sec_config.xml` cleartext = true).

## Other mod features (native-lib.cpp, `OnLoaded`)

GameAnalytics blocked; version label "1.31.0-blaze"; headset key (`ComputeDeviceKey`,
sent as `/k/<key>/` and `dk=` on Photon auth); account-validate bypass; Meta
entitlement/user-proof forced (`OvrHookPollThread`, required because the APK is
re-signed: don't remove it); remote client (announcements, disk audio, screens, video);
item dispenser accepts all items; BLAZE role blocks; first room tutorial -> Starter
House; news-board buttons (Discord, and "Leave a review" opens the on-headset mod menu
at `http://localhost:8080`, the BNM Explorer).

## Data

MongoDB database `DB_NAME` (default `Yeeps131`). Main collections: `accounts`,
`rooms`, `room_maps`, `maps` (`<roomKey>$current` = a room's map), `globals`,
`bundles`, `player_statuses`, plus logs (`editor_logins`, `client_error_*`, `debug_logs`,
`map_history` for reverts).

`server/seed/world/*.json` is the 1.31.0-blaze world (20 rooms on the `beta4` map,
their maps, globals incl. the default player data `default_account_override` and the
news board `announcementData`). `seed/load_world.py` inserts what's missing at startup
and never overwrites. To reset a room: delete its `rooms`/`maps` documents and restart.

## Things that break the 1.31 client (learned the hard way)

- Every attribute the client asks for in GetItem/BatchGetItem must come back with
  the right type. Missing -> it parses null -> `ArgumentNullException ("s")` or
  `NO_DATA_FOUND`. `dynamo_ops._fill_requested` / `_roommap_default` fill defaults.
- Numeric fields must be DynamoDB numbers. A G2_Rooms `lastEditedSessionID` stored as a
  string -> `Int32.Parse(null)` -> "Failed to load game" and a bricked room.
- `spawnRoomKey` must be a normal room (`official_1`); blank is fatal. A community-world
  room as spawn hangs.
- Accounts need non-empty `e_accessKey` / `e_secretKey`, `publicSessionID`,
  `challengeNonce` (login fills them). AWS signatures are never checked.
- `ownedPatterns` must only hold keys the client knows; unknown keys log an error each
  and slow room loading a lot.
- Private worlds (`p_o_<accountID>`, rooms `..._sandbox` 32x80x32 and the main 80x80x80)
  need a doorway node with `transitionKey: "public"` (`NO_TRANSITION_NODE_FOUND`) and
  full room documents (`dynamo_ops._full_room`).
- Maps are SPAM format (`U1BBTQ...`, mapDataVersion 3). `saveInternalStates` =
  `"SPAM"`, u8 3, i32 count, then per entry i32 x,y,z + a .NET string (signs etc.);
  empty = `U1BBTQMAAAAA`. Decoder/encoder: `services/mapsync_decoder.py` / `mapsync_serializer.py`.
- Official rooms are editor-only (in-game saves are refused) and a save that shrinks a
  big map by >85% is refused (anti-wipe), except a player clearing their own private
  world from their own headset.

## Checking a headset

- `adb logcat -s Unity BNMExplorer MYDBG` while the game runs.
- `adb pull /sdcard/Android/data/com.TrassGames.Yeeps/files/bnm_checkpoint.txt`: the mod's
  startup log ("Kit: settings from apk, server=...", "Kit: Photon ids -> ... (ok)").
- The game only runs while the headset is worn. Never send `adb shell input keyevent
  KEYCODE_POWER` (it can trigger a privacy lock that needs the owner).

## Not in this kit (on purpose)

Community worlds (`c_<name>` find nothing; `g2_cwGetCanEnter` says they're off),
popular/featured world lists (empty), live player lookups from the real Yeeps API,
the Discord bot / account linking, and the Photon bot. The server must never contact
the real Yeeps backend.

## Conventions

- Plain text in anything players or hosts see: no emojis. US spelling ("color").
- Never commit `server/.env`, keys, keystores, `.pass` files or APKs (`.gitignore`
  covers them). Secrets only ever go in `server/.env`.
- Match the surrounding code's style; comments explain why.
- After server changes, run `tools/smoke_test.py` against a running server.

## Credits

Blaze's Yeeps APK. Discord: https://discord.gg/VhFmdAyXGk. Thanks to @Mixid for the aws api!
