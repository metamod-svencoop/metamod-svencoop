# AGENTS.md

This file guides how to perform Agent Coding in this repository using progressive disclosure.

## Basic Memory knowledge base (keep context lean)

1. Use `search_notes` first to check existing notes in `memory/` (do not read all by default).
2. Only when needed, use `read_note` to precisely read a specific note (on-demand loading).
3. If note information is insufficient or outdated, then fall back to targeted repository file reads or ContextEngine/symbol/search-based lookup, and maintain it with `write_note`, `edit_note`, or `delete_note`.

## High-level repository information (prefer corresponding memories first)

The following content was already organized into Basic Memory notes during onboarding, so it is not duplicated here:

- Project purpose/background: `project_overview`
- Tech stack: `tech_stack`
- Directory structure and module breakdown: `project_structure`
- Common development commands: `suggested_commands`
- Code style and conventions: `code_style_conventions`
- Development guidelines and caveats: `development_guidelines`
- Post-task checklist: `task_completion_checklist`
- Module topic: `asext`

## "Source file" entry points when notes are insufficient (query/read on demand)

- The eight component directories are submodules under the `metamod-svencoop` GitHub organization. Commit component changes in their own repository and then update the aggregate gitlink.
- Run `git submodule update --init --recursive` after cloning or pulling.
- Each component builds independently. Public headers are in `include/`, private sources and headers in `src/`; HLSDK is owned by `metamod/include/HLSDK/`.
- Native dependencies use explicit `*_SOURCE_PATH` overrides or pinned FetchContent. The aggregate shares the customized AngelScript SDK; Metamod owns Capstone/procmap submodules, and FallGuys owns Bullet as a nested submodule.
- CMake 3.21+ supports Windows MSVC Win32 and Linux i386. Dependencies build from source; no prebuilt `thirdparty/install` is required.
- Native defaults are `build/x86/<Config>` and `install/x86/<Config>`, with separate Debug/Release trees. Install the complete `addons/` payload under that prefix; explicit prefixes override it. Release archives retain their `build/addons/` layout through temporary staging.
- Shared formatting uses FormatValidation commit `13c9fabe058e1f887ad1b03bb6884de911192c6a` and clang-format 23.1.3. Use `FORMAT_VALIDATION_ONLY=ON` with `format-check`/`format`; formatting itself does not require Python. HLSDK and vendor sources are excluded.
- asext/fallguys/slowdl/gsphysfix resolve private symbols only through GoldSrc_VibeSignatures gamedata hosted by metamod (`pfnResolveGameSymbol`, interface 5:17, lookup by module CRC-64 + `symbolName`). Native builds run `metamod/scripts/sync-gamedata.py` (Python 3.8+, network on first build) per `<plugin>/scripts/manifests/<plugin>.json`; `-DMMFG_SYNC_GAMEDATA=OFF` skips it. BLOB engine identities use the original engine file and verified mapped image ranges. See basic-memory `gamedata`.

- Project docs/plugin docs: root `README.md`, component `README.md` and `USAGE.md`, plus navigation pages in `docs/`.
- Build and configuration entry points: `CMakeLists.txt`, `scripts/` (CMake is the only supported build system; the legacy `Makefile`/`metamod.sln`/`*.vcxproj` were removed in favour of it)
- Main module source code: `metamod/`, `fallguys/`, `asext/`, `ascurl/`, `asqcvar/`, `asusermsg/`, `slowdl/`, `gsphysfix/`
- CI: `.github/workflows/`
- Large directories (avoid full reads): `thirdparty/`, `build/`, `build-cmake/`, `intermediate/`, `output/`, `install/`, `Release/`, `Debug/`, `.vs/`

## Progressive disclosure key points

- Read basic-memory notes or search with context-engine first, then locate single files/symbols; avoid reading the whole repository at once.
- For symbol/binary-related directories, prioritize on-demand targeted lookup and avoid full scans.

## Explore SKILLs

- project-level SKILLs should be explored from `.claude/skills` even when we are using Codex.
