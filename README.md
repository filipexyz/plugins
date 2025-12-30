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

Guide for building and running AI agents using Claude Code CLI. Language-agnostic patterns for agent orchestration.

```
/plugin install claude-code-cli@filipelabs
```

Features:
- Authentication with API key or OAuth token (`claude setup-token`)
- Running Claude as a headless agent (`-p` mode)
- Structured JSON output with schema validation
- Multi-agent orchestration patterns (sequential, parallel)
- Tool restrictions and permission modes
- CI/CD integration (GitHub Actions, pre-commit hooks)
- Session management for persistent agents
- MCP server configuration

## License

MIT
