# ArgusAI Runner & Diagnostics Guide

Run E2E tests, diagnose failures, and analyze trends via the `argusai` CLI.

## CLI Command Index

### Core Lifecycle

| Command | Purpose |
|---------|---------|
| `argusai build` | Build Docker image from e2e.yaml |
| `argusai setup` | Start environment (network + mocks + container + health) |
| `argusai run` | Execute test suites (`-s <id>` to filter) |
| `argusai status` | Check container/network/port status |
| `argusai logs` | View container logs (`-n <lines>`, `-f` to follow) |
| `argusai clean` | Stop containers + remove networks |

### Quick Commands

| Command | Purpose |
|---------|---------|
| `argusai dev` | One-step: build + setup, prints access URL |
| `argusai rebuild` | One-step: clean + build + setup |

### History & Diagnostics

| Command | Purpose |
|---------|---------|
| `argusai history` | View past test runs (`-n`, `--status`, `--days`) |
| `argusai trends` | Metric trends (`-m pass-rate\|duration\|flaky`) |
| `argusai flaky` | Detect unstable tests (ranked by score) |
| `argusai compare` | Diff two runs (`--base`, `--target`) |
| `argusai diagnose` | Classify failure + suggest fix (`--run`, `--case`) |
| `argusai report-fix` | Record fix to improve knowledge base |
| `argusai patterns` | Browse known failure patterns |

### Mock Management

| Command | Purpose |
|---------|---------|
| `argusai mock-requests` | View recorded mock requests (method, url, headers) |
| `argusai mock-requests --mock <name>` | View specific mock's requests |
| `argusai mock-requests --clear` | Clear all recorded requests |
| `argusai mock-generate --spec <path>` | Generate `mocks:` YAML from OpenAPI 3.x spec |
| `argusai mock-generate --spec <path> --port 9090` | Generate with custom port |
| `argusai mock-validate --mock <name>` | Check mock route coverage vs OpenAPI spec |

### Environment

| Command | Purpose |
|---------|---------|
| `argusai preflight` | Pre-run health check (`--auto-fix` to clean orphans) |
| `argusai resources` | List all ArgusAI-managed Docker resources |
| `argusai reset-circuit` | Reset Docker circuit breaker |

## Workflows

### Full Test Cycle

```bash
argusai build && argusai setup && argusai run && argusai clean
```

### Quick Dev Start

```bash
argusai dev              # build + setup, prints URL
argusai run              # run tests (re-run as needed)
argusai clean            # when done
```

### Run Specific Suite

```bash
argusai run -s health          # single suite
argusai run -s health,crud     # multiple suites (comma-separated)
```

### Pre-built Image (skip build)

When the Docker image is already built (e.g., from CI or manual build):

```bash
argusai setup --skip-build --no-dashboard
argusai run -s <suite-id>
argusai clean
```

### Docker Proxy Issues

If `docker build` fails with `proxyconnect tcp: dial tcp ... connection refused`, clear proxy env vars:

```bash
env -u http_proxy -u https_proxy -u all_proxy \
  -u HTTP_PROXY -u HTTPS_PROXY -u ALL_PROXY \
  docker build -t myimage:e2e .
```

Then use `argusai setup --skip-build` with the pre-built image.

## Failure Diagnosis

When tests fail, follow this sequence:

### Step 1: Check run result

The `argusai run` output includes per-suite/per-case results with assertion details.

### Step 2: Diagnose with AI

```bash
argusai diagnose --run <run-id> --case "case name"
```

Returns: failure category, known pattern match, suggested fix, confidence score.

| Confidence | Action |
|-----------|--------|
| > 0.7 | Follow the suggested fix directly |
| 0.3 - 0.7 | Consider suggestion but investigate further |
| < 0.3 | Investigate manually using logs and status |

### Step 3: Check if flaky

```bash
argusai flaky
```

| Level | Flaky Score | Meaning |
|-------|------------|---------|
| STABLE | 0 | Always passes |
| MOSTLY_STABLE | 0 - 0.2 | Rarely fails |
| FLAKY | 0.2 - 0.5 | Unstable |
| VERY_FLAKY | 0.5 - 1.0 | Very unstable |
| BROKEN | 1.0 | Always fails (real bug) |

### Step 4: Check logs

```bash
argusai logs -n 200
```

### Step 5: Check mock requests

```bash
argusai mock-requests
```

### Step 6: Report fix

After fixing, report back to improve the knowledge base:

```bash
argusai report-fix --run <id> --case "name" --fix "what was fixed"
```

## History & Trends

```bash
argusai history -n 10                              # Recent runs
argusai trends -m pass-rate -d 7                   # Pass rate over 7 days
argusai compare --base <old-id> --target <new-id>  # Diff two runs
```

## Common Failure Patterns

| Symptom | Likely Cause | Action |
|---------|-------------|--------|
| `HEALTH_CHECK_TIMEOUT` | Service didn't start in time | Check logs, increase `startPeriod` |
| Status code mismatch | API behavior changed | Review endpoint implementation |
| Body assertion failed | Response structure changed | Update test expectations or fix code |
| `CONNECTION_REFUSED` | Container not running | `argusai status`, re-run `argusai setup` |
| Mock not recording | Wrong URL in container config | Verify `environment` in e2e.yaml |
| `DOCKER_UNAVAILABLE` | Docker daemon not running | Start Docker |
| `PORT_CONFLICT` | Port already in use | Free port or set `portConflictStrategy: auto` |
| `CIRCUIT_OPEN` | Repeated Docker failures | Fix Docker, then `argusai reset-circuit` |
| `CONTAINER_RESTART_EXHAUSTED` | Container keeps crashing | Check logs for OOM, missing config |
| `DNS_RESOLUTION_FAILED` | Mock not reachable | Verify containers on same Docker network |
| `DISK_SPACE_LOW` | Insufficient disk space | `docker system prune` |

## Fix Environment Issues

```bash
argusai preflight --auto-fix    # Diagnose + auto-clean orphans
argusai resources               # See all managed Docker resources
argusai reset-circuit           # Reset after Docker recovery
```

## Best Practices

1. **Check status before re-running** — `argusai status` to see if environment is up
2. **Use `dev` for iteration** — simpler than manual build + setup
3. **Always clean up** — `argusai clean` when done
4. **Enable history** — `history.enabled: true` in e2e.yaml for diagnostics
5. **Diagnose before investigating** — `argusai diagnose` may already know the fix
6. **Report fixes** — `argusai report-fix` improves diagnosis over time
7. **Use `--reporter json`** — for machine-readable results
8. **Exit codes** — 0 = all passed, 1 = failures

## Output Parsing

```bash
argusai run --reporter json     # Machine-readable JSON output
argusai run --reporter html --output report.html  # HTML report
```
