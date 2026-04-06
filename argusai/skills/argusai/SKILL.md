---
skill_name: argusai
version: 1.0.0
description: >
  ArgusAI E2E testing — write test configurations, run Docker-based tests,
  and diagnose failures. This skill routes to the appropriate reference guide
  based on your intent: authoring tests or running/diagnosing them.
  Triggers include "帮我写E2E测试", "创建e2e配置", "跑一下E2E测试",
  "运行端到端测试", "write e2e tests", "run e2e tests", "配置Mock服务",
  "诊断测试失败", "argusai", "e2e.yaml".
triggers:
  - "帮我写E2E测试"
  - "创建e2e配置"
  - "write e2e tests"
  - "add test cases"
  - "set up e2e"
  - "create e2e.yaml"
  - "配置Mock服务"
  - "增加测试用例"
  - "写测试"
  - "跑一下E2E测试"
  - "运行端到端测试"
  - "run e2e tests"
  - "run tests"
  - "test the service"
  - "run acceptance tests"
  - "检查服务是否正常"
  - "验证接口"
  - "e2e.yaml"
  - "用CLI跑测试"
  - "argusai命令"
  - "argusai cli"
  - "命令行测试"
  - "诊断测试失败"
  - "查看测试历史"
---

# ArgusAI E2E Testing

Docker-based end-to-end testing platform. Uses `argusai` CLI for all operations.

## Prerequisites

- **Docker >= 24** running (`docker info`)
- **Node.js >= 20**
- `argusai` CLI installed (`npm install -g argusai-cli`)
- `e2e.yaml` in the project directory (for running tests)

## Intent Routing

Based on the user's request, read the appropriate reference guide:

### Scenario A: Write / Author Tests

**When**: User asks to create `e2e.yaml`, write test cases, add test suites, configure mock services, or analyze a project for E2E testing.

**Action**: Read [references/authoring-guide.md](references/authoring-guide.md) and follow the phased workflow inside.

Covers: project analysis, e2e.yaml generation, mock service configuration (manual routes, OpenAPI, verification via `/_mock/requests`), test case writing patterns, assertion syntax, advanced patterns (expr, exec, file, process, port).

### Scenario B: Run / Diagnose Tests

**When**: User asks to run E2E tests, check test results, diagnose failures, view test history or trends, or manage the Docker test environment.

**Action**: Read [references/runner-guide.md](references/runner-guide.md) and follow the workflows inside.

Covers: CLI command reference, full test cycle, quick dev workflow, failure diagnosis, history/trends/flaky analysis, common failure patterns, mock request inspection.

### Scenario C: Both (Write then Run)

**When**: User asks to write tests AND run them in the same session.

**Action**: Read authoring-guide.md first, complete the writing phase, then read runner-guide.md for execution.

## Additional References

- [references/e2e-yaml-full-reference.yaml](references/e2e-yaml-full-reference.yaml) — Complete e2e.yaml config reference with all fields annotated
- [references/playwright-browser-tests.yaml](references/playwright-browser-tests.yaml) — Playwright browser test integration guide (two modes + YAML browser steps)
