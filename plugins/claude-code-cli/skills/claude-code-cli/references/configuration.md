# Configuration Reference

Project configuration, tool restrictions, MCP servers, and CI/CD integration.

## Table of Contents

1. [Tool Configuration](#tool-configuration)
2. [Project Configuration](#project-configuration)
3. [MCP Servers](#mcp-servers)
4. [CI/CD Integration](#cicd-integration)
5. [Debugging](#debugging)

## Tool Configuration

### Restrict Agent Capabilities

```bash
# Read-only agent
claude -p --tools "Read,Glob,Grep" "Analyze the codebase"

# Code review agent (no writes)
claude -p --allowed-tools "Read Grep Glob" "Review PR changes"

# Build agent (specific commands only)
claude -p --allowed-tools "Bash(npm:*) Bash(git:*) Read Edit" "Build and commit"

# Deny dangerous operations
claude -p --disallowed-tools "Bash(rm:*) Bash(sudo:*)" "Clean up project"

# No tools at all (pure reasoning)
claude -p --tools "" "Explain this architecture"
```

### Tool Pattern Syntax

```bash
# Exact tool name
--allowed-tools "Read"

# Bash with command prefix
--allowed-tools "Bash(npm:*)"      # Any npm command
--allowed-tools "Bash(git log:*)"  # Only git log

# Multiple tools
--allowed-tools "Read Edit Bash(npm:*)"
```

## Project Configuration

### Configuration Hierarchy

1. CLI flags (highest priority)
2. `.claude/settings.local.json` (gitignored)
3. `.claude/settings.json` (shared)
4. `~/.claude/settings.local.json`
5. `~/.claude/settings.json` (lowest priority)

### `.claude/settings.json`

Project-level settings (commit to git):

```json
{
  "model": "sonnet",
  "permissions": {
    "allow": [
      "Bash(npm:*)",
      "Bash(git:*)",
      "Bash(make:*)",
      "Edit",
      "Read",
      "Write"
    ],
    "deny": [
      "Bash(rm -rf:*)",
      "Bash(sudo:*)"
    ]
  },
  "mcpServers": {
    "project-tools": {
      "command": "npx",
      "args": ["-y", "my-mcp-server"]
    }
  }
}
```

### `.claude/settings.local.json`

Local overrides (add to .gitignore):

```json
{
  "permissions": {
    "allow": [
      "Bash(docker:*)"
    ]
  }
}
```

### `CLAUDE.md`

Project instructions automatically loaded:

```markdown
# Project Guidelines

## Build Commands
- `npm run build` - Compile TypeScript
- `npm test` - Run Jest tests
- `npm run lint` - ESLint check

## Code Style
- Use TypeScript strict mode
- Follow ESLint rules
- Add JSDoc comments

## Architecture
- Express.js REST API
- PostgreSQL database
- Redis for caching
```

## MCP Servers

### Add MCP Servers

```bash
# HTTP transport
claude mcp add --transport http sentry https://mcp.sentry.dev/mcp

# SSE transport
claude mcp add --transport sse server-name https://example.com/sse

# Stdio transport (local process)
claude mcp add --transport stdio my-server -- npx -y my-mcp-package

# With environment variables
claude mcp add -e API_KEY=xxx my-server -- npx my-package

# With headers
claude mcp add -H "Authorization: Bearer token" my-server https://api.example.com
```

### Scope Options

```bash
# Local (default) - current machine only
claude mcp add -s local my-server -- command

# User - all projects for current user
claude mcp add -s user my-server -- command

# Project - shared with team (stored in .mcp.json)
claude mcp add -s project my-server -- command
```

### Manage Servers

```bash
claude mcp list              # List all servers
claude mcp get my-server     # Show details
claude mcp remove my-server  # Remove server
```

### MCP Config in Agent

```bash
# Load specific MCP config
claude -p --mcp-config ./agent-tools.json "Task with custom tools"

# Strict mode - only use specified MCP servers
claude -p --strict-mcp-config --mcp-config ./minimal-tools.json "Restricted task"
```

## CI/CD Integration

### GitHub Actions

```yaml
name: AI Code Review
on: [pull_request]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code

      - name: AI Review
        env:
          # Use OAuth token (subscription) or API key
          CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          # ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          files=$(git diff --name-only origin/main)
          claude -p --dangerously-skip-permissions \
            --output-format json \
            --max-budget-usd 2 \
            "Review these files: $files" > review.json
          cat review.json
```

### Pre-commit Hook

```bash
#!/bin/bash
# .git/hooks/pre-commit

staged=$(git diff --cached --name-only)

result=$(claude -p --tools "Read" --output-format json \
  --json-schema '{"type":"object","properties":{"approved":{"type":"boolean"},"issues":{"type":"array"}}}' \
  "Check these staged files for issues: $staged")

approved=$(echo "$result" | jq -r '.[-1].structured_output.approved')

if [ "$approved" != "true" ]; then
  echo "Issues found:"
  echo "$result" | jq -r '.[-1].structured_output.issues[]'
  exit 1
fi
```

### Post-merge Hook

```bash
#!/bin/bash
# .git/hooks/post-merge

# Auto-generate changelog entry
claude -p --tools "Read,Write" \
  "Generate changelog entry for recent commits"
```

### GitLab CI

```yaml
ai-review:
  stage: review
  script:
    - npm install -g @anthropic-ai/claude-code
    - |
      claude -p --dangerously-skip-permissions \
        --output-format json \
        "Review MR changes" > review.json
  variables:
    CLAUDE_CODE_OAUTH_TOKEN: $CLAUDE_CODE_OAUTH_TOKEN
```

## Debugging

### Debug Modes

```bash
# Full debug output
claude -p --debug "Task to debug"

# Filter debug categories
claude -p --debug "api,tools" "Task"
claude -p --debug "!statsig,!file" "Task"  # Exclude categories

# Verbose output
claude -p --verbose "Task"
```

### Common Debug Categories

| Category | Description |
|----------|-------------|
| `api` | API requests/responses |
| `tools` | Tool executions |
| `hooks` | Hook executions |
| `file` | File operations |
| `mcp` | MCP server communication |

### Health Check

```bash
# Check installation health
claude doctor

# Update Claude Code
claude update

# Check version
claude -v
```
