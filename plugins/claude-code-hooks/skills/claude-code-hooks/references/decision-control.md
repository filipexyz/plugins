# Decision Control

Hook output controls Claude's behavior through JSON responses.

## PreToolUse Decisions

Control whether a tool call proceeds:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow|deny|ask",
    "permissionDecisionReason": "Explanation",
    "updatedInput": { "field": "modified value" }
  }
}
```

**Decisions:**
- `allow` - Bypass permission system (reason shown to user only)
- `deny` - Block tool call (reason shown to Claude)
- `ask` - Prompt user for confirmation (reason shown to user)

**Input Modification:**
```python
#!/usr/bin/env python3
import json, sys

data = json.load(sys.stdin)
if data["tool_name"] == "Bash":
    # Add safety flag to commands
    cmd = data["tool_input"]["command"]
    print(json.dumps({
        "hookSpecificOutput": {
            "hookEventName": "PreToolUse",
            "permissionDecision": "allow",
            "updatedInput": {"command": f"set -e; {cmd}"}
        }
    }))
else:
    sys.exit(0)
```

## PermissionRequest Decisions

Automatically allow or deny permission dialogs:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "allow|deny",
      "updatedInput": { "command": "modified" },
      "message": "Denial reason for Claude",
      "interrupt": false
    }
  }
}
```

**Auto-approve safe operations:**
```python
#!/usr/bin/env python3
import json, sys

data = json.load(sys.stdin)
tool = data.get("tool_name", "")
inp = data.get("tool_input", {})

# Auto-approve reading docs
if tool == "Read" and inp.get("file_path", "").endswith((".md", ".txt")):
    print(json.dumps({
        "hookSpecificOutput": {
            "hookEventName": "PermissionRequest",
            "decision": {"behavior": "allow"}
        }
    }))
    sys.exit(0)

sys.exit(0)  # Don't interfere
```

## PostToolUse Decisions

Provide feedback after tool execution:

```json
{
  "decision": "block",
  "reason": "Feedback for Claude",
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "additionalContext": "Extra information"
  }
}
```

**Lint check after edit:**
```python
#!/usr/bin/env python3
import json, sys, subprocess

data = json.load(sys.stdin)
path = data.get("tool_input", {}).get("file_path", "")

if path.endswith(".py"):
    result = subprocess.run(
        ["ruff", "check", path],
        capture_output=True, text=True
    )
    if result.returncode != 0:
        print(json.dumps({
            "decision": "block",
            "reason": f"Lint errors:\n{result.stdout}"
        }))
        sys.exit(0)

sys.exit(0)
```

## Stop/SubagentStop Decisions

Control whether Claude stops or continues:

```json
{
  "decision": "block",
  "reason": "Reason to continue working"
}
```

**Check task completion:**
```python
#!/usr/bin/env python3
import json, sys

data = json.load(sys.stdin)

# Prevent infinite loops
if data.get("stop_hook_active"):
    sys.exit(0)

# Check if TODO items remain
with open("TODO.md") as f:
    content = f.read()
    if "[ ]" in content:
        print(json.dumps({
            "decision": "block",
            "reason": "Uncompleted TODO items remain. Continue working."
        }))
        sys.exit(0)

sys.exit(0)
```

## UserPromptSubmit Decisions

Block or modify user prompts:

```json
{
  "decision": "block",
  "reason": "Shown to user (not Claude)",
  "hookSpecificOutput": {
    "hookEventName": "UserPromptSubmit",
    "additionalContext": "Added to context"
  }
}
```

**Add context to prompts:**
```python
#!/usr/bin/env python3
import json, sys
from datetime import datetime

# Simple: just print context to stdout
print(f"Current time: {datetime.now()}")
print(f"Active branch: main")
sys.exit(0)

# Or use JSON for more control
# print(json.dumps({
#     "hookSpecificOutput": {
#         "hookEventName": "UserPromptSubmit",
#         "additionalContext": "Context here"
#     }
# }))
```

## SessionStart Context

Inject context at session start:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "Loaded context"
  }
}
```

**Load project context:**
```bash
#!/bin/bash

# Persist environment variables
if [ -n "$CLAUDE_ENV_FILE" ]; then
    echo 'export NODE_ENV=development' >> "$CLAUDE_ENV_FILE"
fi

# Add context to session
cat << 'EOF'
{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "Project uses TypeScript, React, and Tailwind."
  }
}
EOF
```

## Common Patterns

### Exit Code 2 (Simple Blocking)

For simple blocks, use exit code 2 with stderr:
```bash
#!/bin/bash
# Block .env file modifications
jq -r '.tool_input.file_path' | grep -q '\.env' && {
    echo "Cannot modify .env files" >&2
    exit 2
}
exit 0
```

### JSON Output (Complex Control)

For decisions with reasons, modifications, or context:
```python
#!/usr/bin/env python3
import json, sys

data = json.load(sys.stdin)
# ... logic ...

print(json.dumps({
    "hookSpecificOutput": {
        "hookEventName": "PreToolUse",
        "permissionDecision": "deny",
        "permissionDecisionReason": "Detailed reason here"
    }
}))
sys.exit(0)
```

### Continuing After Stop

Force Claude to continue:
```json
{
  "decision": "block",
  "reason": "Continue because: specific reason"
}
```

### Stopping Immediately

Stop Claude regardless of other decisions:
```json
{
  "continue": false,
  "stopReason": "Message shown to user"
}
```
