---
name: run-tests
description: Run ArgusAI E2E tests for the current project (full cycle or targeted suite)
argument-hint: "[suite-id]"
---

# ArgusAI E2E Test Runner

Execute the ArgusAI E2E testing workflow for the current project using the `argusai` CLI.

## Instructions

1. Determine the **project path** — use the current working directory as the project root
2. Check if `e2e.yaml` exists in the project directory. If not, inform the user that this project has no ArgusAI configuration and offer to help create one
3. If an argument (suite ID) was provided, run only that suite; otherwise run all suites

## Execution Steps

### Step 1: Preflight Check

```bash
argusai preflight --auto-fix
```

Verify Docker is running and environment is clean.

### Step 2: Build & Setup

```bash
argusai build && argusai setup
```

If Docker build fails with proxy errors, strip proxy env vars:

```bash
env -u http_proxy -u https_proxy -u all_proxy \
  -u HTTP_PROXY -u HTTPS_PROXY -u ALL_PROXY \
  docker build -t <image>:e2e .
argusai setup --skip-build
```

### Step 3: Run Tests

```bash
# Run all suites
argusai run

# Run specific suite
argusai run -s <suite-id>
```

### Step 4: Report Results

- Summarize test results clearly: total suites, passed, failed
- For any failures, show the failing test case name and assertion details
- If there are failures, check container logs: `argusai logs -n 200`

### Step 5: Clean Up

Ask the user if they want to clean up the environment, or keep it running for further testing.

```bash
argusai clean
```

## Error Handling

- If `argusai build` fails: show the error and suggest checking the Dockerfile
- If `argusai setup` fails with health check timeout: show container logs and suggest checking the service startup
- If tests fail: show detailed failure information and offer diagnostic steps

### Common Error Recovery

| Error | Recovery |
|-------|----------|
| `DOCKER_UNAVAILABLE` | Start Docker |
| `PORT_CONFLICT` | `argusai preflight --auto-fix` or set `portConflictStrategy: auto` |
| `CIRCUIT_OPEN` | Fix Docker issue, then `argusai reset-circuit` |
| `CONTAINER_RESTART_EXHAUSTED` | `argusai logs` to check crash reason |
| `DISK_SPACE_LOW` | `docker system prune` |
| `DNS_RESOLUTION_FAILED` | Verify Docker network config |
