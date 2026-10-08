# ArgusAI Claude Code Plugin

本目录包含 ArgusAI 的 Claude Code Plugin，业务团队安装后即可让 AI 编程助手直接执行 Docker 容器 E2E 测试。

## 安装方式

```bash
# 第 1 步：注册 marketplace（只需一次）
claude plugin marketplace add jeffkit/argusai-marketplace

# 第 2 步：安装 plugin
claude plugin install argusai
```

## 安装后你会获得什么

### MCP 工具（23 个）

安装后 AI 自动获得以下 MCP 工具，无需额外配置。

> **计数口径**：`argusai` 仓 `packages/mcp/src/server.ts` 中 `server.tool(...)` 的注册数（源码注释编号 Tool 1–23）。该文件有 22 条 `^import { … handle … }` 导入语句，其中 `packages/mcp/src/tools/run.ts` 同时导出 `handleRun` 与 `handleRunSuite` 两个处理器，故 22 条导入 = 23 个工具。
>
> **分类**取自 `server.ts` 的分节注释（`[lifecycle]` / `[mock]` / `[diagnostic]` / `[history]` / `[knowledge]`）；顺序即源码注册顺序。

| # | 工具 | 分类 | 用途 |
|---|------|------|------|
| 1 | `argus_init` | lifecycle | 加载项目 `e2e.yaml` 并创建会话 |
| 2 | `argus_build` | lifecycle | 构建项目服务的 Docker 镜像 |
| 3 | `argus_setup` | lifecycle | 启动测试环境（网络 + Mock + 容器） |
| 4 | `argus_run` | lifecycle | 运行全部/筛选后的测试套件（缺失容器时自动 setup） |
| 5 | `argus_run_suite` | lifecycle | 运行单个指定套件，输出完整每步结果 |
| 6 | `argus_status` | lifecycle | 查看所有受管资源的当前状态 |
| 7 | `argus_logs` | lifecycle | 查看指定容器的近期日志 |
| 8 | `argus_clean` | lifecycle | 停止并移除容器、网络与 Mock |
| 9 | `argus_mock_requests` | mock | 查看 Mock 服务录制的请求 |
| 10 | `argus_preflight_check` | diagnostic | 对环境运行健康预检 |
| 11 | `argus_reset_circuit` | diagnostic | 重置熔断器以探测 Docker 可用性 |
| 12 | `argus_history` | history | 查询项目历史测试运行记录（含通过/失败计数、分页） |
| 13 | `argus_trends` | history | 按时间获取趋势数据（通过率 / 时长 / flaky） |
| 14 | `argus_flaky` | history | 按不稳定度对测试用例排名 |
| 15 | `argus_compare` | history | 对比两次运行，识别回归与修复 |
| 16 | `argus_diagnose` | knowledge | 失败诊断：分类 + 匹配已知模式 + 返回排序后的修复建议 |
| 17 | `argus_report_fix` | knowledge | 记录修复是否生效，提升模式置信度 |
| 18 | `argus_patterns` | knowledge | 浏览/筛选失败模式知识库（内置 + 已学习） |
| 19 | `argus_mock_generate` | mock | 从 OpenAPI spec 生成 `e2e.yaml` mock 配置 |
| 20 | `argus_mock_validate` | mock | 校验 mock 路由对 OpenAPI spec 的覆盖度 |
| 21 | `argus_resources` | diagnostic | 查看所有项目下 ArgusAI 管理的 Docker 资源 |
| 22 | `argus_rebuild` | lifecycle | 一键重建：clean → init → build → setup |
| 23 | `argus_dev` | lifecycle | 一键启动供手动测试：init → build → setup |

### 斜杠命令

| 命令 | 说明 |
|------|------|
| `/run-tests [suite-id]` | 一键运行 E2E 测试（自动 build → setup → run → report） |
| `/init-e2e` | 为当前项目初始化 ArgusAI 配置 |

### Skill（自动触发）

AI 在检测到以下场景时会自动使用 ArgusAI 技能：
- 项目中存在 `e2e.yaml` 文件
- 用户要求运行 E2E 测试、验证接口、测试服务
- 用户提到 "跑一下测试"、"run e2e tests" 等关键词

## 前置条件

使用前请确保：
- **Docker** 已安装且 Daemon 已启动
- **Node.js >= 20** 已安装
- **argusai-mcp** 已全局安装（`npm install -g argusai-mcp`）或位于 PATH 中
- 项目目录中有 `e2e.yaml` 配置文件

## 目录结构

```
argusai-marketplace/
├── .claude-plugin/
│   └── marketplace.json          # Marketplace 清单
├── argusai/                      # ArgusAI Plugin
│   ├── .claude-plugin/
│   │   └── plugin.json           # Plugin 清单
│   ├── .mcp.json                 # MCP Server 配置
│   ├── commands/
│   │   ├── run-tests.md          # /run-tests 命令
│   │   └── init-e2e.md           # /init-e2e 命令
│   └── skills/
│       └── argusai/
│           ├── SKILL.md          # 核心 Skill 定义
│           └── references/       # 参考文档
│               ├── authoring-guide.md
│               ├── e2e-yaml-full-reference.yaml
│               ├── playwright-browser-tests.yaml
│               ├── plugin-assertions.md
│               ├── plugin-development.md
│               └── runner-guide.md
└── README.md                     # 本文件
```

## 使用示例

安装 plugin 后，在 Claude Code 中直接对话即可：

```
用户：帮我跑一下这个项目的 E2E 测试

AI：（自动识别 e2e.yaml → argus_init → argus_build → argus_setup → argus_run → 报告结果）

用户：health 测试失败了，帮我看看怎么回事

AI：（自动调用 argus_logs 查看日志 → 分析失败原因 → 给出修复建议）

用户：Docker 环境好像有问题，帮我检查一下

AI：（调用 argus_preflight_check 检查环境 → 发现孤儿容器 → autoFix 自动清理 → 环境恢复正常）

用户：构建一直失败，熔断了怎么办

AI：（调用 argus_reset_circuit 重置熔断器 → 重新尝试构建）
```

或使用斜杠命令：

```
/run-tests          ← 运行所有测试
/run-tests health   ← 只运行 health 套件
/init-e2e           ← 为当前项目初始化配置
```
