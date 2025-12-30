# Multi-Agent Orchestration Reference

Patterns for building multi-agent systems with Claude Code CLI.

## Table of Contents

1. [Sequential Agents](#sequential-agents)
2. [Parallel Agents](#parallel-agents)
3. [Python Orchestration](#python-orchestration)
4. [Node.js Orchestration](#nodejs-orchestration)
5. [Agent Patterns](#agent-patterns)
6. [Session Management](#session-management)

## Sequential Agents

Pipeline pattern - each agent's output feeds the next:

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

## Parallel Agents

Run multiple agents concurrently for independent tasks:

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

## Python Orchestration

### Basic Agent Runner

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
```

### Multi-Agent Pipeline

```python
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

### Streaming Agent

```python
import subprocess
import json

def stream_agent(prompt: str, on_token=None, on_complete=None):
    proc = subprocess.Popen(
        ["claude", "-p", "--output-format", "stream-json",
         "--include-partial-messages", prompt],
        stdout=subprocess.PIPE, text=True
    )

    for line in proc.stdout:
        if not line.strip():
            continue
        msg = json.loads(line)

        if msg["type"] == "stream_event":
            event = msg["event"]
            if event["type"] == "content_block_delta":
                if on_token:
                    on_token(event["delta"].get("text", ""))
        elif msg["type"] == "result":
            if on_complete:
                on_complete(msg)

    proc.wait()
```

### Parallel Execution

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

def parallel_review(directories: list[str]):
    results = {}

    with ThreadPoolExecutor(max_workers=3) as executor:
        futures = {
            executor.submit(run_agent, f"Review {d}"): d
            for d in directories
        }

        for future in as_completed(futures):
            directory = futures[future]
            results[directory] = future.result()

    return results
```

## Node.js Orchestration

### Basic Agent Runner

```javascript
const { execSync, spawn } = require('child_process');

function runAgent(prompt, options = {}) {
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
```

### Streaming Agent

```javascript
const { spawn } = require('child_process');
const readline = require('readline');

async function streamAgent(prompt, { onToken, onComplete }) {
  const proc = spawn('claude', [
    '-p', '--output-format', 'stream-json',
    '--include-partial-messages', prompt
  ]);

  const rl = readline.createInterface({ input: proc.stdout });

  for await (const line of rl) {
    const msg = JSON.parse(line);

    if (msg.type === 'stream_event') {
      if (msg.event.type === 'content_block_delta') {
        onToken?.(msg.event.delta.text || '');
      }
    } else if (msg.type === 'result') {
      onComplete?.(msg);
    }
  }
}
```

### Parallel Execution

```javascript
async function parallelReview(directories) {
  const promises = directories.map(async (dir) => {
    const result = await runAgent(`Review ${dir}`);
    return { dir, result };
  });

  return Promise.all(promises);
}
```

## Agent Patterns

### Supervisor-Worker Pattern

Supervisor decomposes task, workers execute subtasks:

```bash
#!/bin/bash
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

Agent reviews and improves its own output:

```bash
#!/bin/bash
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

Define agents with specific expertise:

```bash
# Define specialist agents
claude --agents '{
  "architect": {
    "description": "System design expert",
    "prompt": "You are a software architect. Focus on scalability and clean architecture."
  },
  "security": {
    "description": "Security analyst",
    "prompt": "You are a security expert. Identify vulnerabilities and suggest fixes."
  },
  "performance": {
    "description": "Performance optimizer",
    "prompt": "You are a performance expert. Optimize for speed and efficiency."
  }
}'

# Use specialist agents
claude --agent architect -p "Design the API structure"
claude --agent security -p "Review auth implementation"
claude --agent performance -p "Optimize database queries"
```

### Critic Pattern

One agent creates, another critiques:

```bash
#!/bin/bash
# Creator agent
code=$(claude -p --tools "Write" "Implement a sorting algorithm")

# Critic agent
critique=$(claude -p --tools "Read" \
  "Review this code for bugs, edge cases, and improvements")

# Apply feedback
claude -p --tools "Edit" "Apply these improvements: $critique"
```

## Session Management

### Persistent Sessions

Maintain context across multiple invocations:

```bash
# Start session with specific ID
claude --session-id "agent-task-123" -p "Start implementing feature X"

# Continue same session (keeps context)
claude --session-id "agent-task-123" -c -p "Continue with the tests"

# Resume and fork (branch the conversation)
claude --resume "agent-task-123" --fork-session -p "Try alternative approach"
```

### Stateless Agents

For ephemeral, one-off tasks:

```bash
# Disable session persistence
claude -p --no-session-persistence "One-off task"
```

### Session-Based Workflow

```bash
#!/bin/bash
SESSION_ID="project-$(date +%Y%m%d-%H%M%S)"

# Phase 1: Analysis
claude --session-id "$SESSION_ID" -p "Analyze the codebase"

# Phase 2: Planning (continues from analysis)
claude --session-id "$SESSION_ID" -c -p "Create implementation plan"

# Phase 3: Implementation (continues from plan)
claude --session-id "$SESSION_ID" -c -p "Implement the plan"

echo "Session: $SESSION_ID"
```
