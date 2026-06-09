# ArgusAI Runner Guide — via MCP

All ArgusAI operations use the **MCP server** (`npx argusai-mcp`), not the CLI binary.

## In Claude Code (interactive)

MCP tools are available directly by name. No shell commands needed.

### Full Test Cycle

```
argus_setup(projectPath: "/path/to/project")
argus_run(projectPath: "/path/to/project", filter: "smoke")
argus_clean(projectPath: "/path/to/project")
```

### Quick Start (build + setup in one call)

```
argus_dev(projectPath: "/path/to/project")
```

### Run Specific Suite

```
argus_run(projectPath: "/path/to/project", filter: "basic")
argus_run(projectPath: "/path/to/project", filter: "basic,session")
```

### Check Status / Logs

```
argus_status(projectPath: "/path/to/project")
argus_logs(projectPath: "/path/to/project")
```

### Clean Up

```
argus_clean(projectPath: "/path/to/project")
```

---

## In Bash Scripts (self-improve.sh, CI)

Use `mcp2cli` (`~/.local/bin/mcp2cli`) to call the same MCP tools from shell.

```bash
MCP="mcp2cli --mcp-stdio 'npx argusai-mcp'"
PROJECT="/path/to/project"

# Full cycle
$MCP argus_setup "{\"projectPath\": \"$PROJECT\"}"
$MCP argus_run "{\"projectPath\": \"$PROJECT\", \"filter\": \"smoke\"}"
$MCP argus_clean "{\"projectPath\": \"$PROJECT\"}"
```

### Worktree Isolation

Set `isolation.namespace` in `e2e.yaml` to isolate parallel worktree runs:

```yaml
# e2e/e2e.yaml
isolation:
  namespace: "{{env.WORKTREE_ID}}"
```

Then in the bash script:
```bash
export WORKTREE_ID="wt-$(git rev-parse --short HEAD)"
$MCP argus_setup "{\"projectPath\": \"$PROJECT\"}"
```

Each worktree gets its own Docker container (`<service>-<namespace>`)
and network (`argusai-<namespace>-network`), so parallel runs don't collide.

---

## Diagnostics

```
# In Claude Code:
argus_diagnose(projectPath: "/path/to/project", runId: "<id>", caseName: "<name>")
argus_flaky(projectPath: "/path/to/project")
argus_history(projectPath: "/path/to/project")
argus_trends(projectPath: "/path/to/project")

# In bash:
$MCP argus_diagnose "{\"projectPath\": \"$PROJECT\", \"runId\": \"<id>\", \"caseName\": \"<name>\"}"
$MCP argus_flaky "{\"projectPath\": \"$PROJECT\"}"
$MCP argus_history "{\"projectPath\": \"$PROJECT\"}"
```

## Mock Inspection

```
argus_mock_requests(projectPath: "/path/to/project")
argus_mock_requests(projectPath: "/path/to/project", mock: "aimock")
```

## Environment Health

```
argus_preflight_check(projectPath: "/path/to/project")
argus_resources(projectPath: "/path/to/project")
```
