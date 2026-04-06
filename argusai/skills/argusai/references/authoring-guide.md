# ArgusAI Test Authoring Guide

Analyze a project and author complete E2E test configurations: `e2e.yaml` config + YAML test case files.

## Output Structure

```
project/
├── e2e.yaml              # Main config: how to build, run, and test the service
├── tests/
│   ├── health.yaml       # Health check (always create first)
│   ├── api-crud.yaml     # Core API tests
│   ├── auth.yaml         # Auth-related tests (if applicable)
│   └── errors.yaml       # Error/edge case tests
└── .env.example          # Environment variable template
```

## Phase 1: Analyze the Project

### Step 1: Identify the Service Type

| Check | Files to Read | What to Learn |
|-------|---------------|---------------|
| Language/Framework | `package.json`, `requirements.txt`, `go.mod`, `Cargo.toml` | Runtime, framework, dependencies |
| Dockerfile | `Dockerfile`, `Dockerfile.*` | Build process, exposed port, entrypoint |
| API Routes | Route definitions (see table below) | Endpoints, methods, request/response shapes |
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

| Pattern | Likely Dependency |
|---------|------------------|
| `axios`, `fetch`, `got`, `httpx` calls to external URLs | External API — needs Mock |
| `redis`, `ioredis` imports | Redis — configure in environment |
| `mysql`, `pg`, `prisma`, `sequelize` | Database — use test DB or embedded |
| `amqplib`, `kafka` | Message queue — may need mock or skip |
| `aws-sdk`, `@aws-sdk/*` | AWS services — needs Mock |

### Step 4: Find the Health Check Endpoint

Look for: `/health`, `/healthz`, `/livez`, `/readyz`, `/api/health`, `/api/status`, `/ping`.
If none exists, use the simplest GET endpoint that returns 200.

## Phase 2: Write e2e.yaml

### Decision Tree

```
Has Dockerfile?
├── Yes → Use service.build.dockerfile
└── No → Help user create a Dockerfile first

Has health endpoint?
├── Yes → Configure healthcheck.path
└── No → Omit healthcheck (use delay in setup instead)

Has external API dependencies?
├── Yes → Has OpenAPI spec?
│         ├── Yes → Use openapi mock mode
│         └── No → Configure manual mocks section
└── No → Skip mocks

Containers managed externally (e.g. docker-compose)?
├── Yes → Test-only mode: omit service, set resilience.preflight.enabled: false
└── No → Define service as normal
```

### Config Generation Rules

**service.build:**
```yaml
service:
  build:
    dockerfile: <path to Dockerfile>
    context: .
    image: <project-name>:e2e
```

**service.container:**
```yaml
  container:
    name: <project-name>-e2e
    ports:
      - "<host-port>:<container-port>"
    environment:
      NODE_ENV: test
      DATABASE_URL: "{{env.DATABASE_URL}}"
      EXTERNAL_API_URL: http://host.docker.internal:9081  # Point to mock
    healthcheck:
      path: /health
      interval: 5s
      timeout: 3s
      retries: 10
      startPeriod: 15s   # Adjust per service type
```

**Port selection:**
- Container port: from Dockerfile `EXPOSE` or framework defaults
- Host port: start from 8080 (8080, 8081, 8082...)
- Mock ports: start from 9081 (9081, 9082, 9083...)

**Startup time guidelines:**

| Service Type | Suggested startPeriod |
|-------------|----------------------|
| Node.js/Python lightweight | 10-15s |
| Node.js/Python with DB init | 20-30s |
| Java/Spring Boot | 30-60s |
| Service with migrations | 45-90s |

**service.vars:**
```yaml
  vars:
    base_url: http://localhost:<host-port>
```

### History Config (optional, recommended)

```yaml
history:
  enabled: true
  storage: local       # local (SQLite) | memory
  retention: 90d
  flakyWindow: 10
```

### Isolation Config (optional, multi-project)

```yaml
isolation:
  namespace: my-project
  portRange: [10000, 10099]
```

### Test-Only Mode (externally managed containers)

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

Use full URLs in test files (`url: http://localhost:3000/health`).
Workflow: `argusai run` only (skip build/setup).

## Phase 3: Mock Services

For each external dependency identified in Phase 1 Step 3:

### Mock Design Checklist

1. What endpoints does our service call?
2. What request format does it send?
3. What response does our service expect?
4. What error scenarios matter?

### Manual Route Configuration

```yaml
mocks:
  payment-gateway:
    port: 9081
    routes:
      # Happy path
      - method: POST
        path: /api/charge
        response:
          status: 200
          body:
            transaction_id: "{{uuid}}"
            status: "success"

      # Error scenario
      - method: POST
        path: /api/charge
        when:
          body:
            amount: 0
        response:
          status: 400
          body: { error: "Invalid amount" }
```

### Conditional Routing with `when`

Use `when` to return different responses based on request headers, body, or query:

```yaml
mocks:
  tunely-server:
    port: 9090
    routes:
      # Correct API key — success
      - method: POST
        path: /api/tunnels
        when:
          headers:
            x-api-key: "valid-key-123"
        response:
          status: 200
          body: { token: "tun_mock_123", domain: "test.tunnel.local" }

      # Wrong or missing API key — 401
      - method: POST
        path: /api/tunnels
        response:
          status: 401
          body: { detail: "Invalid API key" }
```

Routes are evaluated top-to-bottom; the first matching route wins. Place specific `when` routes before generic fallbacks.

`when` supports: `when.headers`, `when.body`, `when.query`.

### OpenAPI-Driven Mock

When the external dependency has an OpenAPI 3.0/3.1 spec:

```yaml
mocks:
  api-service:
    port: 9082
    openapi: ./specs/api.yaml
    mode: auto               # auto | record | replay | smart
    validate: true           # Request schema validation (422 on error)
    target: http://real-api  # Required only for record mode
    overrides:               # Manual overrides (highest priority)
      - method: POST
        path: /api/special
        response:
          status: 200
          body: { custom: true }
```

| Mode | Use When |
|------|----------|
| `auto` | Default — generates responses from schema examples/types |
| `record` | First time — proxies to real API and records responses |
| `replay` | CI/offline — replays previously recorded responses |
| `smart` | Best of both — replays if recorded, auto-generates otherwise |

### Mock as Verification Tool (`/_mock/requests`)

Mocks record every incoming request. Use the `/_mock/requests` endpoint to verify what your service actually sent to downstream dependencies:

```yaml
cases:
  # Step 1: Trigger the action that calls the downstream mock
  - name: "Create tunnel via API (proxied to mock)"
    request:
      method: POST
      url: "{{config.base_url}}/api/tunnel/create-tunnel"
      headers:
        Authorization: "Bearer {{runtime.login_token}}"
        Content-Type: application/json
      body:
        name: "test-tunnel"
        serverUrl: "http://host.docker.internal:9090"
        accessToken: "valid-key-123"
    expect:
      status: 200
      body:
        success: true

  # Step 2: Query mock's request log to verify headers
  - name: "Verify mock received auth headers"
    request:
      method: GET
      url: "http://localhost:9090/_mock/requests"
    expect:
      status: 200
      body:
        requests:
          length: { gte: 1 }

  # Step 3: Clean up for next test
  - name: "Clear mock request log"
    request:
      method: DELETE
      url: "http://localhost:9090/_mock/requests"
    expect:
      status: 200
```

**Mock diagnostic endpoints:**

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/_mock/requests` | GET | List all recorded requests (method, url, headers, timestamp) |
| `/_mock/requests` | DELETE | Clear the request log |
| `/_mock/health` | GET | Mock service health check |
| `/_mock/routes` | GET | List configured routes |

**URL rule**: Test runner → mock uses `localhost:<port>`, container → mock uses `host.docker.internal:<port>`.

### Response Authoring Tips

- Only include fields the service actually reads from the response
- `{{request.params.id}}` — echo back request parameters
- `{{uuid}}` — generated IDs in creation endpoints
- `{{timestamp}}` — time fields
- `delay: "2s"` — simulate slow responses for timeout testing

## Phase 4: Write Test Cases

### Test Suite Organization

| File | Content | Priority |
|------|---------|----------|
| `tests/health.yaml` | Health check + basic connectivity | **Always create** |
| `tests/crud.yaml` | Core CRUD operations | High |
| `tests/auth.yaml` | Authentication/authorization | High (if applicable) |
| `tests/business.yaml` | Key business flows | Medium |
| `tests/errors.yaml` | Error handling + edge cases | Medium |

### Rule 1: Always start with health check

```yaml
name: Health Check
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

### Rule 2: Test CRUD in lifecycle order

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
    expect:
      status: 201
      body:
        id: { exists: true }
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

  - name: "Delete resource"
    request:
      method: DELETE
      path: "/api/resources/{{runtime.resource_id}}"
    expect:
      status: [200, 204]

  - name: "Verify deleted"
    request:
      method: GET
      path: "/api/resources/{{runtime.resource_id}}"
    expect:
      status: 404

teardown:
  - name: "Cleanup"
    request:
      method: DELETE
      path: "/api/resources/{{runtime.resource_id}}"
    ignoreError: true
```

### Rule 3: Use `save` to chain dependent cases

```yaml
cases:
  - name: "Login"
    request:
      method: POST
      path: /api/auth/login
      body: { username: "testuser", password: "testpass" }
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

### Rule 4: Test error scenarios

```yaml
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

  - name: "Unauthorized access returns 401"
    request:
      method: GET
      path: /api/protected
    expect:
      status: 401
```

### Rule 5: Add teardown for data cleanup

Any test that creates data should have a teardown:

```yaml
teardown:
  - name: "Delete test data"
    request:
      method: DELETE
      path: "/api/resources/{{runtime.resource_id}}"
    ignoreError: true
```

## Assertion Operators

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
| Every item | `field: { every: {cond} }` | `users: { every: { email: { exists: true } } }` |
| Some item | `field: { some: {cond} }` | `users: { some: { role: "admin" } }` |
| Negation | `field: { not: val }` | `code: { not: { in: [500, 502] } }` |
| Response time | `responseTime: { lt: N }` | `responseTime: { lt: 500 }` |

## Advanced Patterns

### Expression Assertions (CEL-like)

```yaml
    expect:
      expr:
        - "body.items.length <= body.pageSize"
        - "body.total >= body.items.length"
        - "body.price > 0 && body.price <= body.maxPrice"
```

Supports: `==`, `!=`, `>`, `>=`, `<`, `<=`, `&&`, `||`, dot-paths, `.length`.

### Compound Assertions (AND/OR)

```yaml
    expect:
      all:
        - name: { exists: true }
        - email: { type: string }
      any:
        - status: "active"
        - status: "pending"
```

### Container Exec

```yaml
  - name: "Verify database record"
    exec:
      command: "sqlite3 /app/db.sqlite 'SELECT COUNT(*) FROM users'"
    expect:
      exitCode: 0
      output:
        contains: "1"
```

Output assertions: `contains`, `notContains`, `matches` (regex), `json` (parse & assert), `length` (line count).

### File Assertion

```yaml
  - name: "Config file is valid"
    file:
      path: /app/config.json
      exists: true
      json:
        port: { gt: 0 }
      permissions: "-rw-r--r--"
```

### Process Assertion

```yaml
  - name: "App process is running"
    process:
      name: node
      running: true
      count: ">=1"
```

### Port Assertion

```yaml
  - name: "Service port is open"
    port:
      port: 3000
      listening: true
```

## Phase 5: Register & Finalize

### Register test suites in e2e.yaml

```yaml
tests:
  suites:
    - name: Health Check
      id: health
      file: tests/health.yaml

    - name: CRUD Tests
      id: crud
      file: tests/crud.yaml
```

Suite ID: short, lowercase, hyphenated (`health`, `crud`, `auth`). Used in `argusai run -s <id>`.

### Create .env.example

```bash
# ArgusAI E2E Test Environment Variables
DATABASE_URL=postgres://testuser:testpass@host.docker.internal:5432/testdb
API_KEY=test-api-key
JWT_SECRET=e2e-test-secret
```

## Checklist

- [ ] e2e.yaml has correct Dockerfile path and build context
- [ ] Container port matches the service's actual listen port
- [ ] Health check path matches an actual endpoint
- [ ] All external dependencies have mock configs (OpenAPI or manual)
- [ ] Container environment points external URLs to mock ports
- [ ] Each test suite is registered in tests.suites
- [ ] Health test is always the first suite
- [ ] Tests with side effects have teardown sections
- [ ] No hardcoded values that should be variables
- [ ] .env.example lists all `{{env.VAR}}` references
