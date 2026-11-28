---
title: Release workflow (tag-triggered, AI notes, draft-first publish)
type: note
permalink: metamod-fallguys/memory/release-workflow-tag-triggered-ai-notes-draft-first-publish
tags:
- ci
- release
- workflow
- publishing
---

# Release workflow (tag-triggered, AI notes, draft-first publish)

## 触发信号
- 需要给 `v*` tag 发布二进制，或修改 `.github/workflows/{release,windows,ubuntu}.yml`、`scripts/release*.py`。
- 线上 release 名称/资产不对，或需要调整 release notes 生成方式。

## 约束与结构
- 移植自 `D:\MetaHookSv`（`release.yml` + `scripts/release.py` + `scripts/release_git.py`），
  provider 兜底默认改为 `claude`。
- `release.yml` 仅在 `push: tags: ['v*']` 触发，`concurrency: release-<ref>`；三个 job：
  - `windows` / `ubuntu`：`uses: ./.github/workflows/{windows,ubuntu}.yml`（这两个文件是
    `workflow_call` 可复用 workflow，只构建 + 上传 `release-windows` / `release-ubuntu` artifact）。
  - `notes`：`environment: release`，`checkout fetch-depth: 0 + submodules: recursive`，
    先 `release.py context` 生成证据，装 claude/codex CLI，再 `release.py notes`，上传 `release-notes`。
  - `publish`：`permissions: contents: write`，下载三个 artifact 到 `release-assets/`，再
    `release.py publish --assets release-assets --notes release-notes/release-notes.md`。
- 六个资产（`EXPECTED_ASSETS`）：`release-windows-{x86,x64}/metamod-svencoop-windows-<arch>.7z`、
  `-with-pdb.7z`、`release-ubuntu-{i386,x86_64}/metamod-svencoop-ubuntu-<arch>.7z`；workflow 用 runner 自带的
  `7z a -t7z` 打包 `build/addons/` 布局（Windows 无 PDB 包用 `-xr!*.pdb`），已弃用 `thedoctor0/zip-release`。
- `windows.yml`/`ubuntu.yml` 已去掉 `push: tags` 与 `softprops/action-gh-release`，避免 tag
  双触发重复发布；两者都跑 `python -B -m unittest discover -s scripts/tests -p 'test_release*.py'`。
- `release_git.py` 原样移植，是只读 Git stdio MCP（单工具 `git_history`：log/show/diff/ls-tree），
  子模块从 `ls-tree` gitlink 解析、`protocol.allow=never`、凭证隔离，与仓库无关。
- 必需仓库设置：`release` environment；secrets `RELEASE_NOTES_API_KEY`、`RELEASE_NOTES_BASE_URL`
  （HTTPS、无凭证/query/fragment）、`RELEASE_NOTES_MODEL`（无默认，必填）；var
  `RELEASE_NOTES_PROVIDER`（缺省 claude）。缺失则 notes fail-closed，不发布。
- tag 匹配前缀 `refs/tags/v`；mmfg tag 是 lightweight（commit），`verify_tag` 已兼容 annotated。

## 正确做法
- 改仓库身份相关处：`INSTRUCTIONS` 文案、`DIFF_PATHS` 排除集（mmfg 特有 `build-cmake/`、`install/`、
  `.vs/`、`Debug/`、`Release/`、`intermediate/`）、`EXPECTED_ASSETS`、upload `Content-Type`
  （按后缀，`.7z`→`application/x-7z-compressed`）、release `name`（`metamod-svencoop-<tag>`）。
- 不要动 `release.py` 的安全不变量：draft-first、资产精确集合校验、tag 未移动校验、
  双语 notes 校验、重试后 fail-closed、凭证回显拒绝、codex CLI 事件白名单。

## 验证方式
- `python -B -m unittest discover -s scripts/tests -p 'test_release*.py' -v`（42 个用例全过）。
- `actionlint`（对全部 workflow）必须 0 问题。
- `context`/`publish` 依赖 `GITHUB_REF`/`GITHUB_SHA`，只能在 tag 事件里端到端验证；
  本地可对真实仓库 smoke `build_context`（给一个祖先 tag 作 baseline），确认输出有界且
  含 `SUBMODULE CHANGES`。

## 适用范围
- 仅本仓库的 tag 发布链路。组件仓库各自独立发布，改动需先提交组件仓库再更新本仓库 gitlink。
