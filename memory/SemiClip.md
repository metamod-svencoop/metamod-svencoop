---
title: SemiClip
type: note
permalink: metamod-fallguys/semi-clip
---

# SemiClip（fallguys）

## 概述
- SemiClip 是按实体/玩家定向的“半碰撞”机制：让某个实体对特定玩家/实体变为非实体碰撞，同时在客户端侧把其 `solid` 设为 `SOLID_NOT`（仅对指定玩家生效）。
- 逻辑分两路：`SemiClip`（影响服务器碰撞 + PlayerMove + 客户端 `solid`）和 `PMSemiClip`（影响 PlayerMove + 客户端 `solid`，不影响服务器 `ShouldCollide`）。

## 核心实现
- 状态存储：`CGameObject` 内部维护 `m_semiclip_to_entities` 与 `m_pm_semiclip_to_entities`（`std::vector<EHANDLE>`），并提供 `Set/Unset/Is*` 方法；`SetPlayerSemiClipMask`/`SetPlayerPMSemiClipMask` 通过 `player_mask` 遍历 `gpGlobals->maxClients` 设置/清除（`fallguys/src/physics.h`）。
- 设置入口：`CPhysicsManager::SetEntitySemiClip*`/`SetEntityPMSemiClip*` 在 `fallguys/src/physics.cpp` 中实现，若 `CGameObject` 不存在会创建并 `AddGameObject`。
- 服务器碰撞过滤：`CPhysicsManager::ShouldCollide` 检查 `IsSemiClipToEntity`，在 `NewShouldCollide`（`fallguys/src/dllapi.cpp`）中使用，返回 false 时 `MRES_SUPERCEDE` 禁止碰撞。
- PlayerMove 过滤：`PM_StartMove` 在 `NewPM_Move` 中调用，通过 `PM_ShouldCollide` 判断：若当前玩家在 `IsSemiClipToEntity` 或 `IsPMSemiClipToEntity` 列表中，则从 `pmove->physents` 排除（`fallguys/src/physics.cpp`）。
- 客户端 `solid`：`CGameObject::AddToFullPack` 若 host 在半碰撞列表中，则将 `state->solid = SOLID_NOT`，让客户端认为该实体对该玩家不可碰撞（`fallguys/src/physics.cpp`）。
- 特殊流程（SolidOptimizer）：`CGameObject::StartFrame` 若启用 `SolidOptimizer`，会先对所有“solid player”设置半碰撞；`CSolidOptimizerGhostPhysicObject::OnTouchRigidBody` 在玩家触碰时调用 `RemovePlayerSemiClipMask` 逐一解除（`fallguys/src/physics.cpp`）。该特性可以大幅度减少客户端侧不必要的ClipNode碰撞检测计算。

## 注意事项
- `player_mask` 按玩家索引 1..`gpGlobals->maxClients` 对应位 `(1 << (entindex - 1))`；`SetEntitySemiClipToPlayer`/`Unset...` 仅接受玩家索引范围（`fallguys/src/server_hook.cpp`）。
- `SemiClip` 与 `PMSemiClip` 的差异：`PMSemiClip` 不影响 `ShouldCollide`（服务器碰撞），但会影响 PM 过滤与客户端 `solid`；`SemiClip` 影响全部路径。
- `SetEntitySemiClip*` 会在实体尚未有 `CGameObject` 时创建对象；实体释放由 `RemoveGameObject` 处理，调用端无需手动创建。
- `PM_StartMove` 用 `std::remove_if` 压缩 `pmove->physents` 并回写 `numphysent`（fallguys `6befe77`，2026-01-17 修复；此前未回写，`[new_end, end)` 尾部残留旧条目，被过滤实体可能仍参与 PM 碰撞，SemiClip 偶发失效）。过滤发生在 `PM_Move` 之前，`onground`/`touchindex` 等 physent 下标均在过滤后计算，因此保持一致；visents/moveents 不过滤。
- `pmove` 声明为 `playermove_t*`，Sven 10152+ 实际是 `playermove_10152_t`（physents 上限 1024，visents 之后字段偏移不同）。`PM_StartMove` 按 `build_number() >= 10152` 选择布局调用模板 `PM_RemovePhysents`；`PM_ShouldCollide` 直接读的 `player_index` 由 static_assert 固定两种布局偏移相同。新增直接经 `pmove` 读取的字段时，须同样按 build 选择布局或加 static_assert。原无调用方且在 10152 布局下偏移错误的 `PM_FlyMove`/`PM_ClipVelocity`/`PM_AddToTouched` 已删除。

## AngelScript-level API（`CEntityFuncs`）

### 预定义
- 预定义宏：`METAMOD_PLUGIN_FALLGUYS`（脚本可用于条件编译）。

### CEntityFuncs 方法
- `bool SetEntitySemiClip(edict_t@ ent, int player_mask)`
- `bool SetEntitySemiClipToPlayer(edict_t@ ent, int entindex)`
- `bool UnsetEntitySemiClipToPlayer(edict_t@ ent, int entindex)`
- `bool SetEntitySemiClipToEntityIndex(edict_t@ ent, int entindex)`
- `bool UnsetEntitySemiClipToEntityIndex(edict_t@ ent, int entindex)`
- `bool SetEntitySemiClipToEntity(edict_t@ ent, edict_t@ targetEntity)`
- `bool UnsetEntitySemiClipToEntity(edict_t@ ent, edict_t@ targetEntity)`
- `bool UnsetEntitySemiClipToAll(edict_t@ ent)`
- `bool SetEntityPMSemiClip(edict_t@ ent, int player_mask)`
- `bool SetEntityPMSemiClipToPlayer(edict_t@ ent, int entindex)`
- `bool UnsetEntityPMSemiClipToPlayer(edict_t@ ent, int entindex)`
- `bool SetEntityPMSemiClipToEntityIndex(edict_t@ ent, int entindex)`
- `bool UnsetEntityPMSemiClipToEntityIndex(edict_t@ ent, int entindex)`
- `bool SetEntityPMSemiClipToEntity(edict_t@ ent, edict_t@ targetEntity)`
- `bool UnsetEntityPMSemiClipToEntity(edict_t@ ent, edict_t@ targetEntity)`
- `bool UnsetEntityPMSemiClipToAll(edict_t@ ent)`
