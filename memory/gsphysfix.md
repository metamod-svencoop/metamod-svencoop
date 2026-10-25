---
title: gsphysfix
type: note
permalink: metamod-svencoop/gsphysfix
tags:
- metamod
- gsphysfix
- physics
- gamedata
- hook
---

# gsphysfix — server SLIDEBOX studio hitbox collision fix (2026-10-25)

## Trigger signal
Players standing on or touching `SOLID_SLIDEBOX` studio models with `$flags 512` (`FMODEL_TRACE_HITBOX`) jitter. The client is grounded but the server thinks the player is airborne (MetaHookSv#931).

## Root cause
Engine `SV_CopyEdictToPhysent` clears `physent_t::studiomodel` and only restores it in the `SOLID_BBOX` branch. `SOLID_SLIDEBOX` falls into the default (bounding box) branch, while client prediction uses hitboxes. The source fix is HLND2T-DiligentGraphics#108.

## Correct approach
- Standalone component `metamod-svencoop/gsphysfix` (aggregate submodule `gsphysfix/`), modelled on slowdl. Symbols come from GoldSrc_VibeSignatures PR #406, release v20261025a.
- **Windows builds and HL 8684 Linux:** post-hook `SV_CopyEdictToPhysent(pe, e, check)` and set `studiomodel` only when the entity is SLIDEBOX, `modelindex > 0`, and `sv_models[i]` is a studio model with flag 0x200.
- **HL 10210 / SC 8948 / SC 10257 Linux** (conversion inlined): entry-hook `SV_AddLinksToPM_` and call the original. Only the outermost recursion level fixes the visents/physents appended by the whole walk. This is equivalent to the source fix because the walker's `*pe = *ve` never reads `studiomodel`. It avoids the full rewrite proposed in issue #405, which depended on truepositions, sv_unlag, client_t and more.
- **Hook selection:** the walker path is used only when both `SV_AddLinksToPM_` and `pmove` resolve; upstream publishes `pmove` only for the inlined builds. Otherwise the CE path is used. HL 10210 Linux keeps a standalone CE that nothing calls; manifest conditionalGroups prune it.
- **Sven Linux PIC:** the walker's first 9 bytes contain `call __x86.get_pc_thunk.bx`. Metamod CDetour `check_thunks` rewrites it as `mov ebx, original_pc`, so a trampoline call to the original is safe.
- **playermove layout:** metamod gamedata does not read the `scalar` kind. Use HLSDK `playermove_10152_t` when Sven and `build_number() >= 10152`, otherwise `playermove_t`. HL 10210 has a build number > 10152 but keeps 600 entries, so the Sven check is required. Compile-time static_asserts pin the offsets.
- **Manifests are split by platform** (`gsphysfix_windows` / `gsphysfix_linux`) because conditionalGroups and symbolExemptions apply per gameVersion, not per platform.

## Pitfalls
- HLSDK pulls in windows.h, whose `min`/`max` macros break `std::min`/`std::max`.
- `physent_t` has padding after `blending[2]`. Tests that memcmp structs must create them with memset and copy them with memcpy.
- The `+localinfo mm_pluginsfile` value is truncated at `-`, so the test plugins file must have a hyphen-free name (e.g. `gsphysfixtestplugins.ini`).
- Separating several console commands with `;` in halflife MCP `run_command` does not work reliably; send them one at a time.

## Verification
- Win32 and Linux i386 Debug/Release builds and behaviour tests pass. Both manifests pass online sync and coverage validation. All 8 components pass the aggregate format check.
- **Live, Windows Sven Co-op 10257:** test map is a copy of `test123` with its vehicles changed to `SOLID_SLIDEBOX` (`test123sb`). With `sv_gsphysfix 1` the player stands on the buggy hitbox (z ≈ 93, FL_ONGROUND). With 0 the player is airborne at the same height and gets pushed off.
- Linux live testing and the walker path have not been verified in a game.

## Scope
Engine-side physent hooks; reusing a post-hook instead of a full rewrite on GoldSrc engines where the function is inlined.

## Relations
- relates_to [[gamedata]]
- relates_to [[project_structure]]
