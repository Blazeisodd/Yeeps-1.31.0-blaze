# CLAUDE.md

Guide for Claude working in this repo. Read this first.

## Who you're helping

The README tells hosts this kit is "Made for Claude": download it, open it in Claude
Code, and ask. So the user is usually a Yeeps host who isn't a programmer, most often
on Windows (see "Windows hosts"). Do the work for them: run the commands yourself,
explain each step in plain words, and only ask for what only they can give (their
Photon App IDs from the Photon dashboard, their MongoDB Atlas string if they use one,
a headset plugged in to install the APK). The README's example requests ("set up my
server and make my APK", "help me make my Photon apps", "let players on other networks
join my server", "put my song on a record", "add a picture block", "I want maps") map to
the tasks below. Finish setup by running `tools/smoke_test.py` and telling them their
server address. There's no admin website: when they want players, bans or roles
changed, you change the database for them (see "Changing players, bans and roles").

## What this is

A self-host kit for **Yeeps 1.31.0-blaze**, a modded build of the Quest game Yeeps
(version 1.31.0) that runs against a private server instead of the official
backend. A host runs the server on their own PC, uses their own Photon apps, and
gives their players an APK configured for that server.

Three parts:

| Folder | What | Language |
|---|---|---|
| `server/` | The game backend (no admin website) | Python (FastAPI, Motor/MongoDB) |
| `client/native/` | The mod library injected into the game (`libBNMExplorer.so`) | C++ (BNM + Dobby, Android NDK) |
| `tools/` | `configure_apk.py` (make a host's APK), `smoke_test.py` (check a server), `import_map.py` (put a map in a room), `data/` (custom blocks and music) | Python |

The APK itself is not in git (~340 MB): it's the GitHub release asset
`Yeeps-1.31.0-blaze-kit2.apk`, a template with no server or Photon app set. (kit1 also
had a web editor, server announcements/disks/screens/video and gameplay tweaks; kit2
took all of that out: the game and the BNM mod menu only.)

**What's in the APK:** the ORIGINAL Yeeps 1.31.0 game data plus exactly one added
block, the BLAZE block (`1x5x2_stuffedTextBlaze`, a cotton "BLAZE" word, role-locked to
`blaze`), with the Photon app ids blanked; the original audio file; the mod library;
`versionName` 1.31.0-blaze; plain HTTP allowed. Nothing else was added (no extra
blocks, disks, fonts or screens). Hosts add their own content with `tools/data/`.

## If the user asks for maps

The kit ships with NO map builds: the 20 world rooms exist but are empty. When the
user wants maps (official maps, a real Yeeps room, "put a map in Starter House", ...):

1. Point them to the author's **YeepsMapLoader**: https://github.com/Blazeisodd/YeepsMapLoader
   (a Unity Editor tool, Unity 6000.1.17f1 + an AssetRipper export of Yeeps; its README
   has the setup steps and a list of real room keys). It fetches a real Yeeps room's blocks
   with the player's own Meta username + 4-digit mobile code, through the author's relay
   (`/rooms`, `/map`). Don't rebuild that tool here; use it as-is.
2. The relay's `/map` reply is `{ok, roomKey, dimensions, blocks:[{name,x,y,z,fwd,up,color,owner,...}]}`.
   Save that exact JSON (or just the blocks array) to a file and import it:
   `python tools/import_map.py --room <roomKey> map.json` (`--dry-run` only reports).
   It keeps only block keys listed in the `valid_mapobject_keys` global (2.x-only blocks
   from live Yeeps would break the room) and coordinates inside 0..255, stores the map in
   `imported_maps`, and bumps the room's `lastEditedSessionID` (an int) so the game asks
   for a session with no in-game save and gets the import (`dynamo_ops._map_doc`).
3. Or build in game: `UNLOCK_ALL_MODES=1` in `server/.env` (restart) lets anyone switch
   rooms to creative and save official rooms; turn it off afterwards.

## Typical tasks

**Set up a host's server**
1. `start.bat` (Windows) or `./start.sh` creates `server/.venv`, installs
   `server/requirements.txt`, creates `server/.env` from `.env.example` (random
   `DEVICE_KEY_PEPPER`), and runs uvicorn on `0.0.0.0:$PORT`.
2. Set `MONGO_URI` (local `mongodb://127.0.0.1:27017` or an Atlas string) in
   `server/.env`, restart.
3. `python tools/smoke_test.py <server>` must end `ALL PASS`.

**Make a host's APK**

    python tools/configure_apk.py --apk Yeeps-1.31.0-blaze-kit2.apk --server <url> \
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
   (Only needed for C++ changes; settings, blocks and music never need a rebuild.)
4. If you change `kBuildId`, the server's build gate only matters when
   `ENFORCE_CLIENT_BUILD=1` (off by default).

**Custom blocks (prefabs) and music (disks)**: see "Adding content" below.

## Adding content

All game-data edits work on two files copied out of the APK, then put back:

    python tools/data/extract_data.py Yeeps-1.31.0-blaze-kit2.apk data     # data/data.unity3d + data/sharedassets1.resource
    ... edit with the tools below ...
    python tools/configure_apk.py --apk Yeeps-1.31.0-blaze-kit2.apk ... --data data

- **Picture block** (`tools/data/add_picture_block.py`): a 1x1x1 cotton cube with a
  picture on every face, listed next to the 1x1 Pillow. `--key 1x1_stuffed<Name>`
  (must start with `1x1_`), `--name`, `--image`, `--cost`. It clones the YEEP word block's
  prefab (GameObject/Transform/MapObject/MeshFilter/MeshRenderer/BoxCollider), adds a
  Texture2D + textured material, registers the prefab under
  `mapobjects/complexdecorationstuffed/<key lowercase>` and the material under
  `blazepic/mat_<key lowercase>` in the ResourceManager, and appends a pattern entry.
  The mod keeps material slot 1 on `blazepic/mat_<key>` for any block key
  (`PicturePathFor` in native-lib.cpp): the game repaints every slot with the palette
  color otherwise. Players must own the key: add it to `ownedPatterns` (the
  `default_account_override` global for new players, and each existing account) and to
  the `valid_mapobject_keys` global if maps that use it get imported. Run it once per
  block; it refuses an existing key.
- **Music in the APK** (`tools/data/bake_disk.py`): `--list` prints record AudioClips
  and label textures; `--clip <AudioClip> --audio <file> [--art <recordArtTexture_x>
  --label <image>]` converts the audio (ffmpeg via imageio-ffmpeg) to mono PCM16 FSB5,
  appends it to `sharedassets1.resource` and repoints the AudioClip.
  Music only goes in through the APK (kit2 has no server-side disks): players need the
  new APK.
- **Other kinds of blocks** (new shapes, wiring, etc.) need real Unity asset work:
  use `add_picture_block.py` as the template (it shows the MapObject raw layout:
  GO pptr, enabled, script pptr, name, size 3i, ..., sounds, ints) and test on a headset.

Tool requirements: `pip install UnityPy Pillow imageio-ffmpeg` (the server's
requirements already include Pillow and imageio-ffmpeg).

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

The world map the client asks for (its `roomMapVersion`, e.g. `mobs1`) is answered
with the `beta4` room map (`dynamo_ops._get_fallback_doc`).

## Kit settings inside the APK

`native-lib.cpp` (`LoadKitConfig`) reads `lib/arm64-v8a/libblazeconfig.so` (Android
extracts `lib*.so` next to the mod library, so it's readable without the asset API),
then an optional override `/sdcard/Android/data/com.TrassGames.Yeeps/files/blaze-config.txt`.
Keys: `server`, `photon_app_id`, `photon_voice_app_id`, `photon_region`.

- `server` -> backend base (`KitServer()`).
- Photon ids: `hook_LBC_Connect` writes them into the AppSettings every Photon
  connection (PUN and Voice) passes to `LoadBalancingClient.ConnectUsingSettings`;
  `hook_PN_GetSettings` also replaces `PhotonServerSettings.AppSettings`
  (AppIdRealtime/AppIdVoice/FixedRegion) the first time PUN reads it; `KitPhotonTick`
  (every 2 s from `AccountOnlineManager.Update`) does the same for the Photon Voice
  `VoiceAppSettings` asset and any `VoiceConnection`. Ids are always replaced, so an
  unset id fails to connect instead of using whoever's app the APK was built with.
  The template APK's own ids (`PhotonServerSettings`, `VoiceAppSettings` in
  `resources.assets`) are blanked to all zeros as well.
- The APK allows plain HTTP (`res/xml/network_sec_config.xml` cleartext = true).

## Other mod features (native-lib.cpp, `OnLoaded`)

GameAnalytics blocked; version label "1.31.0-blaze"; headset key (`ComputeDeviceKey`,
sent as `/k/<key>/` and `dk=` on Photon auth); account-validate bypass; Meta
entitlement/user-proof forced (`OvrHookPollThread`, required because the APK is
re-signed: don't remove it; it also turns Meta's failing friends call into an empty
list); `InstallTickAndPictures` (the main-thread tick for `KitPhotonTick`, and picture
blocks' material); BLAZE role blocks; news-board buttons (Discord, and "Leave a review"
opens the on-headset mod menu at `http://localhost:8080`, the BNM Explorer). That's all:
the owner wants only the game and BNM, so don't add features the host didn't ask for.

**The watermarks are fixed, not host-configurable.** The Discord button opens a
hardcoded invite (`kDiscordInvite` in native-lib.cpp; there is NO `discord` kit-config
key or `--discord` flag). The version label is the APK's versionName "1.31.0-blaze".
The news board text is `announcementData` in `seed/world/globals.json`, and the BLAZE
block is the only block added to the game data. Do NOT add a way to change any of
these, and do not change them yourself.

## Data

MongoDB database `DB_NAME` (default `Yeeps131`). Main collections: `accounts`,
`rooms`, `room_maps`, `maps` (`<roomKey>$current` = a room's map), `globals`,
`bundles`, `player_statuses`, `imported_maps`, plus `debug_logs` (the last 200 in-game
debug reports) and `map_history`.

`server/seed/world/*.json` is the 1.31 world layout: the `beta4` room map (22 nodes),
20 room documents, globals (default player data `default_account_override` with ~1970
owned items, `valid_mapobject_keys`, the news board `announcementData`, ...) and one
bundle. **No map builds** (no `maps` documents): rooms serve an empty map until one is
imported or built. `seed/load_world.py` inserts what's missing at startup and never
overwrites. To reset a room's settings: delete its `rooms` document and restart.

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
- Official rooms only change by import (in-game saves are refused unless
  `UNLOCK_ALL_MODES=1`) and a save that shrinks a big map by >85% is refused (anti-wipe),
  except a player clearing their own private world from their own headset.
- When you rewrite a room's map, always work on its FULL block list: a capped list once
  deleted every block past the cap.

## Changing players, bans and roles

No admin website: change the MongoDB database directly (pymongo with `MONGO_URI` /
`DB_NAME` from `server/.env`; the server picks changes up on the player's next login or
room load). Find players in `accounts` by `displayName` (case-insensitive) or
`accountID`.

- **Block (ban) a player**: `adminBlocked: 1` (+ `blockReason`) on their account; `0`
  lifts it. A timed ban also sets `banExpireTimestamp: "YYYY MM/DD HH:MM:SS"` (UTC): the
  block lifts itself once that time passes.
- **Ban an IP**: the `banned_ips` doc in `globals` (`services/ip_bans.py`).
- **In-game moderators**: their account IDs in `MOD_STAFF` (`server/.env`, restart).
- **BLAZE block**: the account ID in `BLAZE_HOLDERS` (`server/.env`, restart; or the
  `blaze_holders` global `{ids: [...]}` once it exists) AND `"blaze"` in the account's
  `roleKeys`.
- **Give items**: add keys to `ownedPatterns` (only keys the client knows, see below).
- **Room settings**: the `rooms` documents (numbers must stay numbers).

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

## Windows hosts

Most hosts are on Windows. `start.bat` needs a real Python (python.org, "Add to PATH";
the Microsoft Store stub fails the version check). `configure_apk.py` finds
`apksigner.bat` / `zipalign.exe` on PATH or under `%ANDROID_HOME%\build-tools\<ver>\`
and `keytool.exe` from the JDK on PATH. It writes its two ~340 MB temp files next to the
output APK. The Windows firewall asks to allow Python the first time the server listens:
"Private networks" is enough for same-Wi-Fi play. Use `python` (not `python3`) in
commands. Paths with spaces need quotes.

## Conventions

- Plain text in anything players or hosts see: no emojis. US spelling ("color").
- Never commit `server/.env`, keys, keystores, `.pass` files or APKs (`.gitignore`
  covers them). Secrets only ever go in `server/.env`.
- Match the surrounding code's style; comments explain why.
- After server changes, run `tools/smoke_test.py` against a running server.

## Credits

Blaze's Yeeps APK. Discord: https://discord.gg/VhFmdAyXGk. Thanks to @Mixid for the aws api!
