# AGENTS.md — ArgusAI Marketplace

> ArgusAI 的 Claude Code Plugin marketplace：安装后即可让 AI 跑 Docker E2E 测试。
> 负责人：jeffkit | 创建：2026-02-26

## 项目概述

本仓是 Claude Code Plugin 分发仓库，不是 ArgusAI 引擎源码。  
业务团队 `claude plugin marketplace add` + `install argusai` 后，获得 MCP 工具、斜杠命令与 Skills。  
运行时依赖全局/PATH 中的 `argusai-mcp`（源码在兄弟仓 `argusai`）。

**技术栈：** Markdown, Claude Code Plugin, MCP  
**主仓库：** `git@github.com:jeffkit/argusai-marketplace.git`

## 架构地图

Marketplace 清单指向 `argusai/` 插件目录；插件通过 `.mcp.json` 以 stdio 拉起 `npx argusai-mcp`。  
Skills 指导 AI 何时写/跑 e2e；commands 提供一键流程。

关键目录：
- `.claude-plugin/marketplace.json` — marketplace 清单（插件列表）
- `argusai/.claude-plugin/plugin.json` — 插件元数据
- `argusai/.mcp.json` — MCP Server 启动配置（`npx argusai-mcp`）
- `argusai/commands/` — 斜杠命令（`run-tests`、`init-e2e`）
- `argusai/skills/argusai/` — 核心 Skill + `references/` 参考文档
- `argusai/skills/argusai-author/` — 编写向 Skill（如有内容）
- `README.md` — 安装与能力说明

## 开发约定

**分支策略：** `develop` 开发，`release/test` 测试，`main` 生产；PR 合并。

**禁止事项：**
- 禁止在本仓实现引擎逻辑（改行为请去 `argusai` 仓）
- 禁止修改 `.mcp.json` 的 command 却不验证 `argusai-mcp` 可执行
- 禁止让 Skill/命令文档与 `argusai` 仓实际 MCP 工具名/参数脱节
- 禁止提交与插件无关的构建产物或业务项目配置

## 常用命令

```bash
# 本地验证 marketplace / plugin 清单 JSON
python -m json.tool .claude-plugin/marketplace.json
python -m json.tool argusai/.claude-plugin/plugin.json
python -m json.tool argusai/.mcp.json

# 用户侧安装（需本机已装 Claude Code + Docker + argusai-mcp）
claude plugin marketplace add jeffkit/argusai-marketplace
claude plugin install argusai
```

## 当前状态

**当前里程碑：** {待人工填写}

## 深入阅读

| 文档 | 说明 |
|------|------|
| `README.md` | 安装步骤与 MCP/命令一览 |
| `argusai/skills/argusai/SKILL.md` | Skill 触发与行为 |
| `argusai/skills/argusai/references/` | YAML / runner / 断言参考 |
| `../argusai`（兄弟仓） | 引擎与 MCP 实现 |
