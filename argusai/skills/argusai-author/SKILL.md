---
skill_name: argusai-author
version: 0.1.0
description: >
  Write E2E test configurations and test cases for ArgusAI. Use this skill when users
  ask to create e2e.yaml, write E2E tests, add test cases, set up mock services,
  or when you need to analyze a project and generate its E2E testing configuration.
  Triggers include "帮我写E2E测试", "创建e2e配置", "write e2e tests",
  "add test cases", "set up e2e", "create e2e.yaml", "配置Mock服务",
  or when a project has a Dockerfile but no e2e.yaml.
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
---

# ArgusAI Test Authoring Skill

This skill teaches you how to analyze a project and author complete E2E test configurations for ArgusAI. The output is an `e2e.yaml` config file and YAML test case files.

## Overview: What You Need to Create

For any project, a complete ArgusAI E2E test setup consists of:

```
project/
├── e2e.yaml              ← Main config: how to build, run, and test the service
├── tests/
│   ├── health.yaml       ← Health check test (always create this first)
│   ├── api-crud.yaml     ← Core API tests
│   ├── auth.yaml         ← Auth-related tests (if applicable)
│   └── errors.yaml       ← Error/edge case tests
└── .env.example          ← Environment variable template
```

## Phase 1: Analyze the Project

Before writing any config, you MUST understand the project. Investigate these in order:

### Step 1: Identify the Service Type

Read the project's key files to determine:

| Check | Files to Read | What to Learn |
|-------|---------------|---------------|
| Language/Framework | `package.json`, `requirements.txt`, `go.mod`, `Cargo.toml`, `pom.xml` | Runtime, framework, dependencies |
| Dockerfile | `Dockerfile`, `Dockerfile.*` | Build process, exposed port, entrypoint |
| API Routes | Route definitions (see framework table below) | Available endpoints, methods, request/response shapes |
| Config | `.env`, `.env.example`, `config/` | Required environment variables |
| Dependencies | Code imports, docker-compose.yml | External services to mock |

### Step 2: Find API Routes by Framework

| Framework | Where to Find Routes |
|-----------|---------------------|
| Express.js | `app.get/post/put/delete(...)` or `router.*` in `routes/`, `controllers/` |
| Fastify | `fastify.get/post(...)` or route plugins in `routes/` |
| NestJS | `@Get()/@Post()` decorators in `*.controller.ts` |
| Koa | `router.get/post(...)` in route files |
| FastAPI | `@app.get/post(...)` or `@router.*` in route files |
| Flask | `@app.route(...)` or `@blueprint.route(...)` |
| Django | `urlpatterns` in `urls.py` |
| Gin (Go) | `r.GET/POST(...)` in router setup |
| Spring Boot | `@GetMapping/@PostMapping` in controllers |

### Step 3: Identify External Dependencies

Look for HTTP clients, database connections, message queues, etc. These need to be mocked:

| Pattern | Likely Dependency |
|---------|------------------|
| `axios`, `fetch`, `got`, `httpx` calls to external URLs | External API → needs Mock |
| `redis`, `ioredis` imports | Redis → configure in environment |
| `mysql`, `pg`, `prisma`, `sequelize` | Database → use test DB or embedded |
| `amqplib`, `kafka` | Message queue → may need mock or skip |
| `aws-sdk`, `@aws-sdk/*` | AWS services → needs Mock |

### Step 4: Find the Health Check Endpoint

Most services have one. Look for:
- `/health`, `/healthz`, `/livez`, `/readyz`
- `/api/health`, `/api/status`, `/ping`
- If none exists, use the simplest GET endpoint that returns 200

## Phase 2: Write e2e.yaml

### Template

Based on your analysis, generate the config. Here's the decision tree:

```
Has Dockerfile?
├── Yes → Use service.build.dockerfile
└── No → Help user create a Dockerfile first

Has health endpoint?
├── Yes → Configure healthcheck.path
└── No → Omit healthcheck (use delay in setup instead)

Has external API dependencies?
├── Yes → Has OpenAPI spec?
│         ├── Yes → Use openapi mock mode (auto-generates routes from spec)
│         └── No → Configure manual mocks section
└── No → Skip mocks

Uses environment variables?
├── Yes → Map them in container.environment with {{env.VAR}}
└── No → Set minimal env (NODE_ENV: test, etc.)

Want test history persistence?
├── Yes → Add history config (default: SQLite local storage)
└── No → Omit (history disabled by default)

Containers managed externally (e.g. docker-compose)?
├── Yes → Use test-only mode: omit service/services, set resilience.preflight.enabled: false
└── No → Define service/services as normal

Multiple projects on the same machine?
├── Yes → Optionally add isolation.namespace / isolation.portRange to avoid conflicts
└── No → Default project-scoped naming (argusai-<slug>-network) already handles it
```

### Config Generation Rules

**service.build:**
```yaml
service:
  build:
    dockerfile: <path to Dockerfile, relative to project root>
    context: .
    image: <project-name>:e2e      # Use project name + :e2e tag
```

**service.container:**
```yaml
  container:
    name: <project-name>-e2e       # Use project name + -e2e suffix
    ports:
      - "<available-host-port>:<container-port>"
    environment:
      # Always set test mode
      NODE_ENV: test                # or FLASK_ENV, RAILS_ENV, etc.
      # Map secrets from env
      DATABASE_URL: "{{env.DATABASE_URL}}"
      API_KEY: "{{env.API_KEY}}"
      # Point external services to mocks
      PAYMENT_API_URL: http://host.docker.internal:9081
```

**Port selection rules:**
- Container port: read from Dockerfile `EXPOSE` directive or framework defaults
- Host port: start from 8080, avoid conflicts (8080, 8081, 8082...)
- Mock ports: start from 9081 (9081, 9082, 9083...)

**healthcheck (if endpoint exists):**
```yaml
    healthcheck:
      path: /health               # The actual health endpoint path
      interval: 5s                # Fast for E2E
      timeout: 3s
      retries: 10
      startPeriod: 15s            # Adjust based on service startup time
```

Startup time guidelines:
| Service Type | Suggested startPeriod |
|-------------|----------------------|
| Node.js/Python lightweight | 10-15s |
| Node.js/Python with DB init | 20-30s |
| Java/Spring Boot | 30-60s |
| Service with migrations | 45-90s |

**service.vars:**
```yaml
  vars:
    base_url: http://localhost:<host-port>   # Always set this
    # Add any commonly used values in tests
```

### History Config (optional but recommended)

Enable test result persistence for flaky detection and trend analysis:

```yaml
history:
  enabled: true
  storage: local       # local (SQLite) | memory (CI/tests)
  retention: 90d       # How long to keep records
  flakyWindow: 10      # Sliding window size for flaky detection
```

### Isolation Config (optional, for multi-project environments)

Use when multiple projects run on the same machine to ensure Docker resource isolation:

```yaml
isolation:
  namespace: my-project     # Custom Docker resource prefix; default: derived from project.name
  portRange: [10000, 10099] # Port auto-assignment range; default: [9000, 9999]
```

If omitted, ArgusAI derives a namespace from `project.name` (e.g. "My App" → `argusai-my-app-network`).

### Test-Only Config (for externally managed containers)

When containers are started outside ArgusAI (e.g. `docker-compose up`), omit `service`/`services` entirely and disable preflight:

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

In this mode, test files use absolute URLs (`url: http://localhost:3000/health`) instead of relative paths.
The AI workflow becomes: `argus_init` → `argus_run` (no `argus_build` or `argus_setup` needed).

## Phase 3: Design Mock Services

For each external dependency identified in Phase 1 Step 3:

### Mock Design Checklist

1. **What endpoints does our service call?** — Read the HTTP client code
2. **What request format does it send?** — Check the request construction
3. **What response does our service expect?** — Check how the response is consumed
4. **What error scenarios matter?** — Check error handling code

### Mock Configuration Pattern

```yaml
mocks:
  <dependency-name>:               # e.g., payment-gateway, user-service
    port: <host-port>              # 9081, 9082, etc.
    containerPort: <container-port> # If container accesses via Docker network
    routes:
      # Happy path
      - method: <METHOD>
        path: <path our service calls>
        response:
          status: 200
          body:
            <fields our service reads from response>

      # Error scenario (optional but recommended)
      - method: <METHOD>
        path: <path>
        when:
          body:
            <condition for error trigger>
        response:
          status: 500
          body:
            error: "simulated failure"
```

### OpenAPI-Driven Mock (Preferred When Spec Exists)

If the external dependency has an OpenAPI 3.0/3.1 spec, use the auto-generation mode instead of manually writing routes:

```yaml
mocks:
  <dependency-name>:
    port: <host-port>
    openapi: <path-to-openapi-spec>   # YAML or JSON format
    mode: auto                         # auto | record | replay | smart
    validate: true                     # Optional: validate requests against schema (422 on error)
    target: <real-api-url>             # Required only for record mode
    overrides:                         # Optional: manual overrides (highest priority)
      - method: POST
        path: /api/special
        response:
          status: 200
          body: { custom: true }
```

**Mode selection guide:**

| Mode | Use When |
|------|----------|
| `auto` | Default — generates mock responses from schema examples/types |
| `record` | First time setup — proxies to real API and records responses |
| `replay` | CI/offline — replays previously recorded responses |
| `smart` | Best of both — replays if recorded, auto-generates otherwise |

**When to use OpenAPI mock vs manual mock:**
- Has OpenAPI spec → Use `openapi` mode (faster, validates requests, covers all endpoints)
- No spec / need precise control → Use manual `routes` configuration
- Both → Use `openapi` + `overrides` for specific routes that need custom behavior

### Mock Response Authoring Tips (Manual Mode)

- Only include fields that the service actually reads from the response
- Use `{{request.params.id}}` to echo back request parameters (useful for GET-by-ID)
- Use `{{uuid}}` for generated IDs in creation endpoints
- Use `{{timestamp}}` for time fields
- Add `delay: "2s"` to simulate slow responses for timeout testing

## Phase 4: Write Test Cases

### Test Suite Organization

Organize tests into files by domain:

| File | Content | Priority |
|------|---------|----------|
| `tests/health.yaml` | Health check + basic connectivity | **Always create** |
| `tests/crud.yaml` | Core CRUD operations | High |
| `tests/auth.yaml` | Authentication/authorization | High (if applicable) |
| `tests/business.yaml` | Key business flows | Medium |
| `tests/errors.yaml` | Error handling + edge cases | Medium |
| `tests/performance.yaml` | Timeout/slow response scenarios | Low |

### Test Case Writing Rules

**Rule 1: Always start with health check**

```yaml
name: 健康检查
sequential: true

setup:
  - waitHealthy:
      timeout: 60s

cases:
  - name: "Health endpoint returns 200"
    request:
      method: GET
      path: /health
    expect:
      status: 200
```

**Rule 2: Test CRUD in lifecycle order**

Create → Read → Update → Read again → Delete → Verify deleted

```yaml
name: Resource CRUD
sequential: true

setup:
  - waitHealthy:
      timeout: 60s

cases:
  - name: "Create resource"
    request:
      method: POST
      path: /api/resources
      headers:
        Content-Type: application/json
      body:
        name: "e2e-test-{{timestamp}}"
        type: "test"
    expect:
      status: 201
      body:
        id: { exists: true }
        name: { contains: "e2e-test" }
    save:
      resource_id: "id"

  - name: "Read created resource"
    request:
      method: GET
      path: "/api/resources/{{runtime.resource_id}}"
    expect:
      status: 200
      body:
        id: "{{runtime.resource_id}}"
        name: { contains: "e2e-test" }

  - name: "Update resource"
    request:
      method: PUT
      path: "/api/resources/{{runtime.resource_id}}"
      headers:
        Content-Type: application/json
      body:
        name: "updated-name"
    expect:
      status: 200
      body:
        name: "updated-name"

  - name: "Delete resource"
    request:
      method: DELETE
      path: "/api/resources/{{runtime.resource_id}}"
    expect:
      status: [200, 204]

  - name: "Verify resource is deleted"
    request:
      method: GET
      path: "/api/resources/{{runtime.resource_id}}"
    expect:
      status: 404

teardown:
  - name: "Cleanup (in case test failed midway)"
    request:
      method: DELETE
      path: "/api/resources/{{runtime.resource_id}}"
    ignoreError: true
```

**Rule 3: Use `save` to chain dependent cases**

When case B needs data from case A:
```yaml
cases:
  - name: "Login"
    request:
      method: POST
      path: /api/auth/login
      body:
        username: "testuser"
        password: "testpass"
    expect:
      status: 200
      body:
        token: { exists: true }
    save:
      auth_token: "token"

  - name: "Access protected endpoint"
    request:
      method: GET
      path: /api/protected
      headers:
        Authorization: "Bearer {{runtime.auth_token}}"
    expect:
      status: 200
```

**Rule 4: Test error scenarios**

```yaml
name: Error Handling
sequential: true

setup:
  - waitHealthy:
      timeout: 60s

cases:
  - name: "Missing required field returns 400"
    request:
      method: POST
      path: /api/resources
      headers:
        Content-Type: application/json
      body: {}
    expect:
      status: 400

  - name: "Non-existent resource returns 404"
    request:
      method: GET
      path: /api/resources/non-existent-id
    expect:
      status: 404

  - name: "Invalid method returns 405"
    request:
      method: PATCH
      path: /api/health
    expect:
      status: [404, 405]

  - name: "Unauthorized access returns 401"
    request:
      method: GET
      path: /api/protected
    expect:
      status: 401
```

**Rule 5: Use variables to avoid hardcoding**

```yaml
variables:
  test_prefix: "e2e"
  game_id: "{{test_prefix}}-{{timestamp}}"

cases:
  - name: "Create with unique ID"
    request:
      method: POST
      path: /api/games
      body:
        id: "{{config.game_id}}"
```

**Rule 6: Add teardown for data cleanup**

Any test that creates data should have a teardown:

```yaml
teardown:
  - name: "Delete test data"
    request:
      method: DELETE
      path: "/api/resources/{{runtime.resource_id}}"
    ignoreError: true
```

### Assertion Selection Guide

| What to Verify | Assertion to Use |
|---------------|-----------------|
| API returns correct status | `status: 200` |
| Response has required field | `field: { exists: true }` |
| Field has correct value | `field: "expected"` |
| Field is correct type | `field: { type: string }` |
| Numeric field in range | `field: { gt: 0, lte: 100 }` |
| String contains keyword | `field: { contains: "success" }` |
| Array has items | `items: { length: { gt: 0 } }` |
| Response matches pattern | `field: { matches: "^[a-z]+$" }` |
| Field is one of N values | `field: { in: [active, pending] }` |
| Cross-field logic | `expr: "body.items.length == body.total"` |
| Dynamic data (UUID/timestamp) | `field: { type: string, matches: "^[0-9a-f-]{36}$" }` |
| Conditional response shapes | `any: [shape1, shape2]` |
| String ends with suffix | `field: { endsWith: ".json" }` |
| String must NOT contain | `field: { notContains: "error" }` |
| Value must NOT equal | `field: { not: "forbidden_value" }` |
| Value must NOT be in set | `field: { not: { in: [500, 502] } }` |
| Every array item valid | `items: { every: { email: { exists: true } } }` |
| At least one item matches | `items: { some: { role: "admin" } }` |
| Response time SLA | `responseTime: { lt: 500 }` |

## Advanced Patterns

### Pattern 1: Expression Assertions (CEL-like)

For cross-field validation and complex logic, use `expr`:

```yaml
cases:
  - name: "Pagination response is consistent"
    request:
      method: GET
      path: /api/users?page=1&size=10
    expect:
      status: 200
      expr:
        - "body.items.length <= body.pageSize"
        - "body.total >= body.items.length"
        - "body.page == 1"

  - name: "Price calculation is correct"
    request:
      method: GET
      path: "/api/orders/{{runtime.order_id}}"
    expect:
      status: 200
      expr: "body.total == body.subtotal + body.tax"
```

`expr` supports: `==`, `!=`, `>`, `>=`, `<`, `<=`, `&&` (AND), `||` (OR), dot-path access (`body.user.name`), `.length` on arrays/strings.

### Pattern 2: Compound Assertions (AND/OR)

When the response shape depends on conditions:

```yaml
cases:
  - name: "Search returns results or empty with message"
    request:
      method: GET
      path: /api/search?q=test
    expect:
      status: 200
      any:
        - items: { length: { gt: 0 } }
        - message: { contains: "no results" }

  - name: "User profile has all required fields"
    request:
      method: GET
      path: "/api/users/{{runtime.user_id}}"
    expect:
      status: 200
      all:
        - name: { exists: true }
        - email: { type: string, matches: "@" }
        - role: { in: [admin, user, guest] }
```

### Pattern 3: Container Exec Steps

Run commands inside the container and assert output — useful for verifying side effects:

```yaml
cases:
  - name: "API call creates a log entry"
    request:
      method: POST
      path: /api/actions
      body: { action: "test" }
    expect:
      status: 200

  - name: "Verify log file was written"
    exec:
      command: "cat /app/logs/actions.log | tail -1"
    expect:
      exitCode: 0
      output:
        contains: "test"
        matches: "\\d{4}-\\d{2}-\\d{2}.*action.*test"

  - name: "Verify database record count"
    exec:
      command: "sqlite3 /app/data/db.sqlite 'SELECT COUNT(*) FROM actions'"
    expect:
      output:
        json:
          count: { gt: 0 }
```

### Pattern 4: File Assertions

Check files inside the container — config files, generated outputs, etc.:

```yaml
cases:
  - name: "Config file exists with correct settings"
    file:
      path: /app/config.json
      exists: true
      json:
        database_url: { exists: true }
        port: { gt: 0 }
        environment: "test"

  - name: "Log directory has correct permissions"
    file:
      path: /app/logs
      exists: true
      permissions: "drwxr-xr-x"
      owner: "appuser"

  - name: "No secrets in config"
    file:
      path: /app/config.json
      notContains:
        - "password"
        - "secret_key"
```

### Pattern 5: Process and Port Assertions

Verify the service internals are healthy:

```yaml
cases:
  - name: "Application process is running"
    process:
      name: node
      running: true
      count: ">=1"

  - name: "Service listens on expected port"
    port:
      port: 3000
      listening: true

  - name: "Debug port is NOT exposed"
    port:
      port: 9229
      listening: false
```

### Pattern 6: Handling Dynamic Data

For fields that change every time (IDs, timestamps, tokens):

```yaml
cases:
  - name: "Create returns dynamic fields"
    request:
      method: POST
      path: /api/resources
      body: { name: "test" }
    expect:
      status: 201
      body:
        id: { type: string, length: { gt: 0 } }
        name: "test"
        created_at: { type: string, matches: "^\\d{4}-\\d{2}-\\d{2}" }
        updated_at: { exists: true }
```

### Pattern 7: Array Traversal with `every` / `some`

Assert conditions on every or at least one array element:

```yaml
cases:
  - name: "All users have valid email"
    request:
      method: GET
      path: /api/users
    expect:
      status: 200
      body:
        users:
          every:
            email: { exists: true, matches: "@" }
            name: { type: string }
            role: { in: [admin, user, guest] }

  - name: "At least one admin in the list"
    request:
      method: GET
      path: /api/users
    expect:
      status: 200
      body:
        users:
          some:
            role: "admin"
```

`every` vacuously passes on empty arrays; `some` fails on empty arrays.

### Pattern 8: Negation with `not` and `notContains`

Assert that something is NOT the case:

```yaml
cases:
  - name: "Successful response has no error indicators"
    request:
      method: GET
      path: /api/status
    expect:
      status: 200
      body:
        status: { not: "error" }
        message: { notContains: "failed" }
        code: { not: { in: [500, 502, 503] } }
```

`not` wraps any assertion and inverts it. `notContains` is a shorthand for the most common negation.

### Pattern 9: Response Time SLA

Assert that the API responds within acceptable time:

```yaml
cases:
  - name: "Health check is fast"
    request:
      method: GET
      path: /health
    expect:
      status: 200
      responseTime: { lt: 200 }

  - name: "Search completes within SLA"
    request:
      method: GET
      path: /api/search?q=test
    expect:
      status: 200
      responseTime: { lt: 2000 }
```

`responseTime` supports: `lt`, `lte`, `gt`, `gte` (in milliseconds). A plain number `responseTime: 500` means "must complete within 500ms".

### When AI Should NOT Write Programmatic Assertions

These are the only scenarios that truly cannot be asserted with the DSL:

| Scenario | Why It Can't Be Programmatized | Workaround |
|----------|-------------------------------|------------|
| "Error message is user-friendly" | Requires semantic/linguistic judgment | Use `contains` for key phrases |
| "UI layout looks correct" | Visual assertion needs screenshots | Use Playwright runner with visual tests |
| "Business logic is correct" | Requires domain knowledge beyond data | Use `expr` for mathematical validation |

## Phase 5: Register Test Suites

Add all test files to e2e.yaml:

```yaml
tests:
  suites:
    - name: 健康检查
      id: health
      file: tests/health.yaml

    - name: CRUD 测试
      id: crud
      file: tests/crud.yaml

    - name: 错误处理
      id: errors
      file: tests/errors.yaml
```

Suite ID naming: use short, lowercase, hyphenated names (`health`, `crud`, `auth`, `error-handling`). These are used in `argus_run -s <id>` for targeted execution.

## Phase 6: Create .env.example

List all environment variables the test configuration references:

```bash
# ArgusAI E2E Test Environment Variables
# Copy to .env and fill in actual values

# Database (if service needs one)
DATABASE_URL=postgres://testuser:testpass@host.docker.internal:5432/testdb

# External API keys (if needed)
API_KEY=test-api-key

# Other secrets
JWT_SECRET=e2e-test-secret
```

## Complete Example: Node.js API Project

Given a project with:
- Express.js API on port 3000
- Endpoints: `/health`, `/api/users` (CRUD), `/api/auth/login`
- External dependency: payment service at `PAYMENT_API_URL`
- Database: PostgreSQL at `DATABASE_URL`

**Generated e2e.yaml:**

```yaml
version: "1"

project:
  name: my-api
  description: "My API E2E Tests"

service:
  build:
    dockerfile: Dockerfile
    context: .
    image: my-api:e2e

  container:
    name: my-api-e2e
    ports:
      - "8080:3000"
    environment:
      NODE_ENV: test
      DATABASE_URL: "{{env.DATABASE_URL}}"
      JWT_SECRET: "{{env.JWT_SECRET}}"
      PAYMENT_API_URL: http://host.docker.internal:9081
    healthcheck:
      path: /health
      interval: 5s
      timeout: 3s
      retries: 10
      startPeriod: 15s

  vars:
    base_url: http://localhost:8080

history:
  enabled: true
  storage: local
  retention: 90d
  flakyWindow: 10

mocks:
  payment-service:
    port: 9081
    routes:
      - method: POST
        path: /api/charge
        response:
          status: 200
          body:
            transaction_id: "{{uuid}}"
            status: "success"
            amount: "{{request.body.amount}}"

      - method: GET
        path: /api/transactions/:id
        response:
          status: 200
          body:
            id: "{{request.params.id}}"
            status: "completed"

tests:
  suites:
    - name: 健康检查
      id: health
      file: tests/health.yaml

    - name: 用户 CRUD
      id: users
      file: tests/users.yaml

    - name: 认证流程
      id: auth
      file: tests/auth.yaml

    - name: 支付流程
      id: payment
      file: tests/payment.yaml

    - name: 错误处理
      id: errors
      file: tests/errors.yaml
```

## Checklist Before Done

- [ ] e2e.yaml has correct Dockerfile path and build context
- [ ] Container port matches the service's actual listen port
- [ ] Health check path matches an actual endpoint
- [ ] All external dependencies have mock configs (OpenAPI-driven or manual)
- [ ] OpenAPI mocks: spec file path is correct and accessible
- [ ] Container environment points external URLs to mock ports
- [ ] Each test suite is registered in tests.suites
- [ ] Health test is always the first suite
- [ ] Tests with side effects have teardown sections
- [ ] No hardcoded values that should be variables
- [ ] .env.example lists all {{env.VAR}} references
- [ ] History config added if persistence is desired (recommended)
