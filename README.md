# FilipeLabs Plugins

Claude Code plugins by Luis Filipe.

## Installation

```
/plugin marketplace add filipexyz/plugins
```

## Available Plugins

| Plugin | Description |
|--------|-------------|
| `flowsfarm` | Manage n8n workflows with FlowsFarm CLI |
| `claude-code-cli` | Guide for building AI agents using Claude Code CLI |
| `claude-code-hooks` | Implement Claude Code hooks for agent lifecycle control |
| `ratatui` | Build terminal UIs in Rust with Ratatui |

### Install a Plugin

```
/plugin install <plugin-name>@filipelabs
```

## Plugins

### FlowsFarm

Teaches Claude how to use [FlowsFarm](https://github.com/filipexyz/flowsfarm) for n8n workflow management.

```
/plugin install flowsfarm@filipelabs
```

Features:
- Sync workflows between local files and n8n instances
- Create, edit, and push workflow changes
- Manage workflow templates
- Version control your automations

### Claude Code CLI

Build AI agents using Claude Code CLI. Headless mode, JSON parsing, multi-agent orchestration.

```
/plugin install claude-code-cli@filipelabs
```

Includes reference docs for:
- Response parsing (JSON, streaming, subagents)
- Multi-agent orchestration (Python, Node.js, Bash)
- Configuration (tools, CI/CD, MCP servers)

### Claude Code Hooks

Implement lifecycle hooks for deterministic control over agent behavior.

```
/plugin install claude-code-hooks@filipelabs
```

Features:
- Hook events (PreToolUse, PostToolUse, Stop, SessionStart, etc.)
- Decision control (allow, deny, block, continue)
- Auto-formatting, linting, notifications
- File protection and command validation

### Ratatui

Build terminal UIs in Rust with immediate-mode rendering.

```
/plugin install ratatui@filipelabs
```

Features:
- Core architecture and event loop patterns
- Widgets (List, Table, Paragraph, Gauge, Tabs)
- Async integration with Tokio
- Keyboard navigation and accessibility

## License

MIT
