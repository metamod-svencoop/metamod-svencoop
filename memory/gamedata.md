---
title: gamedata
type: note
permalink: metamod-fallguys/gamedata
tags:
- metamod
- gamedata
- asext
- fallguys
- private-symbols
---

# gamedata — metamod 托管的 GoldSrc_VibeSignatures 符号目录

## 概述
自 issue #55（2026-10）起，asext/fallguys 的所有引擎私有符号都从 [GoldSrc_VibeSignatures](https://github.com/hlnd2t/GoldSrc_VibeSignatures) 发布的 gamedata 解析，替代手写签名。实现是 MetaHookSv `MetaHook/src/GameData.cpp` 的精简移植：只保留 function/global 两种 kind，按当前平台（`windows`/`linux`）加载记录，不支持 mirror/client alias；BLOB 来源注册见下文 issue #34 补充。

## 涉及文件
- `metamod/src/gamedata.h`、`metamod/src/gamedata.cpp`：运行时目录（namespace `GameData`）
- `metamod/include/gamesymbol.h`：`meta_gamesymbol_kind_t`、`meta_gamesymbol_status_t`（取值与 MetaHookSv 一致）
- `metamod/include/mutil.h`、`metamod/src/mutil.cpp`：`pfnResolveGameSymbol`、`pfnGetGameSymbolStatusString`（接口 5:17，追加在 `pfnGetCodeSize` 之后）
- `metamod/include/meta_api.h`：`META_INTERFACE_VERSION "5:17"`
- `metamod/src/metamod.cpp`：`meta_init_gamedata()`，在 `meta_load_gamedll()` 之后、`Plugins->load()` 之前
- `metamod/include/signatures_template.h`：`RESOLVE_FROM_GAMEDATA` / `FILL_FROM_GAMEDATA` / `VAR_FROM_GAMEDATA`
- `metamod/scripts/sync-gamedata.py`：构建期同步/裁剪/校验脚本（MetaHookSv 同名脚本改造）
- `metamod/cmake/Build.cmake`：`mmfg_add_gamedata(target manifest)`；`metamod/cmake/Dependencies.cmake`：`mmfg_gamedata_headers()`（RapidJSON + Chocobo1Hash）
- `asext/scripts/manifests/asext.json`、`fallguys/scripts/manifests/fallguys.json`
- 测试：`metamod/tests/test_gamedata.cpp`（CTest `mmfg_test_gamedata`）、`metamod/scripts/tests/test_sync_gamedata.py`

## 数据格式
- 上游索引 `https://hlnd2t.github.io/GoldSrc_VibeSignatures/gamesymbols/index.json`：index schema 4；快照 schema 5，`source.snapshotSchemaVersion` 8，`analysisOutputContractVersion` 3。
- 记录 `{platform, module, symbolName, kind, payload}`；function payload `func_rva/func_size`，global payload `gv_rva`；`binaries.<module>.<platform>.crc64` 为模块文件 CRC-64/XZ。
- 安装布局：`addons/metamod/gamedata/<plugin>/{index.json, svencoop-8948.json, svencoop-10257.json}`。裁剪后的快照用稳定文件名 `<gameVersion>.json`，内容确定（Windows/Linux 构建逐字节相同）。

## 运行时流程
1. `meta_init_gamedata()`：`CollectIndexDirs(<gamedir>/addons/metamod/gamedata)` 递归收集（排序、跳过符号链接）所有含 `index.json` 的目录 → `Initialize()`；每个 index 可选，至少一个加载成功即可用。诊断逐条 `META_WARNING("Gamedata: ...")`，成功时 `META_LOG("Loaded gamedata catalog from ... (N index file(s))")`（写入游戏根目录 `metamod.log`）。
2. 快照校验 size + SHA-256 + 各 schema 版本；同一 (crc64, symbolName) 的内容不同的重复记录标记为 conflicted；非 function/global kind 记为 unsupported。
3. `Resolve(base, symbolName, kind)`：取模块路径（Windows `GetModuleFileNameA`；Linux `dladdr` 且要求 `dli_fbase == base`）→ 计算文件 CRC-64（按 base 缓存，带互斥）→ 查 (crc64, symbolName) → 校验 kind → Windows 用 PE `SizeOfImage` 做越界检查（Linux 不检查）→ 返回 `base + rva`。
4. Linux 基址：插件传入的 `serverBase/engineBase` 来自 `mutil_GetModuleBaseByHandle`（`dladdr(l_ld).dli_fbase`）；Sven 的 `server.so`/`hw.so` 第一个 PT_LOAD vaddr 为 0，所以 `dli_fbase == l_addr`，RVA 直接相加即可。

## 关键决策
- **按 symbolName 查找**（不是 artifact 名）：C++ 将来可能出现同名重载；symbolName 未命中**不回退**，插件拒绝加载。
- **只用 gamedata**：删除签名、Linux 5.15 符号分支、手写 GOT/PLT 解析；旧签名宏只为仓库外插件保留在公共头中。
- **metamod 托管**：目录只加载一次，插件通过 mutil 接口查询；接口版本升到 5:17，因为 `mplugin.cpp` 只拷贝 metamod 自身 `sizeof(mutil_funcs)`，旧 metamod 会给新插件未初始化的成员。
- Linux 专用服务器引擎是 `hw.so`，与客户端同一份二进制；Windows 专用服务器也用同一份 `hw.dll`。

## 构建期同步
- `mmfg_add_gamedata()` 定义 `MMFG_SYNC_GAMEDATA`（默认 ON，需要 Python 3.8+）。自定义目标 `<plugin>_gamedata` 先运行 `sync-gamedata.py --manifest ... --target-dir ${PROJECT_BINARY_DIR}/assets/addons/metamod/gamedata/<plugin> --temp-root ${PROJECT_BINARY_DIR}/gamedata-sync`，再用 `--validate-only` 做覆盖校验；`install(DIRECTORY ... OPTIONAL)`。
- 拿不到上游索引时回退到上次缓存的索引离线构建；目标路径只接受 `.../addons/metamod/gamedata[/<plugin>]`；加锁 Windows 用 msvcrt，POSIX 用 fcntl。
- manifest 字段：`gameVersions`、`platforms`（默认 windows+linux）、`symbols`（两平台共用名）、`platformSymbols`（按平台的名字，如 Windows `unsigned int` 与 Linux `unsigned long` 两种写法）。覆盖校验按 (gameVersion, platform) 逐一检查。
- 没有移植 MetaHookSv 那个 83 KB 的 `validate-gamedata.py`；`--validate-only` 加上覆盖校验就是质量门槛。

## 验证（2026-10-13）
- 阶段一：签名与 gamedata 逐项比对，所有差异都是我方签名的错误（[[gamedata-signature-parity]]）。
- Windows Sven Co-op 10257 实机（halflife-mcp）：
  - `Loaded gamedata catalog ... (2 index file(s))`。
  - 56 个符号全部解析，每个模块反推出唯一基址。
  - `meta list` 中 5 个插件均为 RUN。
  - AngelScript 冒烟（PlayerPostThinkPost、PlayerMovePlaySoundFX + GetRunPlayerMovePlayerIndex、EnablePhysicWorld、SetGroupTrace）正常。
  - 退出码 0xC0000409 是迁移前就存在的问题（旧版 DLL 同样出现）。
- CTest：Windows 8/8，WSL Ubuntu 24.04 i386（带 ASan/UBSan）15/15。
- Python：`python -B -m unittest discover -s metamod/scripts/tests` 14 个用例通过。
- CI：metamod 组件的 `build.yml` 以 `MMFG_METAMOD_BUILD_TESTS=ON` 配置，并在 Windows/Ubuntu × Debug/Release 上跑 CTest 和上面的 Python 测试（Debug 本地也验证过：Windows 8/8，Linux 15/15）。
- Linux 运行时没有真机验证：WSL 缺少 32 位依赖（libcurl/SDL2/GL/steam_api），无法 dlopen 真实的 `.so`。

## 新增符号步骤
1. 在上游快照中确认两个平台都发布了，并记下各自的 symbolName。
2. 把它加入插件 manifest：两平台同名放 `symbols`，名字不同放 `platformSymbols`。
3. 在插件 `gamesymbols.h` 中加 `X_GameSymbol`，如需区分平台用 `#ifdef _WIN32`。
4. 在 `Meta_Attach` 中加 `FILL_FROM_GAMEDATA` 或 `VAR_FROM_GAMEDATA`。
5. 用 `--validate-only` 在构建时确认覆盖校验通过。

## Relations
- relates_to [[game_private_funcs_vars]]
- relates_to [[PrivateSymbols]]
- relates_to [[gamedata-signature-parity]]
- relates_to [[asext]]

## Issue #34：SlowDL 与 BLOB 来源注册（2026-10-20）

- 触发信号：HL 3248/3266/3329/3647 的 BLOB 引擎不在普通 Windows 模块表内，旧 VAC 路径仅估算代码范围，不能提供原始文件 CRC 与可靠映像范围。
- 正确做法：`src/blob.cpp` 只解析原始文件描述；`engineinfo.cpp` 从可执行文件目录查找候选，核对当前进程映射和公开 engine 函数代码后注册来源。候选缺失、不匹配或有歧义时失败；不加载引擎、不伪造 HMODULE。
- `GameData::RegisterModuleFileSource` 关联运行时基址、原始文件路径、映像大小和有效节范围。CRC 仍来自原始 hw.dll，地址为已验证基址加 RVA；解析前优先查注册来源，返回地址须落入有效节。重注册及注销清除该基址 CRC 缓存，EngineInfo 关闭时注销来源。公共接口保持 5:17。
- SlowDL 为独立插件，只依赖 Metamod SDK，解析下载入口、DropClient、host_client、cmd_source、sv_player 五个符号；客户端保持不透明。普通 UDP 下载可踢出，customdecal 与 Sven 当前地图 soundcache 豁免；两个日志开关独立。manifest 分成双平台 slowdl 与 Windows-only slowdl_windows，避免更改同步器契约。
- Hook 生命周期经验：InlineHook 的事务只在启动插件加载后统一提交；PT_ANYTIME 插件在运行时加载必须立即启用 Hook。Metamod 注册插件 cvar 时复制结构，运行时值须通过 engine 的 CVarGetFloat 查询，不能读取原始 cvar_t.value。
- 验证：Windows Debug/Release 各 9 个 Metamod 自动测试和 1 个 SlowDL 测试通过；Linux i386 各 16+1 通过（Metamod 带 ASan/UBSan）。四个真实原始 BLOB 文件的 CRC、解析及 20 个符号地址通过外部输入测试，测试映像不是运行中的游戏。15 个平台输入的 manifest 覆盖验证通过。
- 实机边界：已有 Windows Sven 10257 的 FastDL 成功、HTTP 失败 UDP 回退、普通下载踢出及 soundcache 豁免证据；未对全部版本、customdecal 和最后的运行时加载修正做实机验证。用户要求停止后续实机测试，未测项目不记为实机通过。
- 适用范围：Metamod 内部 BLOB 身份解析及使用事务 Hook/插件 cvar 的新插件。组件 CTest 使用构建树中的 metamod/tests 目录，避免混入第三方 procmap 的未构建 simple 示例。


## Decay 索引兼容性（2026-10-24）

- 触发信号：GoldSrc_VibeSignatures v20261024a 新增无版本标签 `decay`；默认 Pages 索引已发布 27 个条目。旧同步器报 `remote index version entry 14 contains an invalid gameVersion: 'decay'`。
- 根因 / 约束：同步器先校验完整上游 index，再按 manifest 筛选。即使只选择 Sven Co-op，也会被未选中的 Decay 阻断。下载缓存回退不会捕获索引解析错误。
- 正确做法：保留原版本化标签规则，显式追加唯一无版本例外 `decay`；不放开其他未知无版本标签。现有 ASExt、FallGuys、SlowDL、ASExtHook、HalflifeCLI manifest 无需选择 Decay 或客户端鼠标符号。astemplate 与 asexthook 通过更新 Metamod 子模块复用修复，HalflifeCLI 修复自身脚本副本。
- 验证方式：新增 5 项行为测试先复现拒绝，再验证稳定/内容寻址文件名、manifest gameVersions/symbolExemptions/conditionalGroups.when/exempt，以及未选择的 Decay 不阻断筛选。Metamod Python 19 项通过；Windows Release 原生构建、格式检查通过；CTest 9 项通过、BLOB 外部样本测试跳过。8 组消费者在线同步和 validate-only 全通过，共 32 份裁剪快照；固定 `PYTHONHASHSEED=0` 后与旧裁剪逻辑逐字节一致。
- 适用范围：消费完整 GoldSrc_VibeSignatures index 的构建期同步器；此次不改变运行时 API 和游戏支持范围。已有 `prune_snapshot` 遍历 used_modules 集合，跨 Python 进程的二进制对象键顺序可能受随机 hash seed 影响，因此逐字节对比要固定 seed；不能据此声称任意环境下字节顺序一致。

## GSPhysFix 消费者（2026-10-25）

- 新增 `gsphysfix_windows` / `gsphysfix_linux` 两个 manifest：Linux 用 `conditionalGroups` 按 gameVersion 选择 `SV_CopyEdictToPhysent`（hl-8684）或 `SV_AddLinksToPM_` + `pmove` + `build_number`（hl-10210、svencoop-8948/10257），`alternativeGroups` 覆盖 Linux mangled 名。conditionalGroups/symbolExemptions 只按 gameVersion 生效，平台差异须拆分 manifest。
- 运行时 gamedata 仍只支持 function/global；上游 `scalar`（如 `playermove_t_*_offset`）不可直接消费，需用 HLSDK 固定结构 + static_assert 对齐上游数值。详见 [[gsphysfix]]。
