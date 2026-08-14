# Vengtoo Toolkit

Agent skills for [Vengtoo](https://vengtoo.com) — add fine-grained authorization to any project, manage your policy model, authorize AI agent tool calls, and debug access decisions, all from your AI coding agent.

## Prerequisites

The skills work without any configuration — they provide code patterns and step-by-step guidance for any project.

To use MCP tools (create policies, list subjects, evaluate access decisions from the chat), your agent needs `VENGTOO_API_KEY` available as an environment variable. Set it however your shell loads env vars — the value needs to be present when Claude Code starts. Get your key at [console.vengtoo.com](https://console.vengtoo.com) → Settings → API Keys.

## Install

```bash
claude plugins add https://github.com/vengtoo/vengtoo-toolkit/plugins/vengtoo
```

## Skills

| Command | What it does |
|---|---|
| `/vengtoo` | Install the SDK and wire the first authorization check (Node.js, Python, Go) |
| `/vengtoo-agent` | Deploy the Vengtoo Agent locally for sub-millisecond decisions |
| `/vengtoo-policies` | Build resource types, subjects, roles, policies, and assignments |
| `/vengtoo-mcp` | Authorize MCP tool calls — Gateway and Adapter patterns |
| `/vengtoo-abac` | Add attribute-based conditions (time windows, IP ranges, resource properties) |
| `/vengtoo-debug` | Diagnose unexpected authorization decisions |
| `/vengtoo-ai-agents` | Delegation chains, human-in-the-loop escalation, JIT time-boxed access |
| `/vengtoo-terraform` | Manage the full authorization model as Terraform HCL |
| `/vengtoo-migrate` | Move off OPA/Rego, Cedar, Casbin, or homegrown RBAC — discover, translate, shadow-test, cut over |

## MCP Server

The toolkit wires the [Vengtoo MCP Server](https://github.com/vengtoo/vengtoo-mcp-server) automatically. Once the plugin is installed, your agent can talk to Vengtoo cloud directly — list subjects, create policies, evaluate access decisions — without leaving the chat.

## Links

- [Vengtoo Console](https://console.vengtoo.com)
- [Documentation](https://docs.vengtoo.com)
- [MCP Server](https://github.com/vengtoo/vengtoo-mcp-server)
