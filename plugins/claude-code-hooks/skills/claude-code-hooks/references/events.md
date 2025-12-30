# Hook Events Reference

Complete input/output schemas for each hook event.

## Common Input Fields

All hooks receive:
```json
{
  "session_id": "abc123",
  "transcript_path": "/path/to/session.jsonl",
  "cwd": "/current/working/directory",
  "permission_mode": "default|plan|acceptEdits|bypassPermissions",
  "hook_event_name": "EventName"
}
```

## PreToolUse

Runs before tool execution. Can block, allow, or modify tool calls.

**Matchers:** `Bash`, `Read`, `Write`, `Edit`, `Glob`, `Grep`, `Task`, `WebFetch`, `WebSearch`, `mcp__*`

**Input:**
```json
{
  "hook_event_name": "PreToolUse",
  "tool_name": "Write",
  "tool_input": {
    "file_path": "/path/to/file.ts",
    "content": "file contents"
  },
  "tool_use_id": "toolu_01ABC123"
}
```

**Output (JSON):**
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow|deny|ask",
    "permissionDecisionReason": "Reason text",
    "updatedInput": { "file_path": "/modified/path" }
  }
}
```

## PermissionRequest

Runs when user is shown a permission dialog.

**Matchers:** Same as PreToolUse

**Input:**
```json
{
  "hook_event_name": "PermissionRequest",
  "tool_name": "Bash",
  "tool_input": {
    "command": "npm install"
  },
  "tool_use_id": "toolu_01ABC123"
}
```

**Output (JSON):**
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "allow|deny",
      "updatedInput": { "command": "npm ci" },
      "message": "Denial reason",
      "interrupt": false
    }
  }
}
```

## PostToolUse

Runs after tool completes. Provides feedback to Claude.

**Matchers:** Same as PreToolUse

**Input:**
```json
{
  "hook_event_name": "PostToolUse",
  "tool_name": "Write",
  "tool_input": {
    "file_path": "/path/to/file.ts",
    "content": "contents"
  },
  "tool_response": {
    "filePath": "/path/to/file.ts",
    "success": true
  },
  "tool_use_id": "toolu_01ABC123"
}
```

**Output (JSON):**
```json
{
  "decision": "block",
  "reason": "Feedback for Claude",
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "additionalContext": "Extra context"
  }
}
```

## UserPromptSubmit

Runs when user submits a prompt, before processing.

**Matchers:** Not applicable (no matcher needed)

**Input:**
```json
{
  "hook_event_name": "UserPromptSubmit",
  "prompt": "User's prompt text"
}
```

**Output (JSON):**
```json
{
  "decision": "block",
  "reason": "Shown to user",
  "hookSpecificOutput": {
    "hookEventName": "UserPromptSubmit",
    "additionalContext": "Added to context"
  }
}
```

**Plain text stdout** is added as context (simplest method).

## Notification

Runs when Claude sends notifications.

**Matchers:** `permission_prompt`, `idle_prompt`, `auth_success`, `elicitation_dialog`

**Input:**
```json
{
  "hook_event_name": "Notification",
  "message": "Claude needs your permission",
  "notification_type": "permission_prompt"
}
```

**Output:** Exit code only (no decisions). stderr shown in debug mode.

## Stop

Runs when main agent finishes responding.

**Matchers:** Not applicable

**Input:**
```json
{
  "hook_event_name": "Stop",
  "stop_hook_active": false
}
```

`stop_hook_active` is true if Claude is already continuing due to a stop hook. Check this to prevent infinite loops.

**Output (JSON):**
```json
{
  "decision": "block",
  "reason": "Continue because..."
}
```

## SubagentStop

Runs when a subagent (Task tool) finishes.

**Matchers:** Not applicable

**Input:**
```json
{
  "hook_event_name": "SubagentStop",
  "stop_hook_active": false
}
```

**Output:** Same as Stop.

## PreCompact

Runs before compact operation.

**Matchers:** `manual` (from /compact), `auto` (from full context)

**Input:**
```json
{
  "hook_event_name": "PreCompact",
  "trigger": "manual|auto",
  "custom_instructions": ""
}
```

**Output:** Informational only (no decisions).

## SessionStart

Runs when session starts or resumes.

**Matchers:** `startup`, `resume`, `clear`, `compact`

**Input:**
```json
{
  "hook_event_name": "SessionStart",
  "source": "startup|resume|clear|compact"
}
```

**Environment:** `CLAUDE_ENV_FILE` available to persist environment variables.

**Output (JSON):**
```json
{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "Loaded context"
  }
}
```

**Plain text stdout** is added as context.

## SessionEnd

Runs when session ends.

**Matchers:** Not applicable

**Input:**
```json
{
  "hook_event_name": "SessionEnd",
  "reason": "clear|logout|prompt_input_exit|other"
}
```

**Output:** Cleanup only (no decisions).

## Prompt-Based Hooks

Alternative to command hooks, using LLM evaluation:

```json
{
  "hooks": {
    "Stop": [{
      "hooks": [{
        "type": "prompt",
        "prompt": "Evaluate if Claude should stop: $ARGUMENTS",
        "timeout": 30
      }]
    }]
  }
}
```

Use `$ARGUMENTS` as placeholder for hook input JSON.

**LLM Response Schema:**
```json
{
  "decision": "approve|block",
  "reason": "Explanation",
  "continue": false,
  "stopReason": "Message to user",
  "systemMessage": "Warning for user"
}
```

Best for context-aware decisions in Stop/SubagentStop hooks.
