# GPARS — General-Purpose Agent Reference Standard

An agent is general-purpose when its capabilities are not limited by what is embedded inside it.

GPARS defines a mandatory separation between cognitive reasoning and environment-modifying actions for AI agents. All external operations must be performed through [MCP](https://modelcontextprotocol.io/)-compliant servers — not through embedded tools. This disaggregation enables agents to compose arbitrary capability ecosystems without architectural lock-in.

## Key Principles

- **Cognitive Plane / Action Plane separation** — agents reason internally but act exclusively through MCP servers.
- **User-owned security** — the user controls the Action Plane and defines what agents are permitted to do. Agents cannot self-authorize.
- **Declarative manifests** — agents declare what MCP servers they need. The manifest is a requirement, not an authorization grant.
- **Discovery-based authorization** — agents are not told their permissions. They discover boundaries by receiving denials, just like OS processes.

## Specification

The current draft specification is at [`spec/v0.1/gpars_v0.1_spec.md`](./spec/v0.1/gpars_v0.1_spec.md).

The agent capability manifest schema is at [`spec/v0.1/agent_capability_manifest_schema.json`](./spec/v0.1/agent_capability_manifest_schema.json).

## Examples

See the [`examples/`](./examples/) directory for sample agent manifests:

- [`minimal_agent_manifest.json`](./examples/minimal_agent_manifest.json) — smallest valid manifest
- [`coding_agent_manifest.json`](./examples/coding_agent_manifest.json) — coding assistant with bash, filesystem, and git
- [`research_agent_manifest.json`](./examples/research_agent_manifest.json) — research assistant with browser and database

## Status

GPARS is currently a **v0.1 draft**. Feedback, questions, and contributions are welcome.

## Author

[Ismael Kaissy](https://github.com/15m43lk4155y)

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md).

## License

This project is licensed under the [MIT License](./LICENSE).
