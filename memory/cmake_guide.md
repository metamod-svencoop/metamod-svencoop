---
title: cmake_guide
type: note
permalink: metamod-fallguys/cmake-guide
---

# CMake Build Guide

Requirements: CMake 3.21+, Git, MSVC Win32 or Linux GCC i386 multilib.

```sh
git submodule update --init --recursive
cmake -S . -B build/x86/Release -A Win32
cmake --build build/x86/Release --config Release --parallel 6
cmake --install build/x86/Release --config Release
```

Linux omits -A Win32 and sets -DCMAKE_BUILD_TYPE=Release; Debug is also supported.
Windows uses /MT (/MTd in Debug), Linux uses -m32 and PIC. Release includes debug symbols.
Installed components retain their DLL/SO names under addons/metamod/dlls; Windows PDBs
are installed alongside them. Metamod's definition file is src/metamod.def.

Each component is a standalone CMake project. Plugins depend on Metamod public headers;
ascurl, asqcvar, asusermsg and fallguys consume ASExt public headers and load ASExt at runtime.
The Metamod::SDK and ASExt::SDK interface targets do not build dependency DLLs/SOs.

Source overrides: METAMOD_SOURCE_PATH, ASEXT_SOURCE_PATH, ANGELSCRIPT_SOURCE_PATH,
CAPSTONE_SOURCE_PATH, PROCMAP_SOURCE_PATH; FallGuys adds BULLET3_SOURCE_PATH. Explicit
CMake variables override environment defaults; missing overrides fetch fixed commits.
Invalid supplied paths fail. Metamod and FallGuys first reuse their initialized nested
submodules (Capstone/procmap, bullet3_fork). The aggregate injects initialized shared
trees. Vendor libraries compile in the native build tree, with shared target guards;
no prebuilt thirdparty/install is required.

Visual Studio filters: each component calls Metamod's `mmfg_assign_source_groups(<target>)`
(metamod/cmake/Build.cmake) after its last `target_sources`, so late platform sources are
grouped too. Filters mirror the directory under the owning tree (component, build tree as
Generated/, or a `*_SOURCE_PATH` dependency); component pins must reference metamod 55f3fc9+.

For formatting install clang-format==23.1.3, configure FORMAT_VALIDATION_ONLY=ON,
then build format-check or format. No native compiler is needed. Generated .clang-format
is ignored; vendor trees are excluded. Normal native builds do not install Python tooling.

## Default build and install layout

Use separate `build/x86/Debug` and `build/x86/Release` trees. Single-config generators use `CMAKE_BUILD_TYPE`; multi-config generators derive Debug/Release from the tree leaf, falling back to Release for legacy paths. The default install prefix is `<top-level-source>/install/x86/<Config>`, containing the full `addons/metamod/` payload (DLL/SO, Windows PDBs, and aggregate plugins.ini). Explicit prefixes and existing cached prefixes are preserved. SDK imports and nested component builds do not change the caller's prefix. Aggregate and component CI use this layout; release ZIPs retain `build/addons/` via temporary staging. Shared layout helpers are provided by Metamod; standalone plugin pins must reference the revision providing them.

## Aggregate Visual Studio solution folders

- Trigger: component targets, nested tests and native vendor libraries appear flat at the aggregate solution root. `USE_FOLDERS` enables grouping but does not assign a target's `FOLDER`.
- Implementation: the aggregate includes `cmake/TargetFolders.cmake` and calls `mmfg_assign_solution_folders()` after all components and FormatValidation targets exist. It walks configured `SUBDIRECTORIES` and `BUILDSYSTEM_TARGETS`, including component tests; imported and alias targets are not collected.
- Layout: each component has its own top-level folder (`metamod`, `asext`, `ascurl`, `asqcvar`, `asusermsg`, `fallguys`, `slowdl`, `gsphysfix`), containing its native, test and gamedata targets. Vendor build subtrees map to `ThirdParty/Bullet`, `ThirdParty/Capstone` and `ThirdParty/Procmap` when configured. All format targets, including aggregate `format` and `format-check`, use `FormatValidation`.
- Constraint: Bullet and Capstone sources can be nested inside component trees or supplied by overrides/FetchContent. Match the fixed `vendor/<dependency>` binary subtrees before matching component source roots; use `cmake_path(IS_PREFIX ... NORMALIZE ...)` for directory boundaries. Preserve any explicit nonempty target `FOLDER`, including CTest's existing folders.
- Applicability: aggregate IDE organization only. Component repositories, standalone builds, public APIs, build dependencies and project-internal source filters are unchanged. `USE_FOLDERS` is enabled before the formatting-only early return as well.
- Verification (2026-10-20): VS 2022 Win32 Debug/Release configure and generated solution folder checks passed; both retained all 56 projects and identical project dependency graphs. Separate partial-component, tests-disabled and formatting-only configurations passed. A temporary compiler-free fixture verified Procmap, external vendor sources, nested vendor precedence, component path boundaries and explicit folder preservation. Aggregate `format-check` passed for all seven components (186 files, zero differences), and `git diff --check` passed. No native compilation was performed. Validation scripts and build artifacts are temporary and are not tracked.
