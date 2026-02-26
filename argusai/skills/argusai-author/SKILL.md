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
├── Yes → Configure mocks section
└── No → Skip mocks

Uses environment variables?
├── Yes → Map them in container.environment with {{env.VAR}}
└── No → Set minimal env (NODE_ENV: test, etc.)
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

### Mock Response Authoring Tips

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
- [ ] All external dependencies have mock configs
- [ ] Container environment points external URLs to mock ports
- [ ] Each test suite is registered in tests.suites
- [ ] Health test is always the first suite
- [ ] Tests with side effects have teardown sections
- [ ] No hardcoded values that should be variables
- [ ] .env.example lists all {{env.VAR}} references
