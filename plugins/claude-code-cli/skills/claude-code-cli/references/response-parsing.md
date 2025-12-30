# Response Parsing Reference

Complete guide for parsing Claude Code CLI output in all formats.

## Table of Contents

1. [Output Formats](#output-formats)
2. [JSON Output Structure](#json-output-structure)
3. [Stream-JSON (JSONL)](#stream-json-jsonl)
4. [Structured Output with Schema](#structured-output-with-schema)
5. [Tool Use Messages](#tool-use-messages)
6. [Subagents (Task Tool)](#subagents-task-tool)
7. [TodoWrite Tool](#todowrite-tool)
8. [Background Tasks](#background-tasks)
9. [Error Handling](#error-handling)
10. [Init Message Details](#init-message-details)

## Output Formats

| Format | Flag | Use Case |
|--------|------|----------|
| `text` | `--output-format text` | Simple scripts, human-readable |
| `json` | `--output-format json` | Programmatic parsing, single response |
| `stream-json` | `--output-format stream-json` | Real-time UI updates, long tasks |

### Text Output (Default)

```bash
claude -p "What is 2+2?"
# Output: 4
```

Exit code 0 = success, non-zero = error.

## JSON Output Structure

Returns a JSON array with message objects:

```bash
echo "Hello" | claude -p --output-format json
```

```json
[
  {"type": "system", "subtype": "init", "session_id": "...", "tools": [...], "model": "..."},
  {"type": "assistant", "message": {"role": "assistant", "content": [{"type": "text", "text": "..."}]}},
  {"type": "result", "subtype": "success", "is_error": false, "result": "...", "total_cost_usd": 0.001}
]
```

### Message Types

| Type | Description |
|------|-------------|
| `system` | Session initialization with tools, model, plugins |
| `assistant` | Claude's response (text or tool_use) |
| `user` | Tool results fed back to Claude |
| `result` | Final status with cost and usage |

### Parsing with jq

```bash
# Get final text result
claude -p --output-format json "prompt" | jq -r '.[-1].result'

# Check success
claude -p --output-format json "prompt" | jq '.[-1].is_error'

# Get cost
claude -p --output-format json "prompt" | jq '.[-1].total_cost_usd'

# Get structured output (when using --json-schema)
claude -p --output-format json --json-schema '...' "prompt" | jq '.[-1].structured_output'
```

### Parsing in Python

```python
import subprocess
import json

def run_claude(prompt: str) -> dict:
    result = subprocess.run(
        ["claude", "-p", "--output-format", "json", prompt],
        capture_output=True, text=True
    )
    messages = json.loads(result.stdout)
    result_msg = next(m for m in messages if m["type"] == "result")

    return {
        "success": not result_msg["is_error"],
        "text": result_msg["result"],
        "cost": result_msg["total_cost_usd"],
        "structured": result_msg.get("structured_output")
    }
```

### Parsing in Node.js

```javascript
const { execSync } = require('child_process');

function runClaude(prompt) {
  const output = execSync(`claude -p --output-format json "${prompt}"`, { encoding: 'utf-8' });
  const messages = JSON.parse(output);
  const result = messages.find(m => m.type === 'result');

  return {
    success: !result.is_error,
    text: result.result,
    cost: result.total_cost_usd,
    structured: result.structured_output
  };
}
```

## Stream-JSON (JSONL)

Each line is a separate JSON object (newline-delimited):

```bash
echo "Count to 3" | claude -p --output-format stream-json
```

```jsonl
{"type":"system","subtype":"init",...}
{"type":"assistant","message":{...}}
{"type":"result","subtype":"success",...}
```

### With Partial Messages

Add `--include-partial-messages` for real-time token streaming:

```jsonl
{"type":"system","subtype":"init",...}
{"type":"stream_event","event":{"type":"message_start",...}}
{"type":"stream_event","event":{"type":"content_block_delta","delta":{"text":"1"}}}
{"type":"stream_event","event":{"type":"content_block_delta","delta":{"text":"2"}}}
{"type":"stream_event","event":{"type":"message_stop"}}
{"type":"assistant","message":{...}}
{"type":"result","subtype":"success",...}
```

### Streaming Parser (Python)

```python
import subprocess
import json

def stream_claude(prompt: str, on_token=None, on_complete=None):
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
                text = event["delta"].get("text", "")
                if on_token:
                    on_token(text)
        elif msg["type"] == "result":
            if on_complete:
                on_complete(msg)

    proc.wait()

# Usage
stream_claude(
    "Explain Python",
    on_token=lambda t: print(t, end="", flush=True),
    on_complete=lambda r: print(f"\n\nCost: ${r['total_cost_usd']:.4f}")
)
```

### Streaming Parser (Node.js)

```javascript
const { spawn } = require('child_process');
const readline = require('readline');

async function streamClaude(prompt, { onToken, onComplete }) {
  const proc = spawn('claude', [
    '-p', '--output-format', 'stream-json',
    '--include-partial-messages', prompt
  ]);

  const rl = readline.createInterface({ input: proc.stdout });

  for await (const line of rl) {
    const msg = JSON.parse(line);

    if (msg.type === 'stream_event') {
      const event = msg.event;
      if (event.type === 'content_block_delta') {
        onToken?.(event.delta.text || '');
      }
    } else if (msg.type === 'result') {
      onComplete?.(msg);
    }
  }
}
```

## Structured Output with Schema

Use `--json-schema` to enforce response structure:

```bash
echo "What is 2+2?" | claude -p --output-format json \
  --json-schema '{"type":"object","properties":{"answer":{"type":"number"},"explanation":{"type":"string"}},"required":["answer"]}'
```

The structured output appears in the result message:

```json
{"type": "result", "structured_output": {"answer": 4, "explanation": "Basic addition"}, ...}
```

## Tool Use Messages

When Claude uses tools, you'll see tool_use and tool_result messages:

```json
[
  {"type": "assistant", "message": {"content": [
    {"type": "text", "text": "I'll create the file."},
    {"type": "tool_use", "id": "toolu_...", "name": "Write", "input": {"file_path": "/tmp/test.txt", "content": "Hello"}}
  ]}},
  {"type": "user", "message": {"content": [
    {"type": "tool_result", "tool_use_id": "toolu_...", "content": "File created successfully"}
  ]}, "tool_use_result": {"type": "create", "filePath": "/tmp/test.txt"}},
  {"type": "result", "subtype": "success", "result": "Done!"}
]
```

### Extracting Tool Actions

```bash
# List all tools used
claude -p --output-format json "Create test.txt" | \
  jq '[.[] | select(.type=="assistant") | .message.content[] | select(.type=="tool_use") | {tool: .name, input: .input}]'
```

## Subagents (Task Tool)

Claude can spawn subagents for complex tasks:

```json
{"type": "assistant", "message": {"content": [
  {"type": "tool_use", "name": "Task", "input": {
    "subagent_type": "Explore",
    "description": "Find config files",
    "prompt": "Find all .json config files",
    "run_in_background": false
  }}
]}}
```

### Subagent Types

| Type | Purpose |
|------|---------|
| `general-purpose` | Multi-step tasks, research |
| `Explore` | Fast codebase exploration |
| `Plan` | Architecture and planning |
| `claude-code-guide` | Claude Code documentation |
| `debug-detective` | Bug investigation |

### Subagent Messages

Subagent messages have `parent_tool_use_id` linking them to the Task:

```json
{"type": "user", "message": {...}, "parent_tool_use_id": "toolu_xxx"}
{"type": "assistant", "message": {...}, "parent_tool_use_id": "toolu_xxx"}
```

### Task Result

```json
{"type": "user", "tool_use_result": {
  "status": "completed",
  "agentId": "abc1234",
  "content": [{"type": "text", "text": "Found 3 files..."}],
  "totalDurationMs": 3950,
  "totalTokens": 13510,
  "totalToolUseCount": 1
}}
```

The `agentId` can be used to resume the agent later with `resume` parameter.

## TodoWrite Tool

The TodoWrite tool manages task lists:

```json
{"type": "tool_use", "name": "TodoWrite", "input": {
  "todos": [
    {"content": "Implement feature", "status": "in_progress", "activeForm": "Implementing feature"},
    {"content": "Write tests", "status": "pending", "activeForm": "Writing tests"},
    {"content": "Review code", "status": "completed", "activeForm": "Reviewing code"}
  ]
}}
```

### Todo Status Values

| Status | Description |
|--------|-------------|
| `pending` | Not started |
| `in_progress` | Currently working (only one at a time) |
| `completed` | Finished |

### TodoWrite Result

```json
{"tool_use_result": {
  "oldTodos": [...],
  "newTodos": [
    {"content": "Implement feature", "status": "in_progress", "activeForm": "Implementing feature"}
  ]
}}
```

### Tracking Task Progress

```bash
# Extract current todos from JSON output
claude -p --output-format json "task" | \
  jq '[.[] | select(.type=="user") | .tool_use_result | select(.newTodos) | .newTodos] | last'
```

## Background Tasks

Both Task and Bash tools support `run_in_background`:

```json
{"type": "tool_use", "name": "Task", "input": {
  "subagent_type": "general-purpose",
  "prompt": "Long running task...",
  "run_in_background": true
}}
```

Result for background task:

```json
{"tool_use_result": {
  "isAsync": true,
  "status": "async_launched",
  "agentId": "abc1234",
  "outputFile": "/tmp/claude/.../tasks/abc1234.output"
}}
```

Use `TaskOutput` tool to retrieve results:
- `block: true` - Wait for completion
- `block: false` - Check status immediately

## Error Handling

### CLI Errors (Exit Code Non-Zero)

```bash
claude -p --invalid-flag "test"
# stderr: error: unknown option '--invalid-flag'
# exit code: 1
```

### API/Runtime Errors (in JSON)

```json
{"type": "result", "subtype": "error", "is_error": true, "error": "Rate limit exceeded", ...}
```

### Robust Error Handling (Bash)

```bash
#!/bin/bash
output=$(claude -p --output-format json "task" 2>&1)
exit_code=$?

if [ $exit_code -ne 0 ]; then
  echo "CLI Error: $output" >&2
  exit 1
fi

is_error=$(echo "$output" | jq -r '.[-1].is_error')
if [ "$is_error" = "true" ]; then
  error=$(echo "$output" | jq -r '.[-1].error // .[-1].result')
  echo "API Error: $error" >&2
  exit 1
fi

result=$(echo "$output" | jq -r '.[-1].result')
echo "$result"
```

## Init Message Details

The first `system` message contains useful metadata:

```json
{
  "type": "system",
  "subtype": "init",
  "cwd": "/path/to/project",
  "session_id": "uuid",
  "tools": ["Bash", "Read", "Edit", "Write", ...],
  "mcp_servers": [{"name": "server", "status": "connected"}],
  "model": "claude-sonnet-4-5-20250929",
  "permissionMode": "default",
  "skills": ["skill-name", ...],
  "plugins": [{"name": "plugin", "path": "..."}]
}
```

Use this to verify configuration before processing responses.
