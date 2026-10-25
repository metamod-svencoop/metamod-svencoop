# metamod-svencoop

This is a modified version of metamod along with a group of metamod plugins mainly to provide server-side feature expansions for Sven Co-op.

This metamod is based on [Bots-United's metamod-p](https://github.com/Bots-United/metamod-p)

* Any third-party plugins such as [amxmodx](https://github.com/alliedmodders/amxmodx) are still compatible with this modified version of metamod.

* You are welcome to request for any new hooks or server-side features which is not implemented by Sven Co-op team yet.

* Compatibility with other games (e.g. Counter-Strike) is not guaranteed as it's designed only for Sven Co-op.

## The differences between this metamod and Bots-United's one ?

1. A fallback solution was added to mitigate a problem that `GiveFnptrsToDll` could not be found in plugin dll (This used to happen when plugin was compiled by a newer version of Visual Studio like VS2022). see also https://github.com/Bots-United/metamod-p/issues/24

2. A couple of new APIs were added to mutil API set.

## Related project

[Fall Guys in Sven Co-op](https://github.com/hzqst/sven-fallguys)

[Physics vehicle demo in Sven Co-op](https://github.com/hzqst/sven-vehicle)

[AngelScript plugin template (astemplate)](https://github.com/metamod-svencoop/astemplate)

# Plugins

All plugins below are loaded by this metamod. They are installed into
`Sven Co-op/svencoop/addons/metamod/dlls/` and enabled in `plugins.ini`.

### fallguys.dll (fallguys.so)

Server-side feature expansions for the map `Fall Guys in Sven Co-op`, exposed to
AngelScript through `g_EngineFuncs` / `g_EntityFuncs` / `g_Hooks`.

* This is required by map `Fall Guys in Sven Co-op`

Features :

* GroupMask / GroupOperation manipulation (`g_EngineFuncs.SetGroupTrace`)
* Set brush entity as **Super Pusher**, pushing players and monsters backward on impact
* Custom footsteps, replace the sound played for any movement type per map
* Server-side Level of Detail : an entity's `modelindex` / `scale` / `body` is switched per player by distance
* Semi-Visible : an entity is visible only to the specified player(s)
* SemiClip and PlayerMove-only SemiClip : disable collision / phys-interaction, or only player movement, per player
* SemiRenderEffects : override `rendermode` / `renderamt` / `rendercolor` / `renderfx` for the specified player(s)
* Create physic objects, simulation runs in Bullet engine instead of GoldSrc hull clipping
* Entity follow, similar to `trigger_setorigin` but without extra entity and the latency it brings
* Query the player currently running player-move code, the player's view entity, and information from a sound file
* Hooks : `AddToFullPack`, `PlayerPostThink_Post`, `PlayerTouchTrigger`, `PlayerTouchPlayer`, `PlayerTouchImpact`

Use macro `METAMOD_PLUGIN_FALLGUYS` to detect its availability.

[Documentation](https://github.com/metamod-svencoop/fallguys)

### asext.dll (asext.so)

This plugin provides ability of registering third-party hooks or methods in Sven Co-op's AngelScript engine.

* This is required by map `Fall Guys in Sven Co-op`

It locates the private AngelScript implementation in `server.dll` / `server.so` through
the gamedata catalog hosted by metamod (see [Game data](#game-data)), then exports the
registration and hooking capability through `asext_api.h`.

Features :

* Register object methods (`ASEXT_RegisterObjectMethod`) and global functions (`ASEXT_RegisterGlobalFunction`) into the AngelScript engine
* Register AngelScript hooks (`ASEXT_RegisterHook`) and call them from C++ (`ASEXT_CallHook`)
* Set the default namespace (`ASEXT_SetDefaultNamespace`) while registering symbols
* Define words to the AngelScript script builder (`ASEXT_CScriptBuilder_DefineWord`), which is how the `METAMOD_PLUGIN_*` macros are exposed
* Iterate an AngelScript `dictionary` (`ASEXT_CScriptDictionary_*`) and fetch a type info by name
* Registrations must happen before AngelScript initialization ; `Meta_Attach` is early enough

```cpp
IMPORT_ASEXT_API_DEFINE();
LOAD_PLUGIN(PLID, "addons/metamod/dlls/asext.dll", PLUG_LOADTIME::PT_ANYTIME, &asextHandle);
IMPORT_ASEXT_API(asext);
```

[Documentation](https://github.com/metamod-svencoop/asext)

### ascurl.dll (ascurl.so)

This plugin provides ability of using libcurl to send HTTP request in angelscript. mainly for server ops and developers to use in their own angelscript plugin.

* This is not required if you just gonna play `Fall Guys in Sven Co-op`

Features :

* Create / send / destroy an HTTP request (`CreateHTTPRequest`, `SendHTTPRequest`, `DestroyHTTPRequest`) with `GET` / `POST` / `PUT`
* Synchronous and asynchronous mode, with separate connect timeout and transfer timeout
* Set post fields, append request headers, append multipart form string / blob, upload a blob
* Asynchronous callback (`SetHTTPRequestCallback`) and response code / header / body retrieval (`GetHTTPResponse`)
* Hashing and encoding helpers : `hmac_sha1`, `hmac_md5`, `md5`, `base64_encode`

Use macro `METAMOD_PLUGIN_ASCURL` to detect its availability.

[Documentation](https://github.com/metamod-svencoop/ascurl)

### asqcvar.dll (asqcvar.so)

This plugin provides ability of retreiving cvars from client. mainly for server ops and developers to use in their own angelscript plugin.

* This is not required if you just gonna play `Fall Guys in Sven Co-op`

Features :

* Send a querycvar request to a client (network message `svc_sendcvarvalue2`)
* Register an asynchronous callback for the response (`g_EngineFuncs.SetQueryCvar2Callback`)
* Hooks : `Hooks::Player::QueryCvar`, `Hooks::Player::QueryCvar2`

Use macro `METAMOD_PLUGIN_ASQCVAR` to detect its availability.

[Documentation](https://github.com/metamod-svencoop/asqcvar)

### asusermsg.dll (asusermsg.so)

This plugin provides ability of hooking UserMsg. mainly for server ops and developers to use in their own angelscript plugin.

* This is not required if you just gonna play `Fall Guys in Sven Co-op`

Features :

* Register a UserMsg hook (`g_EngineFuncs.RegisterUserMsgHook`) by message id or name
* Inspect the current message in the hook : argument count, argument type and argument value (`GetUserMsgArgCount`, `GetUserMsgArgType`, `GetUserMsgArgInteger`, `GetUserMsgArgString`)
* Block the original message (`BlockCurrentUserMsg`) and resend your own message through `NetworkMessage`
* Enable / disable all UserMsg hooks (`EnableUserMsgHookGlobal`), which also prevents recursive calls

Use macro `METAMOD_PLUGIN_ASUSERMSG` to detect its availability.

[Documentation](https://github.com/metamod-svencoop/asusermsg)

### slowdl.dll (slowdl.so)

Monitors UDP download requests, including FastDL fallback. No ASExt dependency.

* `sv_slowdl_kick` (default `0`) disconnects clients requesting ordinary files.
* `sv_slowdl_console_log` and `sv_slowdl_file_log` (both default `1`) independently log requests to the console and `logs/slowdl_YYYYMMDD.log`.
* Custom decals and Sven Co-op's current-map soundcache are exempt from kicking and remain visible in logs.
* Includes gamedata for all 15 published engine/platform inputs; old Half-Life BLOB builds require the accompanying Metamod update.

[Documentation and supported builds](https://github.com/metamod-svencoop/slowdl)

### gsphysfix.dll (gsphysfix.so)

Fixes server-side player movement against `SOLID_SLIDEBOX` studio models flagged for hitbox traces (`$flags 512`). The engine gave such physents a bounding box while client prediction used hitboxes, so players standing on or touching them were grounded on the client but airborne on the server. No ASExt dependency.

* `sv_gsphysfix` (default `1`) applies the hitbox rule to server physents.
* Hooks `SV_CopyEdictToPhysent`, or `SV_AddLinksToPM_` on Linux builds that inline the conversion; only `physent_t::studiomodel` changes.
* Includes gamedata for all 15 published engine/platform inputs (GoldSrc_VibeSignatures v20261025a or later).

[Documentation and supported builds](https://github.com/metamod-svencoop/gsphysfix)

# Installation

tutorial video: https://www.youtube.com/watch?v=r2NRp5TZmUM

1. Download latest build (-windows or -linux depending on your OS) from [GitHub Release](https://github.com/metamod-svencoop/metamod-svencoop/releases), then unzip it.

2. Copy everything from the previously unarchived `build` directory into `\steamapps\common\Sven Co-op\svencoop` 

* (Warning: neither `svencoop_addon` nor `svencoop_downloads` is supported)*

* The directory hierarchy should be something like this :

```
Sven Co-op/ (or Sven Co-op Dedicated Server/)
|-- svencoop/
|   `-- addons/
|       `-- metamod/
|           |-- dlls/
|           |   |-- asqcvar.dll (asqcvar.so)
|           |   |-- ascurl.dll (ascurl.so)
|           |   |-- asusermsg.dll (asusermsg.so)
|           |   |-- asext.dll (asext.so)
|           |   `-- fallguys.dll (fallguys.so)
|           |-- gamedata/
|           |   |-- asext/
|           |   `-- fallguys/
|           `-- plugins.ini
|-- svencoop_addons/
|-- svencoop_downloads/
`-- svencoop.exe (or svends.exe / svends_run.sh / svencoop.sh)
```

3. If you had installed metamod and metamod plugins from other sources such as [Bots-United's metamod-p](https://github.com/Bots-United/metamod-p) or [jkivilin's metamod-p](https://github.com/jkivilin/metamod-p), you will have to add those metamod plugins back to `plugins.ini` which might have been overwritten in the step (1).

4. You should either 

* Add `-dll addons/metamod/dlls/metamod.dll` (Windows) or `-dll addons/metamod/dlls/metamod.so` (linux) to launch parameter.

or

* Edit `Sven Co-op/svencoop/liblist.gam`, and change:

```
gamedll "dlls/server.dll"
gamedll_linux "dlls/server.so"
```

to
 
```
gamedll "addons/metamod/dlls/metamod.dll"
gamedll_linux "addons/metamod/dlls/metamod.so"
```

to make metamod work.

The edited `liblist.gam` should be something like this ![](/img/1.png)

* `addons` must be installed into `/Sven Co-op/svencoop`, neither `/Sven Co-op/svencoop_addons` nor `/Sven Co-op/svencoop_download`

* All plugins from this repository are not binary-compatible with `metamod-p` from other sources. You should always use metamod from [metamod-svencoop](https://github.com/metamod-svencoop/metamod-svencoop/tree/main/metamod) to load those plugins.

* Other third-party plugins ( e.g [amxmodx](https://github.com/alliedmodders/amxmodx) ) are still binary-compatible with [metamod-svencoop](https://github.com/metamod-svencoop/metamod-svencoop/tree/main/metamod). You don't have to re-compile them. Just put them in the `plugins.ini`.

# Build

The eight components are independent repositories under the
[metamod-svencoop organization](https://github.com/metamod-svencoop), tracked here as
submodules. Public headers are in `include/`, implementation in `src/`. Metamod owns
HLSDK under `metamod/include/HLSDK/` and its private Capstone/procmap submodules.

Requirements: CMake 3.21+, Git, Python 3.8+, and Windows MSVC Win32 or Linux i386 GCC multilib.

```sh
git clone --recursive https://github.com/metamod-svencoop/metamod-svencoop
cd metamod-svencoop
git submodule update --init --recursive
cmake -S . -B build/x86/Release -A Win32
cmake --build build/x86/Release --config Release --parallel 6
cmake --install build/x86/Release --config Release
```

On Linux omit `-A Win32` and use `-DCMAKE_BUILD_TYPE=Release`; install `gcc-multilib`
and `g++-multilib`. For Debug use `build/x86/Debug` and `--config Debug` on Windows,
or `-DCMAKE_BUILD_TYPE=Debug` on Linux. Configure a separate tree for each configuration.
Installation places the complete `addons/` payload under `install/x86/<Config>/`,
including Windows PDBs, the default plugin list and `addons/metamod/gamedata/`. Copy that directory's contents
into the game's `svencoop/` directory. An explicit `CMAKE_INSTALL_PREFIX` or
`cmake --install --prefix` overrides the default; existing trees retain their cached prefix.
Release archives retain their existing `build/addons/` layout for installation.
The existing `scripts/build-*.bat` and `scripts/build-*.sh` names remain compatibility
entry points to this unified build, independent of the caller's working directory.
Dependencies are compiled within the same CMake tree; no `thirdparty/install` is needed.

## Game data

asext and fallguys no longer carry byte signatures. Their private engine and server
symbols come from [GoldSrc_VibeSignatures](https://github.com/hlnd2t/GoldSrc_VibeSignatures)
gamedata, which currently covers Sven Co-op `svencoop-8948` and `svencoop-10257` on
Windows and Linux. At build time each plugin downloads the published snapshots, keeps the
symbols listed in its `scripts/manifests/<plugin>.json` and installs them to
`addons/metamod/gamedata/<plugin>/`. At run time metamod (interface 5:17) loads every
`index.json` under `addons/metamod/gamedata/` and resolves a symbol by the CRC-64 of the
loaded `hw`/`server` module plus its `symbolName`. On a game build that the gamedata does not
cover, asext and fallguys log `Failed to resolve ...` and refuse to load.

SlowDL uses the same resolver with a broader engine manifest. Install both its
`slowdl` and `slowdl_windows` gamedata directories; see its supported-build matrix.
GSPhysFix likewise needs both `gsphysfix_windows` and `gsphysfix_linux`.
For BLOB engines, Metamod verifies the loaded image against the original engine
file before using that file's CRC and the registered section ranges.

The first build needs network access to download the snapshots; later builds reuse the
cache in the build tree. `-DMMFG_SYNC_GAMEDATA=OFF` skips the download and Python entirely.
The synchronizer is covered by `python -B -m unittest discover -s metamod/scripts/tests`.

## Standalone components and dependencies

Each component can be cloned and built directly; see its own README. Plugins accept
`METAMOD_SOURCE_PATH` and `ASEXT_SOURCE_PATH` for local component clones. Additional
source overrides are `ANGELSCRIPT_SOURCE_PATH`, `CAPSTONE_SOURCE_PATH`,
`RAPIDJSON_SOURCE_PATH`, `CHOCOBO1HASH_SOURCE_PATH`, and `PROCMAP_SOURCE_PATH`. FallGuys additionally accepts `BULLET3_SOURCE_PATH`, defaulting
to its nested `thirdparty/bullet3_fork` submodule. Explicit CMake paths override
environment defaults. Empty paths fetch fixed commits, preserving the customized
AngelScript ABI. Invalid explicit paths fail; dependency source trees are read-only.

The aggregate injects its initialized local dependency trees and reuses vendor targets.
`MMFG_BUILD_METAMOD`, `MMFG_BUILD_ASEXT`, `MMFG_BUILD_ASCURL`, `MMFG_BUILD_ASQCVAR`,
`MMFG_BUILD_ASUSERMSG`, `MMFG_BUILD_FALLGUYS`, `MMFG_BUILD_SLOWDL`, and `MMFG_BUILD_GSPHYSFIX` may disable native components.
SDK consumption does not compile dependency plugins; their runtime loading stays unchanged.

## Formatting

```sh
python -m pip install clang-format==23.1.3
cmake -S . -B build-cmake/format -DFORMAT_VALIDATION_ONLY=ON
cmake --build build-cmake/format --target format-check
cmake --build build-cmake/format --target format
```

Formatting works without a native compiler. The aggregate targets cover all eight
components, including disabled native components; `<component>-format-check` and
`<component>-format` select one. Normal native builds do not install or execute formatting.
The shared FormatValidation commit is fixed at `13c9fabe058e1f887ad1b03bb6884de911192c6a`;
`FORMAT_VALIDATION_SOURCE_PATH` can override its local path. Generated `.clang-format`
files are ignored; HLSDK and vendor sources are excluded.

## Updating components

Commit and publish changes in the component repository first, then commit the corresponding
gitlink in this repository. FetchContent dependency revisions are full commit IDs; update
them deliberately when changing a shared interface. The original aggregate history remains
available; component histories retain their relevant commits and contributors.

## Releases

Pushing a `v*` tag runs the `release` workflow: `windows` and `ubuntu` build in parallel and
upload their archives as artifacts, then `notes` generates bilingual release notes with the
Claude CLI (override with the `RELEASE_NOTES_PROVIDER` variable; `codex` is also supported),
and `publish` validates the exact asset set and the tag target before making the release
public. The release is created as a draft and only un-drafted once every check passes.

The `notes` job needs the `release` environment and the `RELEASE_NOTES_API_KEY`,
`RELEASE_NOTES_BASE_URL` (an HTTPS endpoint without credentials, query or fragment) and
`RELEASE_NOTES_MODEL` settings; without them note generation fails closed and no release is
published. The release scripts are covered by `python -B -m unittest discover -s scripts/tests
-p 'test_release*.py'`, which the build workflows run on every change.
