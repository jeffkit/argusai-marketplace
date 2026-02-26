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

### MCP 工具（11 个）

安装后 AI 自动获得以下 MCP 工具，无需额外配置：

| 工具 | 说明 |
|------|------|
| `argus_init` | 加载项目 e2e.yaml 配置 |
| `argus_build` | 构建 Docker 镜像（含熔断器保护） |
| `argus_setup` | 启动测试环境（预检 + 端口解析 + 孤儿清理 + 网络验证） |
| `argus_run` | 运行测试套件 |
| `argus_run_suite` | 运行单个测试套件 |
| `argus_status` | 查看环境状态 |
| `argus_logs` | 查看容器日志 |
| `argus_clean` | 清理测试资源 |
| `argus_mock_requests` | 查看 Mock 请求录制 |
| `argus_preflight_check` | 主动检查环境健康（Docker/磁盘/孤儿资源），支持自动修复 |
| `argus_reset_circuit` | 重置熔断器状态（Docker 异常恢复后使用） |

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
│       └── argusai-e2e/
│           ├── SKILL.md          # 核心 Skill 定义
│           └── references/       # 参考文档
│               ├── yaml-test-syntax.md
│               └── e2e-yaml-config.md
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
