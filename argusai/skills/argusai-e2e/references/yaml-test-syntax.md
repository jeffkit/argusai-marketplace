# YAML Test File Complete Syntax Reference

## Test File Structure

```yaml
name: "Suite Name"              # Required — suite display name
description: "Description"      # Optional
sequential: true                # Execute cases in order (default: true)

variables:                      # Suite-level variables
  game_id: "test-{{timestamp}}"
  token: "my-token"

setup:                          # Pre-test steps (run before cases)
  - waitHealthy:
      timeout: 60s              # Wait for service to become healthy
  - delay: 3s                   # Simple delay
  - name: "Setup request"       # Named HTTP request
    request:
      method: POST
      path: /api/init
      body: { key: value }
    ignoreError: true           # Don't fail setup if this request fails

teardown:                       # Post-test cleanup (always runs)
  - name: "Cleanup"
    request:
      method: DELETE
      path: /api/cleanup
    ignoreError: true

cases:                          # Test cases (required)
  - name: "Case Name"          # Required — case display name
    delay: 2s                   # Wait before executing this case
    request:                    # HTTP request definition
      method: GET               # GET | POST | PUT | PATCH | DELETE
      path: /api/resource       # Request path
      headers:                  # Optional headers
        Authorization: "Bearer {{config.token}}"
        Content-Type: application/json
      body:                     # Optional request body (ignored for GET)
        key: value
      timeout: 30s              # Request timeout (optional)
    expect:                     # Assertions (optional)
      status: 200
      headers:
        content-type: application/json
      body:
        field: expected_value
    save:                       # Save response values (optional)
      my_id: "data.id"         # JSONPath-like extraction
      token: "headers.x-token"
```

## Variable System

| Template | Source | Example |
|----------|--------|---------|
| `{{config.xxx}}` | `service.vars` in e2e.yaml + `variables` in test file | `{{config.base_url}}` |
| `{{env.xxx}}` | Environment variables / `.env` file | `{{env.API_KEY}}` |
| `{{runtime.xxx}}` | Values saved via `save` in previous cases | `{{runtime.my_id}}` |
| `{{timestamp}}` | Current ISO-8601 timestamp | `2026-02-26T10:30:00.000Z` |
| `{{uuid}}` | Random UUID v4 | `a1b2c3d4-...` |

## Assertion DSL Complete Reference

### Status Code

```yaml
expect:
  status: 200                  # Exact match
  status: [200, 201]           # Any of these
```

### Body — Exact Match

```yaml
expect:
  body:
    name: "hello"              # String
    count: 42                  # Number
    active: true               # Boolean
    data: null                 # Null
```

### Body — Operators

```yaml
expect:
  body:
    # Type check
    name: { type: string }     # string | number | boolean | object | array | null

    # Existence
    token: { exists: true }    # Field exists (not null, not undefined)

    # Enum
    status: { in: [active, pending, disabled] }

    # Numeric comparison
    count:
      gt: 0                    # Greater than
      gte: 1                   # Greater than or equal
      lt: 100                  # Less than
      lte: 99                  # Less than or equal

    # String operations
    message:
      contains: "success"      # Contains substring
      startsWith: "OK"         # Starts with
      matches: "^\\d+$"        # Regex match

    # Length check
    items:
      length: 5                # Exact length
    items:
      length:                  # Range length
        gt: 0
        lte: 100
```

### Body — Shorthand DSL

```yaml
expect:
  body:
    token: $exists             # Same as { exists: true }
    status: $regex:^ok$        # Same as { matches: "^ok$" }
```

### Nested Object

```yaml
expect:
  body:
    user:
      name: "Alice"
      profile:
        age: { gt: 18 }
        email: { type: string }
```

### Header Assertions

```yaml
expect:
  headers:
    content-type: application/json    # Case-insensitive match
    x-request-id: { exists: true }
```

### Expression Assertions (CEL-like)

```yaml
expect:
  expr: "body.items.length == body.total"
  # Or multiple expressions (all must pass):
  expr:
    - "body.items.length <= body.pageSize"
    - "body.total >= 0"
    - "body.page == 1"
    - "body.price > 0 && body.price <= body.maxPrice"
    - "status == 200 || status == 201"
```

Supported: `==`, `!=`, `>`, `>=`, `<`, `<=`, `&&` (AND), `||` (OR).
Path access: `status`, `body.x.y.z`, `headers.content-type`, `body.items.length`.

### Compound Assertions

```yaml
expect:
  # AND — all conditions must pass
  all:
    - name: { exists: true }
    - email: { type: string }
    - role: { in: [admin, user] }

  # OR — any one condition passing is sufficient
  any:
    - status: "active"
    - status: "pending"
    - status: "processing"
```

## Non-HTTP Step Types

### Exec Step (run command in container)

```yaml
cases:
  - name: "Check database"
    exec:
      command: "psql -c 'SELECT COUNT(*) FROM users'"
      container: my-db-container     # Optional, defaults to main container
    expect:
      exitCode: 0
      output:
        contains: "1"               # String or array of strings
        notContains: "ERROR"         # String or array
        matches: "\\d+"             # Regex
        json:                        # Parse output as JSON and assert
          count: { gt: 0 }
        length: ">0"                # Line count: ">0", "==5", "<=10"
```

### File Step (assert container files)

```yaml
cases:
  - name: "Config file is correct"
    file:
      path: /app/config.json        # Container file path
      container: my-container        # Optional
      exists: true                   # File exists?
      contains: "database_url"       # String or array
      notContains: "password"        # String or array
      matches: "\"port\":\\s*\\d+"   # Regex on content
      json:                          # Parse as JSON and assert fields
        database_url: { exists: true }
        port: { gt: 0 }
      permissions: "-rw-r--r--"      # Unix permission string
      owner: "appuser"               # File owner
      size: ">0"                     # Size check: ">0", "<1024"
```

### Process Step (assert container processes)

```yaml
cases:
  - name: "Node process running"
    process:
      name: node                     # Process name to grep
      container: my-container        # Optional
      running: true                  # Is it running?
      count: ">=1"                   # Process count: "==1", ">=2"
      user: "appuser"               # Running as which user?
```

### Port Step (assert port listening)

```yaml
cases:
  - name: "API port is listening"
    port:
      port: 3000                     # Port number
      container: my-container        # Optional (checks inside container)
      host: localhost                # Optional (checks from host)
      listening: true                # Is port open?
      timeout: 5s                    # Connection timeout
```

## Time Formats

| Format | Example | Milliseconds |
|--------|---------|-------------|
| Milliseconds | `100ms` | 100 |
| Seconds | `5s` | 5000 |
| Minutes | `2m` | 120000 |
| Hours | `1h` | 3600000 |
| Number | `500` | 500 (treated as ms) |

## Mock Service Configuration (in e2e.yaml)

```yaml
mocks:
  service-name:
    port: 9081                 # Host port
    containerPort: 8081        # Container-accessible port (optional)
    routes:
      - method: GET
        path: /api/users/:id   # Supports route parameters
        response:
          status: 200
          delay: "2s"          # Optional response delay
          body:
            id: "{{request.params.id}}"
            name: "User {{request.params.id}}"
        when:                  # Optional conditional matching
          body:
            type: "create"
```

### Mock Template Variables

| Variable | Description |
|----------|-------------|
| `{{request.body.xxx}}` | Request body field |
| `{{request.params.xxx}}` | Route parameter |
| `{{request.query.xxx}}` | Query parameter |
| `{{timestamp}}` | ISO-8601 timestamp |
| `{{uuid}}` | Random UUID v4 |

### Mock Diagnostic Endpoints

Each mock service exposes:

| Endpoint | Description |
|----------|-------------|
| `GET /_mock/health` | Health check |
| `GET /_mock/requests` | View all recorded requests |
| `POST /_mock/requests/clear` | Clear recorded requests |
