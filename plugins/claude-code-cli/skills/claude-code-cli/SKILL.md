---
name: claude-code-cli
description: Guide for building and running AI agents using Claude Code CLI. Use when developing autonomous agents, multi-agent systems, CI/CD automation, or scripting Claude for programmatic tasks. Language-agnostic patterns for agent orchestration.
license: MIT
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
---

# Claude Code Agent Development Guide

Build autonomous agents using Claude Code CLI. Language-agnostic patterns for agent orchestration, multi-agent systems, and programmatic Claude integration.

## Core Concepts

Claude Code can run as a **headless agent** via `-p` (print mode), accepting prompts and returning structured outputs. This enables:

- Autonomous task execution
- Multi-agent orchestration
- CI/CD integration
- Programmatic workflows
- Language-agnostic agent development

## Authentication

### Option 1: Anthropic API Key

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
```

### Option 2: Claude Code OAuth Token (Pro/Team subscription)

Generate a long-lived token using your Claude subscription (no API key needed):

```bash
# Interactive setup - generates OAuth token
claude setup-token
```

Use the token via environment variable:

```bash
export CLAUDE_CODE_OAUTH_TOKEN="your-oauth-token"
```

This is ideal for CI/CD and server deployments where you want to use your Claude subscription instead of API credits.

### CI/CD Authentication Example

```yaml
# GitHub Actions with OAuth token
env:
  CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}

# Or with API key
env:
  ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

## Running Claude as an Agent

### Basic Agent Invocation

```bash
# Single task agent
claude -p "Implement a REST API endpoint for user registration"

# Agent with file context
claude -p "Review and fix bugs in $(cat src/main.py)"

# Piped input
echo "Create unit tests for the auth module" | claude -p
```

### Autonomous Mode (Sandboxed)

```bash
# Full autonomy - no permission prompts
claude -p --dangerously-skip-permissions "Build and test the feature"

# With budget limit for safety
claude -p --dangerously-skip-permissions --max-budget-usd 10 "Refactor the codebase"
```

### Structured Agent Output

```bash
# JSON response
claude -p --output-format json "Analyze codebase structure"

# Enforce output schema
claude -p --output-format json \
  --json-schema '{
    "type": "object",
    "properties": {
      "status": {"type": "string", "enum": ["success", "failure"]},
      "files_changed": {"type": "array", "items": {"type": "string"}},
      "summary": {"type": "string"}
    },
    "required": ["status", "summary"]
  }' \
  "Fix all TypeScript errors in the project"
```

### Streaming Agent Output

```bash
# Real-time streaming for long tasks
claude -p --output-format stream-json "Implement the full feature"

# With partial messages for UI updates
claude -p --output-format stream-json --include-partial-messages "Complex task..."
```

## Agent Tool Configuration

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

## Multi-Agent Orchestration

### Sequential Agents (Bash)

```bash
#!/bin/bash
# Pipeline: Analyze -> Plan -> Implement -> Test

# Agent 1: Analyzer
analysis=$(claude -p --output-format json \
  "Analyze the codebase and identify improvement areas")

# Agent 2: Planner
plan=$(echo "$analysis" | claude -p --output-format json \
  "Create implementation plan based on this analysis")

# Agent 3: Implementer
claude -p --dangerously-skip-permissions \
  "Implement this plan: $plan"

# Agent 4: Tester
claude -p --allowed-tools "Bash(npm test:*) Read" \
  "Run tests and report results"
```

### Parallel Agents (Bash)

```bash
#!/bin/bash
# Run multiple agents concurrently

# Start agents in background
claude -p "Review src/auth/" > auth-review.txt &
claude -p "Review src/api/" > api-review.txt &
claude -p "Review src/db/" > db-review.txt &

# Wait for all
wait

# Consolidate results
claude -p "Summarize these reviews: $(cat *-review.txt)"
```

### Python Orchestration

```python
import subprocess
import json

def run_agent(prompt: str, tools: list[str] = None, schema: dict = None) -> dict:
    """Run Claude as an agent and return structured output."""
    cmd = ["claude", "-p", "--output-format", "json"]

    if tools:
        cmd.extend(["--tools", ",".join(tools)])

    if schema:
        cmd.extend(["--json-schema", json.dumps(schema)])

    cmd.append(prompt)

    result = subprocess.run(cmd, capture_output=True, text=True)
    return json.loads(result.stdout)

# Example: Multi-agent pipeline
def code_review_pipeline(files: list[str]):
    # Agent 1: Static analysis
    analysis = run_agent(
        f"Analyze these files for issues: {files}",
        tools=["Read", "Grep"],
        schema={"type": "object", "properties": {"issues": {"type": "array"}}}
    )

    # Agent 2: Fix issues
    if analysis.get("issues"):
        run_agent(
            f"Fix these issues: {json.dumps(analysis['issues'])}",
            tools=["Read", "Edit"]
        )

    return analysis
```

### Node.js Orchestration

```javascript
const { execSync, spawn } = require('child_process');

async function runAgent(prompt, options = {}) {
  const args = ['-p', '--output-format', 'json'];

  if (options.tools) {
    args.push('--tools', options.tools.join(','));
  }

  if (options.schema) {
    args.push('--json-schema', JSON.stringify(options.schema));
  }

  if (options.skipPermissions) {
    args.push('--dangerously-skip-permissions');
  }

  args.push(prompt);

  const result = execSync(`claude ${args.join(' ')}`, { encoding: 'utf-8' });
  return JSON.parse(result);
}

// Streaming agent for long tasks
function streamAgent(prompt, onChunk) {
  const proc = spawn('claude', ['-p', '--output-format', 'stream-json', prompt]);

  proc.stdout.on('data', (data) => {
    const lines = data.toString().split('\n').filter(Boolean);
    lines.forEach(line => onChunk(JSON.parse(line)));
  });

  return new Promise((resolve, reject) => {
    proc.on('close', resolve);
    proc.on('error', reject);
  });
}
```

## Agent Patterns

### Supervisor-Worker Pattern

```bash
#!/bin/bash
# Supervisor decomposes task, workers execute

# Supervisor: Break down task
subtasks=$(claude -p --output-format json \
  --json-schema '{"type":"object","properties":{"tasks":{"type":"array","items":{"type":"string"}}}}' \
  "Break this into subtasks: $TASK")

# Workers: Execute each subtask
echo "$subtasks" | jq -r '.tasks[]' | while read task; do
  claude -p --dangerously-skip-permissions "$task"
done
```

### Reflection Pattern

```bash
#!/bin/bash
# Agent reviews and improves its own output

# Initial attempt
result=$(claude -p "Implement user authentication")

# Self-review
review=$(claude -p --tools "Read" \
  "Review this implementation for security issues: $result")

# Improve based on review
claude -p --dangerously-skip-permissions \
  "Improve the implementation based on this review: $review"
```

### Specialist Agents

```bash
# Define specialist agents
claude --agents '{
  "architect": {
    "description": "System design expert",
    "prompt": "You are a software architect. Focus on scalability, maintainability, and clean architecture."
  },
  "security": {
    "description": "Security analyst",
    "prompt": "You are a security expert. Identify vulnerabilities and suggest fixes."
  },
  "performance": {
    "description": "Performance optimizer",
    "prompt": "You are a performance expert. Optimize for speed and resource efficiency."
  }
}'

# Use specialist agents
claude --agent architect -p "Design the API structure"
claude --agent security -p "Review auth implementation"
claude --agent performance -p "Optimize database queries"
```

## Session Management for Agents

### Persistent Agent Sessions

```bash
# Start session with specific ID
claude --session-id "agent-task-123" -p "Start implementing feature X"

# Continue same session
claude --session-id "agent-task-123" -c -p "Continue with the tests"

# Resume and fork (branch the conversation)
claude --resume "agent-task-123" --fork-session -p "Try alternative approach"
```

### Stateless Agents

```bash
# Disable session persistence for ephemeral agents
claude -p --no-session-persistence "One-off task"
```

## Project Configuration for Agents

### `.claude/settings.json`

```json
{
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

### `CLAUDE.md` - Agent Instructions

```markdown
# Agent Instructions

## Project Context
This is a TypeScript REST API using Express.

## Coding Standards
- Use async/await
- Add JSDoc comments
- Write tests for new features

## Commands
- `npm run build` - Compile TypeScript
- `npm test` - Run Jest tests
- `npm run lint` - ESLint check
```

## CI/CD Agent Integration

### GitHub Actions

```yaml
name: AI Code Review
on: [pull_request]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: AI Review
        env:
          # Use OAuth token (subscription-based) or API key
          CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          # ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          # Get changed files
          files=$(git diff --name-only origin/main)

          # Run review agent
          claude -p --dangerously-skip-permissions \
            --output-format json \
            --max-budget-usd 2 \
            "Review these changed files for issues: $files" \
            > review.json

          # Post results
          cat review.json
```

### Pre-commit Agent

```bash
#!/bin/bash
# .git/hooks/pre-commit

staged=$(git diff --cached --name-only)

result=$(claude -p --tools "Read" --output-format json \
  --json-schema '{"type":"object","properties":{"approved":{"type":"boolean"},"issues":{"type":"array"}}}' \
  "Check these staged files for issues: $staged")

approved=$(echo "$result" | jq -r '.approved')

if [ "$approved" != "true" ]; then
  echo "Issues found:"
  echo "$result" | jq -r '.issues[]'
  exit 1
fi
```

## MCP Servers for Agents

### Add Tools via MCP

```bash
# Add external tools for agents
claude mcp add --transport stdio github -- npx -y @anthropic/mcp-github
claude mcp add --transport http api https://api.example.com/mcp

# Use in agent
claude -p "Create a GitHub issue for this bug"
```

### Custom MCP Config

```bash
# Load specific MCP config for agent
claude -p --mcp-config ./agent-tools.json "Task with custom tools"

# Strict mode - only use specified MCP servers
claude -p --strict-mcp-config --mcp-config ./minimal-tools.json "Restricted task"
```

## Agent Debugging

```bash
# Debug mode for agent development
claude -p --debug "Task to debug"

# Filter debug output
claude -p --debug "api,tools" "Task"

# Verbose output
claude -p --verbose "Task"
```

## Best Practices

1. **Budget Limits**: Always set `--max-budget-usd` for autonomous agents
2. **Tool Restrictions**: Use minimal tool set with `--tools` or `--allowed-tools`
3. **Structured Output**: Use `--json-schema` for reliable parsing
4. **Sandboxing**: Only use `--dangerously-skip-permissions` in isolated environments
5. **Session Management**: Use `--session-id` for traceable agent runs
6. **Error Handling**: Parse JSON output and check for errors
7. **Timeouts**: Implement timeouts in your orchestration layer
