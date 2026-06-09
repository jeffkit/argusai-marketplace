---
skill_name: argusai
version: 2.0.0
description: >
  ArgusAI E2E testing — run Docker-based tests via the argusai MCP server.
  In Claude Code sessions, MCP tools (argus_setup, argus_run, etc.) are
  called directly. In bash scripts (self-improve.sh, CI), use mcp2cli to
  invoke the same MCP tools from the command line.
  Triggers: "帮我写E2E测试", "创建e2e配置", "跑一下E2E测试", "运行端到端测试",
  "write e2e tests", "run e2e tests", "argusai", "e2e.yaml".
triggers:
  - "帮我写E2E测试"
  - "创建e2e配置"
  - "write e2e tests"
  - "add test cases"
  - "set up e2e"
  - "create e2e.yaml"
  - "配置Mock服务"
  - "跑一下E2E测试"
  - "运行端到端测试"
  - "run e2e tests"
  - "test the service"
  - "检查服务是否正常"
  - "验证接口"
  - "e2e.yaml"
  - "诊断测试失败"
  - "查看测试历史"
---

# ArgusAI E2E Testing

Docker-based end-to-end testing platform. **All operations go through the
argusai MCP server** (`npx argusai-mcp`), not the `argusai` CLI binary.

## Two Invocation Modes

### Mode 1: Claude Code session (interactive)

MCP tools are available directly. Call them by name:

```
argus_setup(projectPath: "/path/to/project")
argus_run(projectPath: "/path/to/project", filter: "smoke")
argus_clean(projectPath: "/path/to/project")
```

No shell commands needed.

### Mode 2: Bash scripts (self-improve.sh, CI pipelines)

Use `mcp2cli` to call the same MCP tools from shell:

```bash
MCP="mcp2cli --mcp-stdio 'npx argusai-mcp'"

# Setup environment
$MCP argus_setup '{"projectPath": "/path/to/project"}'

# Run a specific suite
$MCP argus_run '{"projectPath": "/path/to/project", "filter": "smoke"}'

# Clean up
$MCP argus_clean '{"projectPath": "/path/to/project"}'
```

`mcp2cli` binary is at `~/.local/bin/mcp2cli`.

## Available MCP Tools

| Tool | Purpose |
|------|---------|
| `argus_init` | Generate project template (e2e.yaml + tests/) |
| `argus_build` | Build Docker image |
| `argus_setup` | Start environment (network + mocks + container) |
| `argus_run` | Execute test suites (`filter` for specific suite) |
| `argus_run_suite` | Run a single named suite |
| `argus_status` | Check container/network/port status |
| `argus_logs` | View container logs |
| `argus_clean` | Stop containers + remove networks |
| `argus_dev` | One-step: build + setup |
| `argus_rebuild` | One-step: clean + build + setup |
| `argus_history` | View past test runs |
| `argus_trends` | Metric trends (pass-rate, duration, flaky) |
| `argus_flaky` | Detect unstable tests |
| `argus_diagnose` | Classify failure + suggest fix |
| `argus_preflight_check` | Pre-run health check |
| `argus_resources` | List all managed Docker resources |
| `argus_mock_requests` | View recorded mock requests |
| `argus_mock_generate` | Generate mock config from OpenAPI spec |

## Worktree Isolation (multi-worktree E2E)

Each git worktree can run its own isolated E2E environment by setting
`isolation.namespace` in `e2e.yaml`. The MCP server uses this to create
unique container names and networks per namespace:

```yaml
# e2e/e2e.yaml
isolation:
  namespace: "{{env.WORKTREE_ID}}"   # injected by self-improve.sh
```

In `self-improve.sh`, set `WORKTREE_ID` to the worktree short ID before
calling `argus_setup`. Containers will be named `<service>-<namespace>`
and networks `argusai-<namespace>-network`, preventing collisions between
parallel worktree runs.

## Intent Routing

| 你想做的事 | 读哪个文档 |
|----------|---------|
| 写 e2e.yaml、写测试用例、配置 mock | [references/authoring-guide.md](references/authoring-guide.md) |
| 使用 `recursive-session`/`recursive-cost`/`llm-judge` 等 plugin 断言 | [references/plugin-assertions.md](references/plugin-assertions.md) |
| 开发自定义断言 Plugin | [references/plugin-development.md](references/plugin-development.md) |
| 在 bash 脚本里用 mcp2cli 跑测试 | 本文档 Mode 2 部分 |

Full field reference: [references/e2e-yaml-full-reference.yaml](references/e2e-yaml-full-reference.yaml)

## Full Cycle (bash script example)

```bash
MCP="mcp2cli --mcp-stdio 'npx argusai-mcp'"
PROJECT="/path/to/project"

# Build image (skip if pre-built)
$MCP argus_build "{\"projectPath\": \"$PROJECT\"}"

# Start environment
$MCP argus_setup "{\"projectPath\": \"$PROJECT\"}"

# Run smoke suite
$MCP argus_run "{\"projectPath\": \"$PROJECT\", \"filter\": \"smoke\"}"

# Clean up
$MCP argus_clean "{\"projectPath\": \"$PROJECT\"}"
```

## Diagnose Failures

```bash
# In Claude Code:
argus_diagnose(projectPath: "/path/to/project", runId: "<run-id>", caseName: "failing case")
argus_flaky(projectPath: "/path/to/project")
argus_history(projectPath: "/path/to/project")
```
