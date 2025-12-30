# Hook Examples

Real-world hook implementations for common use cases.

## Code Formatting

### TypeScript/JavaScript with Prettier

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{
        "type": "command",
        "command": "jq -r '.tool_input.file_path' | { read f; echo \"$f\" | grep -qE '\\.(ts|tsx|js|jsx)$' && npx prettier --write \"$f\" 2>/dev/null; } || true"
      }]
    }]
  }
}
```

### Python with Black

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{
        "type": "command",
        "command": "jq -r '.tool_input.file_path' | { read f; echo \"$f\" | grep -q '\\.py$' && black --quiet \"$f\" 2>/dev/null; } || true"
      }]
    }]
  }
}
```

### Go with gofmt

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{
        "type": "command",
        "command": "jq -r '.tool_input.file_path' | { read f; echo \"$f\" | grep -q '\\.go$' && gofmt -w \"$f\"; } || true"
      }]
    }]
  }
}
```

## Linting and Validation

### ESLint Check

```python
#!/usr/bin/env python3
# .claude/hooks/eslint-check.py
import json, sys, subprocess

data = json.load(sys.stdin)
path = data.get("tool_input", {}).get("file_path", "")

if path.endswith((".ts", ".tsx", ".js", ".jsx")):
    result = subprocess.run(
        ["npx", "eslint", "--format", "compact", path],
        capture_output=True, text=True
    )
    if result.returncode != 0:
        errors = result.stdout.strip()
        if errors:
            print(json.dumps({
                "decision": "block",
                "reason": f"ESLint errors:\n{errors}\n\nFix these issues."
            }))
            sys.exit(0)

sys.exit(0)
```

### Type Check with tsc

```python
#!/usr/bin/env python3
# .claude/hooks/typecheck.py
import json, sys, subprocess, os

data = json.load(sys.stdin)
path = data.get("tool_input", {}).get("file_path", "")

if path.endswith(".ts") or path.endswith(".tsx"):
    result = subprocess.run(
        ["npx", "tsc", "--noEmit", path],
        capture_output=True, text=True, cwd=os.environ.get("CLAUDE_PROJECT_DIR", ".")
    )
    if result.returncode != 0:
        print(json.dumps({
            "decision": "block",
            "reason": f"TypeScript errors:\n{result.stdout}{result.stderr}"
        }))
        sys.exit(0)

sys.exit(0)
```

## File Protection

### Block Sensitive Files

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{
        "type": "command",
        "command": "jq -r '.tool_input.file_path' | grep -qE '(\\.env|\\.git/|secrets|credentials|password)' && echo 'Cannot modify sensitive files' >&2 && exit 2 || exit 0"
      }]
    }]
  }
}
```

### Protect Production Configs

```python
#!/usr/bin/env python3
import json, sys

PROTECTED = [
    "production.config",
    "prod.env",
    ".github/workflows/deploy",
    "infrastructure/",
]

data = json.load(sys.stdin)
path = data.get("tool_input", {}).get("file_path", "")

for pattern in PROTECTED:
    if pattern in path:
        print(f"Cannot modify production file: {path}", file=sys.stderr)
        sys.exit(2)

sys.exit(0)
```

## Logging and Monitoring

### Command Audit Log

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "jq -r '[.session_id, (.tool_input.command | @json)] | @tsv' >> ~/.claude/audit.log"
      }]
    }]
  }
}
```

### File Change Log

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Write|Edit",
      "hooks": [{
        "type": "command",
        "command": "jq -r '[now | strftime(\"%Y-%m-%d %H:%M:%S\"), .tool_name, .tool_input.file_path] | @tsv' >> ~/.claude/file-changes.log"
      }]
    }]
  }
}
```

## Notifications

### Desktop Notification (macOS)

```json
{
  "hooks": {
    "Notification": [{
      "matcher": "idle_prompt",
      "hooks": [{
        "type": "command",
        "command": "osascript -e 'display notification \"Claude is waiting for input\" with title \"Claude Code\"'"
      }]
    }]
  }
}
```

### Desktop Notification (Linux)

```json
{
  "hooks": {
    "Notification": [{
      "matcher": "permission_prompt",
      "hooks": [{
        "type": "command",
        "command": "notify-send 'Claude Code' 'Permission required'"
      }]
    }]
  }
}
```

### Slack Webhook

```bash
#!/bin/bash
# .claude/hooks/slack-notify.sh
MESSAGE=$(jq -r '.message')
curl -s -X POST -H 'Content-type: application/json' \
  --data "{\"text\": \"$MESSAGE\"}" \
  "$SLACK_WEBHOOK_URL"
```

## Bash Command Validation

### Block Dangerous Commands

```python
#!/usr/bin/env python3
import json, sys, re

DANGEROUS = [
    r"\brm\s+-rf\s+/",
    r"\bsudo\s+rm\b",
    r"\b>\s*/dev/sd",
    r"\bdd\s+if=.*of=/dev/",
    r"\bmkfs\.",
    r":(){:|:&};:",
]

data = json.load(sys.stdin)
if data.get("tool_name") != "Bash":
    sys.exit(0)

cmd = data.get("tool_input", {}).get("command", "")

for pattern in DANGEROUS:
    if re.search(pattern, cmd):
        print(f"Blocked dangerous command: {cmd}", file=sys.stderr)
        sys.exit(2)

sys.exit(0)
```

### Suggest Better Alternatives

```python
#!/usr/bin/env python3
import json, sys, re

SUGGESTIONS = [
    (r"\bgrep\b(?!.*\|)", "Use 'rg' (ripgrep) instead of grep"),
    (r"\bfind\s+\S+\s+-name\b", "Use 'fd' or 'rg --files' instead of find"),
    (r"\bcat\s+\S+\s*\|\s*grep", "Use 'grep FILE' directly, or 'rg PATTERN FILE'"),
]

data = json.load(sys.stdin)
if data.get("tool_name") != "Bash":
    sys.exit(0)

cmd = data.get("tool_input", {}).get("command", "")
issues = []

for pattern, msg in SUGGESTIONS:
    if re.search(pattern, cmd):
        issues.append(msg)

if issues:
    print("\n".join(f"• {m}" for m in issues), file=sys.stderr)
    sys.exit(2)

sys.exit(0)
```

## Session Context

### Load Project Info at Start

```bash
#!/bin/bash
# .claude/hooks/session-init.sh

# Persist environment
if [ -n "$CLAUDE_ENV_FILE" ]; then
    echo 'export NODE_ENV=development' >> "$CLAUDE_ENV_FILE"
fi

# Output context (added to session)
cat << EOF
Project: $(basename "$CLAUDE_PROJECT_DIR")
Node: $(node -v 2>/dev/null || echo "not installed")
Branch: $(git branch --show-current 2>/dev/null || echo "unknown")
EOF
```

### Load Recent Git Changes

```bash
#!/bin/bash
echo "Recent changes:"
git log --oneline -5 2>/dev/null || echo "No git history"
echo ""
echo "Modified files:"
git status --porcelain 2>/dev/null | head -10 || echo "Clean"
```

## Stop Hook: Task Verification

### Ensure Tests Pass

```python
#!/usr/bin/env python3
import json, sys, subprocess

data = json.load(sys.stdin)

# Prevent infinite loops
if data.get("stop_hook_active"):
    sys.exit(0)

# Run tests
result = subprocess.run(
    ["npm", "test", "--", "--passWithNoTests"],
    capture_output=True, text=True
)

if result.returncode != 0:
    print(json.dumps({
        "decision": "block",
        "reason": f"Tests are failing:\n{result.stdout}\n\nFix the tests before stopping."
    }))
    sys.exit(0)

sys.exit(0)
```

## Plugin Hooks

Hooks in plugin's `hooks/hooks.json`:

```json
{
  "description": "Auto-format code on save",
  "hooks": {
    "PostToolUse": [{
      "matcher": "Write|Edit",
      "hooks": [{
        "type": "command",
        "command": "${CLAUDE_PLUGIN_ROOT}/scripts/format.sh",
        "timeout": 30
      }]
    }]
  }
}
```

Script uses `$CLAUDE_PLUGIN_ROOT`:
```bash
#!/bin/bash
# ${CLAUDE_PLUGIN_ROOT}/scripts/format.sh
FILE=$(jq -r '.tool_input.file_path')
case "$FILE" in
    *.py) black --quiet "$FILE" ;;
    *.ts|*.js) npx prettier --write "$FILE" ;;
esac
```
