---
skill_name: argusai-e2e
version: 0.1.0
description: >
  Run Docker-based E2E tests using ArgusAI MCP tools. Use this skill when users
  ask to run end-to-end tests, verify API behavior, test services in Docker containers,
  set up mock services, or perform acceptance testing. Triggers include phrases like
  "跑一下E2E测试", "运行端到端测试", "run e2e tests", "verify the API",
  "test the service", "run acceptance tests", "检查服务是否正常",
  or when the current project contains an e2e.yaml file.
triggers:
  - "跑一下E2E测试"
  - "运行端到端测试"
  - "run e2e tests"
  - "run tests"
  - "test the service"
  - "run acceptance tests"
  - "检查服务是否正常"
  - "验证接口"
  - "e2e.yaml"
---

# ArgusAI E2E Testing Skill

You have access to ArgusAI MCP tools for running Docker-based end-to-end tests. This skill teaches you how to use these tools effectively.

## Prerequisites

Before using ArgusAI tools, verify:
1. **Docker** is running (`docker info` should succeed)
2. **e2e.yaml** exists in the project directory
3. **argusai-mcp** package is available (installed via plugin MCP config)

## Available MCP Tools

You have 21 MCP tools available through the ArgusAI MCP server:

### Core Test Lifecycle (9 tools)

| Tool | Purpose | When to Use |
|------|---------|-------------|
| `argus_init` | Load project config | **Always call first** — loads e2e.yaml and creates a session |
| `argus_build` | Build Docker images | After code changes, or first time setup (circuit breaker protected) |
| `argus_setup` | Start test environment | After build — runs preflight, port resolution, orphan cleanup, network verification |
| `argus_run` | Run all/filtered test suites | Main test execution |
| `argus_run_suite` | Run a single test suite | For targeted testing |
| `argus_status` | Check environment status | To verify containers are running |
| `argus_logs` | View container logs | When tests fail, to diagnose issues |
| `argus_clean` | Clean up resources | After testing is complete |
| `argus_mock_requests` | View mock request recordings | To verify outbound requests |

### Resilience & Self-Healing (2 tools)

| Tool | Purpose | When to Use |
|------|---------|-------------|
| `argus_preflight_check` | Check environment health | Proactively verify Docker, disk, orphan resources; use `autoFix: true` to auto-clean |
| `argus_reset_circuit` | Reset circuit breaker | When Docker recovers after failures, reset from open → half-open |

### History & Trend Analysis (4 tools)

| Tool | Purpose | When to Use |
|------|---------|-------------|
| `argus_history` | Query historical test runs | View recent test execution records with filtering/pagination |
| `argus_trends` | Get trend data | View pass-rate, duration, or flaky metrics over time |
| `argus_flaky` | List flaky tests | Identify unstable tests ranked by flaky score |
| `argus_compare` | Compare two runs | Diff two test runs to spot regressions or improvements |

### Intelligent Diagnostics (3 tools)

| Tool | Purpose | When to Use |
|------|---------|-------------|
| `argus_diagnose` | Diagnose a failure | Classify failure + match knowledge base + get fix suggestions |
| `argus_report_fix` | Report a fix | After fixing a failure, report back to improve knowledge base confidence |
| `argus_patterns` | Browse failure patterns | View/search known failure patterns in the knowledge base |

### OpenAPI Smart Mock (2 tools)

| Tool | Purpose | When to Use |
|------|---------|-------------|
| `argus_mock_generate` | Generate mock config | Generate Mock YAML config from an OpenAPI spec file |
| `argus_mock_validate` | Validate mock coverage | Check if current mock config covers all OpenAPI spec endpoints |

### Multi-Project Isolation (1 tool)

| Tool | Purpose | When to Use |
|------|---------|-------------|
| `argus_resources` | Global resource view | List all ArgusAI-managed containers/networks/ports grouped by project; debug port conflicts or orphan resources |

## Standard Workflow

### Full Test Cycle

Follow this sequence for a complete E2E test run:

```
1. argus_init(projectPath)     → Load configuration
2. argus_build(projectPath)    → Build Docker image (circuit breaker protected)
3. argus_setup(projectPath)    → Start environment (preflight + port resolve + orphan cleanup + mocks + containers + network verify)
4. argus_run(projectPath)      → Execute all test suites
5. argus_clean(projectPath)    → Clean up everything
```

### Quick Re-test (Environment Already Running)

If the environment is already running (check with `argus_status`):

```
1. argus_init(projectPath)     → Reload config
2. argus_run(projectPath)      → Run tests
```

### Targeted Testing

To run a specific test suite:

```
argus_run_suite(projectPath, suiteId: "health")
```

Or filter multiple suites:

```
argus_run(projectPath, filter: "health,api")
```

## Tool Parameters Reference

### argus_init
```
projectPath: string (required) — Absolute path to the project directory containing e2e.yaml
configFile?: string — Config filename override (default: "e2e.yaml")
```

### argus_build
```
projectPath: string (required) — Project path (must have active session from init)
noCache?: boolean — Disable Docker layer cache for clean rebuild
service?: string — Build specific service in multi-service mode
```

### argus_setup
```
projectPath: string (required) — Project path (must have built images)
timeout?: string — Health check timeout override, e.g. "120s"
```

### argus_run
```
projectPath: string (required) — Project path (must have running environment)
filter?: string — Suite ID filter (comma-separated for multiple, e.g. "health,api")
parallel?: boolean — Override parallel execution setting
```

### argus_run_suite
```
projectPath: string (required) — Project path
suiteId: string (required) — Suite identifier to run
```

### argus_status
```
projectPath: string (required) — Project path
```

### argus_logs
```
projectPath: string (required) — Project path
container: string (required) — Container name
lines?: number — Number of tail lines (default: 100)
since?: string — Show logs since timestamp, e.g. "5m", "2h"
```

### argus_clean
```
projectPath: string (required) — Project path
force?: boolean — Force remove stuck containers
```

### argus_mock_requests
```
projectPath: string (required) — Project path
mockName?: string — Specific mock name (default: all mocks)
since?: string — Filter requests after timestamp
clear?: boolean — Clear request log after reading
```

### argus_preflight_check
```
projectPath: string (required) — Project path
skipDiskCheck?: boolean — Skip disk space check
skipOrphanCheck?: boolean — Skip orphan resource detection
autoFix?: boolean — Automatically clean up orphan resources
```

### argus_reset_circuit
```
projectPath: string (required) — Project path
```

### argus_history
```
projectPath: string (required) — Project path
limit?: number — Max records to return (default: 20)
status?: string — Filter by status: "pass" | "fail" | "error"
since?: string — Filter records after this ISO-8601 date
```

### argus_trends
```
projectPath: string (required) — Project path
metric: string (required) — Metric type: "pass-rate" | "duration" | "flaky"
days?: number — Number of days to look back (default: 30)
suiteId?: string — Filter by suite ID
```

### argus_flaky
```
projectPath: string (required) — Project path
topN?: number — Number of top flaky tests to return (default: 10)
threshold?: number — Minimum flaky score threshold (default: 0)
suiteId?: string — Filter by suite ID
```

### argus_compare
```
projectPath: string (required) — Project path
runId1: string (required) — First run ID
runId2: string (required) — Second run ID
```

### argus_diagnose
```
projectPath: string (required) — Project path
runId: string (required) — The test run ID containing the failure
caseName: string (required) — The failed test case name
suiteId?: string — Suite ID for more precise matching
```

### argus_report_fix
```
projectPath: string (required) — Project path
runId: string (required) — The run ID of the original failure
caseName: string (required) — The test case that was fixed
fixDescription: string (required) — Description of what was done to fix it
suiteId?: string — Suite ID
```

### argus_patterns
```
projectPath: string (required) — Project path
category?: string — Filter by failure category
source?: string — Filter by source: "built-in" | "learned"
limit?: number — Max patterns to return
```

### argus_mock_generate
```
projectPath: string (required) — Project path
specPath: string (required) — Path to OpenAPI spec file (YAML or JSON)
mockName?: string — Name for the generated mock service
port?: number — Port for the mock service
```

### argus_mock_validate
```
projectPath: string (required) — Project path
mockName: string (required) — Name of the mock service to validate
specPath?: string — Path to OpenAPI spec (overrides the one in config)
```

### argus_resources
```
(no parameters) — Returns all ArgusAI-managed Docker resources across all projects
```

## Response Format

All tools return a JSON envelope:

```json
{
  "success": true,
  "data": { ... },
  "timestamp": 1234567890
}
```

On error:
```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable message",
    "details": { ... }
  },
  "timestamp": 1234567890
}
```

## Failure Diagnosis Workflow

When tests fail, follow this diagnostic sequence:

1. **Check the run result** — the `argus_run` response includes per-suite and per-case results with assertion details
2. **Use intelligent diagnostics** — `argus_diagnose(projectPath, runId, caseName)` to automatically classify the failure, match known patterns, and get fix suggestions with confidence scores
3. **Check flaky status** — `argus_flaky(projectPath)` to see if the failing test is known to be unstable
4. **View container logs** — `argus_logs(projectPath, container, lines: 200)` to see application errors
5. **Check mock requests** — `argus_mock_requests(projectPath)` to verify the service sent correct requests to dependencies
6. **Check environment status** — `argus_status(projectPath)` to verify containers are healthy
7. **After fixing** — `argus_report_fix(projectPath, runId, caseName, fixDescription)` to improve the knowledge base

### Intelligent Diagnostics Details

The `argus_diagnose` tool returns:
- **Failure category**: One of 10 categories (ASSERTION_MISMATCH, HTTP_ERROR, TIMEOUT, CONNECTION_REFUSED, CONTAINER_OOM, CONTAINER_CRASH, MOCK_MISMATCH, CONFIG_ERROR, NETWORK_ERROR, UNKNOWN)
- **Known pattern match**: If matched, includes pattern description, suggested fix, confidence score (0-1), and similar historical resolutions
- **Flaky info**: Whether this test case is known to be unstable (with flaky score and stability level)

Use the confidence score to decide actions:
- **confidence > 0.7**: Follow the suggested fix directly
- **confidence 0.3-0.7**: Consider the suggestion but investigate further
- **confidence < 0.3 or no match**: Investigate manually using logs and status

### History & Trend Analysis

Use history tools to make data-driven decisions:

```
argus_history(projectPath, limit: 10)                    → Recent test runs
argus_trends(projectPath, metric: "pass-rate", days: 7)  → Pass rate trend
argus_flaky(projectPath, topN: 5)                        → Top 5 flaky tests
argus_compare(projectPath, runId1, runId2)               → Diff two runs
```

Flaky test stability levels:
| Level | Flaky Score | Meaning |
|-------|------------|---------|
| STABLE | 0 | Always passes |
| MOSTLY_STABLE | 0 - 0.2 | Rarely fails |
| FLAKY | 0.2 - 0.5 | Unstable |
| VERY_FLAKY | 0.5 - 1.0 | Very unstable |
| BROKEN | 1.0 | Always fails (real bug) |

### Resilience & Recovery Workflow

When the environment itself is problematic:

1. **Run preflight check** — `argus_preflight_check(projectPath)` to diagnose Docker daemon, disk space, and orphan resources
2. **Auto-fix issues** — `argus_preflight_check(projectPath, autoFix: true)` to automatically clean orphan resources
3. **Reset circuit breaker** — if builds/setups fail with `CIRCUIT_OPEN`, call `argus_reset_circuit(projectPath)` after resolving the underlying issue
4. **Check error codes** — all errors include structured codes (e.g., `DOCKER_UNAVAILABLE`, `PORT_CONFLICT`, `CONTAINER_RESTART_EXHAUSTED`) with `suggestedActions` for AI-guided recovery

### Common Failure Patterns

| Symptom | Likely Cause | Action |
|---------|-------------|--------|
| `HEALTH_CHECK_TIMEOUT` | Service didn't start in time | Check logs, increase `startPeriod` |
| Status code mismatch | API behavior changed | Review the endpoint implementation |
| Body assertion failed | Response structure changed | Update test expectations or fix code |
| `CONNECTION_REFUSED` | Container not running | Run `argus_status`, maybe re-run `argus_setup` |
| Mock not recording requests | Wrong URL in container config | Verify `environment` in e2e.yaml |
| `DOCKER_UNAVAILABLE` | Docker daemon not running | Start Docker Desktop/daemon |
| `PORT_CONFLICT` | Port already in use | Set `resilience.network.portConflictStrategy: auto` or free the port |
| `CIRCUIT_OPEN` | Repeated Docker failures | Fix Docker issue, then `argus_reset_circuit` |
| `CONTAINER_RESTART_EXHAUSTED` | Container keeps crashing | Check logs for root cause (OOM, missing config, etc.) |
| `DNS_RESOLUTION_FAILED` | Mock not reachable in network | Verify containers are on the same Docker network |
| `DISK_SPACE_LOW` | Insufficient disk space | Free disk space, `docker system prune` |

## e2e.yaml Quick Reference

The e2e.yaml config file defines:

```yaml
version: "1"
project:
  name: string             # Project name
service:
  build:
    dockerfile: string     # Dockerfile path
    image: string          # Image name
  container:
    name: string           # Container name
    ports: ["host:container"]
    environment: {}        # Env vars (supports {{env.VAR}} templates)
    healthcheck:
      path: /health
      interval: 10s
      timeout: 5s
      retries: 10
      startPeriod: 30s
  vars: {}                 # Custom vars ({{config.xxx}})

history:                   # Optional test persistence
  enabled: true
  storage: local           # local (SQLite) | memory
  retention: 90d
  flakyWindow: 10          # Sliding window for flaky detection

mocks:                     # Optional mock services
  mock-name:
    port: 9081
    routes:
      - method: GET
        path: /api/endpoint
        response:
          status: 200
          body: { key: "value" }

  # OpenAPI-driven mock (auto-generates routes from spec)
  api-mock:
    port: 9082
    openapi: ./specs/api.yaml   # OpenAPI 3.0/3.1 spec
    mode: auto                   # auto | record | replay | smart
    validate: true               # Request schema validation (422 on error)
    target: http://real-api:8080 # Proxy target for record mode
    overrides:                   # Manual route overrides (highest priority)
      - method: POST
        path: /api/override
        response:
          status: 200
          body: { custom: true }

tests:
  suites:
    - name: string
      id: string           # Used in argus_run filter
      file: string         # Path to test YAML file
      runner: yaml         # yaml | vitest | pytest | shell | exec | playwright
```

## Test Step Types (5 types)

### Type 1: HTTP Request (most common)

```yaml
cases:
  - name: "Test case"
    request:
      method: GET
      path: /api/endpoint
      headers: { Authorization: "Bearer {{config.token}}" }
      body: { key: "value" }
      timeout: 30s
    expect:
      status: 200
      body:
        field: "exact_value"
        count: { gt: 0 }
        token: { exists: true }
    save:
      my_var: "data.id"
```

### Type 2: Container Exec

Run a command inside the Docker container:

```yaml
  - name: "Check database record"
    exec:
      command: "sqlite3 /app/db.sqlite 'SELECT COUNT(*) FROM users'"
    expect:
      exitCode: 0
      output:
        contains: "1"
```

Output assertions: `contains`, `notContains`, `matches` (regex), `json` (parse & assert), `length` (line count).

### Type 3: File Assertion

Check files inside the container:

```yaml
  - name: "Config file is valid"
    file:
      path: /app/config.json
      exists: true
      json:
        port: { gt: 0 }
      permissions: "-rw-r--r--"
      owner: "appuser"
```

### Type 4: Process Assertion

Check running processes:

```yaml
  - name: "App process is running"
    process:
      name: node
      running: true
      count: ">=1"
      user: "appuser"
```

### Type 5: Port Assertion

Check port listening:

```yaml
  - name: "Service port is open"
    port:
      port: 3000
      listening: true
```

## Body Assertion Operators

| Operator | Syntax | Example |
|----------|--------|---------|
| Exact match | `field: value` | `name: "hello"` |
| Type check | `field: { type: T }` | `id: { type: string }` |
| Existence | `field: { exists: bool }` | `token: { exists: true }` |
| Enum | `field: { in: [...] }` | `status: { in: [active, pending] }` |
| Greater than | `field: { gt: N }` | `count: { gt: 0 }` |
| Greater or equal | `field: { gte: N }` | `age: { gte: 18 }` |
| Less than | `field: { lt: N }` | `score: { lt: 100 }` |
| Less or equal | `field: { lte: N }` | `retries: { lte: 3 }` |
| Contains | `field: { contains: S }` | `msg: { contains: "ok" }` |
| Not contains | `field: { notContains: S }` | `msg: { notContains: "error" }` |
| Starts with | `field: { startsWith: S }` | `url: { startsWith: "https" }` |
| Ends with | `field: { endsWith: S }` | `file: { endsWith: ".json" }` |
| Regex | `field: { matches: P }` | `id: { matches: "^[a-f0-9]+$" }` |
| Length | `field: { length: N }` | `items: { length: 5 }` |
| Length range | `field: { length: { gt: 0 } }` | `items: { length: { gt: 0, lte: 50 } }` |
| Every item | `field: { every: {conditions} }` | `users: { every: { email: { exists: true } } }` |
| Some item | `field: { some: {conditions} }` | `users: { some: { role: "admin" } }` |
| Negation | `field: { not: value_or_ops }` | `code: { not: { in: [500, 502] } }` |
| Response time | `responseTime: { lt: N }` | `responseTime: { lt: 500 }` |

### Expression Assertions (CEL-like)

For cross-field validation:

```yaml
    expect:
      expr:
        - "body.items.length == body.total"
        - "body.price > 0 && body.price <= body.maxPrice"
        - "status == 200 || status == 201"
```

Supports: `==`, `!=`, `>`, `>=`, `<`, `<=`, `&&`, `||`, dot-paths, `.length`.

### Compound Assertions

```yaml
    expect:
      all:                    # ALL conditions must pass (AND)
        - name: { exists: true }
        - email: { type: string }
      any:                    # ANY condition passes (OR)
        - status: "active"
        - status: "pending"
```

## Multi-Project Isolation

When multiple projects share the same machine or MCP Server:

- Each project gets its own Docker network: `argusai-<project-slug>-network` (derived from `project.name`)
- Process-level `PortAllocator` prevents port races during concurrent setup
- Use `isolation.namespace` to override the namespace prefix
- Use `isolation.portRange` to assign dedicated port ranges per project

```yaml
isolation:
  namespace: my-project     # Optional, default: derived from project.name
  portRange: [10000, 10099] # Optional, default: [9000, 9999]
```

Use `argus_resources` (no params) to see all running ArgusAI containers and networks:
```
argus_resources → { projects: [{ project, containers, networks, claimedPorts, session }] }
```

## Test-Only Mode (No Services)

When containers are managed externally (e.g. by `docker-compose`), omit `service`/`services` entirely and disable preflight:

```yaml
version: "1"
project:
  name: my-stack

resilience:
  preflight:
    enabled: false

tests:
  suites:
    - name: Health Check
      id: health
      file: tests/health.yaml
```

In test YAML files, use full URLs instead of relative paths:
```yaml
cases:
  - name: "Service is up"
    request:
      method: GET
      url: http://localhost:3000/health
    expect:
      status: 200
```

Workflow for test-only mode: `argus_init` → `argus_run` (skip `argus_build` and `argus_setup`).

## Best Practices

1. **Always init first** — `argus_init` must be called before any other tool
2. **Check status before re-running** — use `argus_status` to see if environment is already up
3. **Clean up after testing** — always call `argus_clean` when done
4. **Use targeted runs** — when debugging, use `argus_run_suite` for faster feedback
5. **Use `argus_diagnose` on failures** — get intelligent classification, fix suggestions, and confidence scores before manual investigation
6. **Check flaky status** — before investigating a failure, check `argus_flaky` to see if it's a known unstable test
7. **Report fixes** — after fixing a failure, call `argus_report_fix` to improve the knowledge base
8. **Read logs on failure** — container logs usually reveal the root cause
9. **projectPath must be absolute** — always use the full absolute path
10. **Use `exec` to verify side effects** — after API calls, use exec to check DB/files/logs
11. **Use `expr` for cross-field logic** — validates consistency within a response
12. **Use `file`/`process`/`port` for infrastructure checks** — verify container state beyond HTTP
