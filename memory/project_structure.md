---
title: project_structure
type: note
permalink: metamod-fallguys/project-structure
---

# Project Structure

The aggregate owns orchestration, packaging, documentation and the plugin list.
`metamod/`, `asext/`, `ascurl/`, `asqcvar/`, `asusermsg/`, `fallguys/`, `slowdl/`, `gsphysfix/` are git submodules
of independent repositories in the metamod-svencoop organization.

- Components: root CMakeLists.txt and cmake/, public include/, private src/.
- Metamod: include/HLSDK/ is the preserved SDK tree; thirdparty/capstone_fork and
  thirdparty/procmap_fork are private nested submodules. Plugin SDK excludes src/.
- ASExt: include/asext_api.h and include/std_string.h are its existing public interface.
- ASCurl: thirdparty/curl and thirdparty/openssl retain header trees.
- FallGuys: thirdparty/fmod retains headers; thirdparty/bullet3_fork is a nested
  submodule, so Bullet is FallGuys-owned and compiled from its own fork.
- Aggregate thirdparty/: angelscript-sdk and FormatValidation submodules.
- scripts/: legacy names wrap unified CMake builds. docs/ contains component navigation;
  usage lives in each component's USAGE.md. memory/ remains project knowledge.
- Installed payload: addons/metamod/dlls/{metamod,asext,ascurl,asqcvar,asusermsg,fallguys}.dll
  or .so; Windows includes PDBs. addons/metamod/plugins.ini is aggregate-owned.

CMake is the only build system. Prebuilt thirdparty/install is no longer an input.
Preserve existing licensing declarations; the aggregate LICENSE is MIT, while imported
source trees retain their original licenses.
