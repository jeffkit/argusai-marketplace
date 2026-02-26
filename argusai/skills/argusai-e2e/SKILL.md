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

You have 11 MCP tools available through the ArgusAI MCP server:

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
| `argus_preflight_check` | Check environment health | Proactively verify Docker, disk, orphan resources; use `autoFix: true` to auto-clean |
| `argus_reset_circuit` | Reset circuit breaker | When Docker recovers after failures, reset from open → half-open |

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
2. **View container logs** — `argus_logs(projectPath, container, lines: 200)` to see application errors
3. **Check mock requests** — `argus_mock_requests(projectPath)` to verify the service sent correct requests to dependencies
4. **Check environment status** — `argus_status(projectPath)` to verify containers are healthy

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

mocks:                     # Optional mock services
  mock-name:
    port: 9081
    routes:
      - method: GET
        path: /api/endpoint
        response:
          status: 200
          body: { key: "value" }

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

## Best Practices

1. **Always init first** — `argus_init` must be called before any other tool
2. **Check status before re-running** — use `argus_status` to see if environment is already up
3. **Clean up after testing** — always call `argus_clean` when done
4. **Use targeted runs** — when debugging, use `argus_run_suite` for faster feedback
5. **Read logs on failure** — container logs usually reveal the root cause
6. **projectPath must be absolute** — always use the full absolute path
7. **Use `exec` to verify side effects** — after API calls, use exec to check DB/files/logs
8. **Use `expr` for cross-field logic** — validates consistency within a response
9. **Use `file`/`process`/`port` for infrastructure checks** — verify container state beyond HTTP
