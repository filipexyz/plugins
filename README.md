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

## License

MIT
