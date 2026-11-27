---
title: game_private_funcs_vars
type: note
permalink: metamod-fallguys/game-private-funcs-vars
---

# 游戏 DLL 私有函数/私有变量引入机制（fallguys / asext）

## 概述
本仓库通过一套“声明/定义函数指针 + 运行时从 gamedata 解析地址”的宏体系，把游戏 `server.dll/server.so` 与 `hw.dll/hw.so` 内的私有函数、私有全局变量地址引入到插件中，并在需要时进一步做 inline hook、或者直接调用。
自 issue #55（2026-10）起，asext/fallguys **只**通过 metamod 托管的 GoldSrc_VibeSignatures gamedata 定位私有符号；手写签名、Linux 5.15 符号分支、手写 GOT/PLT 解析均已删除（详见 [[gamedata]]）。

## 职责
- 用 `gamesymbols.h`（原 `signatures.h`，git mv 而来）为每个符号 `Xxx` 定义 `Xxx_GameSymbol`（gamedata 中的 `symbolName`）。
- 用 `serverdef.h`（以及 fallguys 的 `enginedef.h`）声明私有函数指针类型 `fnXxx`，并用 `PRIVATE_FUNCTION_EXTERN(Xxx)` 暴露全局指针。
- 在某个 `.cpp`（通常是 `server_hook.cpp`）里用 `PRIVATE_FUNCTION_DEFINE(Xxx)` 落地定义 `g_pfn_Xxx / g_call_original_Xxx`。
- 在 `meta_api.cpp` 的 `Meta_Attach` 中通过 `FILL_FROM_GAMEDATA / VAR_FROM_GAMEDATA` 在运行时填充函数指针与变量地址；任一失败即 `LOG_ERROR` 并 `return FALSE` 拒绝加载插件。
- （可选）通过 `INSTALL_INLINEHOOK(Xxx)` 把 `g_pfn_Xxx` 替换为 `NewXxx`，并把 trampoline 写回到 `g_call_original_Xxx` 供后续调用原函数。

## 涉及文件 (不要带行号)
- metamod/include/signatures_template.h
- metamod/include/gamesymbol.h
- metamod/include/mutil.h
- fallguys/src/gamesymbols.h
- fallguys/src/serverdef.h
- fallguys/src/enginedef.h
- fallguys/src/meta_api.cpp
- fallguys/src/server_hook.cpp
- fallguys/scripts/manifests/fallguys.json
- asext/src/gamesymbols.h
- asext/src/serverdef.h
- asext/src/meta_api.cpp
- asext/src/server_hook.cpp
- asext/scripts/manifests/asext.json

## 架构
1) “函数指针”这条线：`PRIVATE_FUNCTION_EXTERN/DEFINE`
- 宏定义在 `metamod/include/signatures_template.h`：
  - `PRIVATE_FUNCTION_EXTERN(name)` 展开为 `extern fn##name g_pfn_##name; extern fn##name g_call_original_##name;`
  - `PRIVATE_FUNCTION_DEFINE(name)` 展开为 `fn##name g_pfn_##name; fn##name g_call_original_##name;`
- 约束（命名耦合）：
  - 必须先有 `typedef ... (*fnName)(...)`，宏通过 `fn##name` 拼出类型名。
  - 变量名固定为 `g_pfn_Name`（当前地址）与 `g_call_original_Name`（未 hook 时与 `g_pfn_` 相同；hook 后变为 trampoline）。

2) “符号名来源”这条线：`gamesymbols.h`
- 每个符号定义 `#define Xxx_GameSymbol "<symbolName>"`，宏通过 `name##_GameSymbol` 拼接引用。
- C++ 函数的 symbolName 是去修饰后的签名，**Windows/Linux 写法不同**（`unsigned int` vs `unsigned long`，`std::string` vs `std::__cxx11::basic_string<...>`），用 `#ifdef _WIN32` 分流；个别 Linux 符号上游以 mangled 名发布（如 `_ZN11CPlayerMove13PlayStepSoundEifb`）。
- 本地变量名与上游名不同时只改宏值：`pmovevars_GameSymbol "movevars"`、`pg_groupop_GameSymbol "g_groupop"`、`g_pServerManager_GameSymbol "CASServerManager::GetInstance()::pInstance"`。
- 同一 symbolName 可被两个本地别名引用：`SV_SingleClipMoveToEntity` 与 `SV_SingleClipMoveToEntity_10152` 都对应 `"SV_SingleClipMoveToEntity"`（8948/10257 各自发布的是该版本的实现），由 `g_pfn_build_number() >= 10152` 选择用哪个函数指针类型。
- 每个 `_GameSymbol` 必须同时出现在插件 manifest（`scripts/manifests/<plugin>.json` 的 `symbols` 或 `platformSymbols`）中，否则构建期不会把它保留进裁剪后的 gamedata。

3) “运行时填充”这条线：`FILL_FROM_GAMEDATA / VAR_FROM_GAMEDATA`
- 宏在 `metamod/include/signatures_template.h`，核心是 `RESOLVE_FROM_GAMEDATA(dll, name, kind, out)`：调用 `gpMetaUtilFuncs->pfnResolveGameSymbol(dll##Base, name##_GameSymbol, kind, &addr)`，失败时 `LOG_ERROR(... pfnGetGameSymbolStatusString(status))` 并 `return FALSE`。
- `dll` 是 token（`engine`/`server`），宏内部拼 `engineBase/serverBase`，因此 `meta_api.cpp` 仍须按约定命名局部变量 `engineBase`、`serverBase`（不再需要 `*CodeBase/*CodeSize`）。
- `FILL_FROM_GAMEDATA(dll, name)`：kind=FUNCTION，写入 `g_pfn_name` 并同步 `g_call_original_name`。
- `VAR_FROM_GAMEDATA(dll, name)`：kind=GLOBAL，`name = (decltype(name))addr`。GLOBAL 解析结果是**变量本身的地址**（`gv_rva`），不是读出来的指针：`g_pServerManager` 是 `CASServerManager**`（= `&pInstance`）、`sv_models` 是数组地址、`host_frametime` 是 `double*`。
- 旧的 `FILL_FROM_SIGNATURE* / VAR_FROM_SIGNATURE* / FILL_FROM_SYMBOL` 宏已随接口 5:18 清理移除，所有插件统一使用 gamedata 宏。

4) fallguys 插件的落地方式
- server：`PM_PlaySoundFX_SERVER`、`CPlayerMove_PlayStepSound`、`RegisterSCScriptColor24`。
- engine 函数：`build_number`、`SV_Physics`、`SV_PushEntity`、`SV_PushMove`、`SV_PushRotate`、`SV_WriteMovevarsToClient`、`SV_SingleClipMoveToEntity(_10152)`。
- engine 变量：`sv_models`、`host_frametime`、`pmovevars`、`sv_areanodes`、`pg_groupop`、`pg_groupmask`。
- `Meta_Query` 要求 metamod 接口 ≥ 5:18（需要 5:18 `mutil_funcs_t` 布局与 `pfnResolveGameSymbol`）。

5) asext 插件的落地方式
- 39 个 server 函数（`CASHook_*`、`CString_*`、`asGetActiveContext`、`CASDocumentation_*`、`CASBaseManager_GetTypeInfoByName`、`CScriptDictionary_*` 等）+ `VAR_FROM_GAMEDATA(server, g_pServerManager)`。
- `Meta_Query` 用 strcmp 严格比较接口版本，asext 随 metamod 一起以 5:18 构建。

## 依赖
- metamod mutil：`pfnResolveGameSymbol`、`pfnGetGameSymbolStatusString`（接口 5:18）。
- metamod 在 `meta_load_gamedll()` 之后、`Plugins->load()` 之前加载 `<gamedir>/addons/metamod/gamedata/**/index.json`。

## 注意事项
- 强命名耦合：`PRIVATE_FUNCTION_*` 要求类型名是 `fn<name>`，gamedata 宏要求 `<name>_GameSymbol`；`dll` token 必须对应 `meta_api.cpp` 中的 `engineBase/serverBase`。
- hook 语义：未安装 hook 时，`g_call_original_*` 通常与 `g_pfn_*` 相同；安装 hook 后 `INSTALL_INLINEHOOK` 会把 trampoline 写回 `g_call_original_*`，调用原函数应走 `g_call_original_*`。
- 新增私有符号：在上游 GoldSrc_VibeSignatures 确认已发布 → 加 manifest 条目（两平台名字不同就放 `platformSymbols`）→ 加 `_GameSymbol` 宏 → 加 `FILL/VAR_FROM_GAMEDATA`。symbolName 未命中不会回退到 artifact 名，插件直接拒绝加载。
- 游戏版本不在 gamedata 覆盖范围（目前 svencoop-8948 / svencoop-10257）时，按模块 CRC-64 查不到 → `module was not found in the gamedata`，插件拒绝加载。

## 调用方（可选）
- `fallguys/src/meta_api.cpp`、`asext/src/meta_api.cpp`：负责在 `Meta_Attach` 阶段填充所有私有函数/变量。
- `fallguys/src/server_hook.cpp`、`asext/src/server_hook.cpp`：定义私有函数指针、安装 inline hook，并在业务逻辑中使用 `g_pfn_* / g_call_original_*`。

## Relations
- relates_to [[gamedata]]
- relates_to [[PrivateSymbols]]
- relates_to [[gamedata-signature-parity]]
