# General-Purpose Agent Reference Standard (GPARS)

| | |
|---|---|
| **Version** | 0.1.0 |
| **Status** | Draft |
| **Date** | 2026-03-04 |

## Abstract

An agent is general-purpose when its capabilities are not limited by what is embedded inside it. GPARS achieves this by mandating a strict separation between cognitive reasoning and environment-modifying actions: all external operations MUST be performed through MCP-compliant servers, not through embedded tools. This disaggregation enables agents to compose arbitrary capability ecosystems without architectural lock-in. Agents declare their requirements via a machine-readable manifest, and authorization is enforced by the user at the Action Plane boundary — agents cannot self-authorize or self-assert their identity.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Scope](#2-scope)
3. [Conformance Language](#3-conformance-language)
4. [Terminology](#4-terminology)
5. [Architecture](#5-architecture)
   - 5.1 [Cognitive Plane](#51-cognitive-plane)
   - 5.2 [Action Plane](#52-action-plane)
   - 5.3 [Plane Boundary](#53-plane-boundary)
6. [Environment-Modifying Operations](#6-environment-modifying-operations)
   - 6.1 [EMO Rules](#61-emo-rules)
   - 6.2 [EMO Classification](#62-emo-classification)
   - 6.3 [Edge Cases](#63-edge-cases)
7. [Agent Manifest](#7-agent-manifest)
   - 7.1 [Manifest Structure](#71-manifest-structure)
   - 7.2 [Required MCP Servers](#72-required-mcp-servers)
   - 7.3 [Optional MCP Servers](#73-optional-mcp-servers)
   - 7.4 [Identity Field](#74-identity-field)
   - 7.5 [Permission Scopes](#75-permission-scopes)
8. [Security Policy](#8-security-policy)
   - 8.1 [Policy Authority](#81-policy-authority)
   - 8.2 [Policy Scope](#82-policy-scope)
   - 8.3 [Standard Errors](#83-standard-errors)
9. [Activation and Processing Model](#9-activation-and-processing-model)
10. [Enforcement Model](#10-enforcement-model)
11. [Compliance Levels](#11-compliance-levels)
12. [Security Considerations](#12-security-considerations)
13. [Extension Model](#13-extension-model)
14. [Rationale](#14-rationale)
15. [Non-Goals (v0.1)](#15-non-goals-v01)
16. [References](#16-references)
- [Appendix A: Worked Example](#appendix-a-worked-example)

---

## 1. Introduction

The General-Purpose Agent Reference Standard (GPARS) defines a normative structure for separating cognitive agents from external capabilities using the Model Context Protocol (MCP).

Current agent systems — even those leveraging MCP — embed tool implementations directly within the agent loop. This creates tight coupling between reasoning and execution, produces vendor-specific behavior, and prevents true portability. A general-purpose agent cannot presume intrinsic capabilities; it must operate across environments by composing externalized services.

GPARS establishes a **strict separation between cognition and action**: all Environment-Modifying Operations MUST be externalized through MCP-compliant servers. Agents declare their capability requirements via a machine-readable manifest, enabling deterministic, composable, and governable capability surfaces.

This specification targets **MCP 2025-11-25** (latest stable revision at time of writing). Future GPARS versions MAY update this dependency as MCP evolves.

---

## 2. Scope

GPARS v0.1 standardizes:

- Mandatory separation between cognitive reasoning and environment-modifying actions.
- Agent capability declaration via a manifest.
- Mandatory declaration of required MCP servers and permission scopes.
- A reference architecture defining the boundary between cognition and action.
- A user-controlled enforcement point at the plane boundary.
- A security policy model where the user controls agent authorization.
- A standard authorization denial error at the MCP level.
- Enforcement expectations during agent operation.

GPARS v0.1 does NOT standardize:

- Authentication mechanisms.
- Cryptographic identity schemes.
- Plane boundary enforcement point implementations.
- MCP server implementation details.
- Token formats or session metadata schemas.
- Degradation or fallback policies.
- Policy management engine implementations.
- Fine-grained policy rule syntax.

---

## 3. Conformance Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", and "MAY" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

---

## 4. Terminology

| Term | Definition |
|------|------------|
| **Agent** | A cognitive process that generates MCP requests. An agent reasons, plans, and evaluates goals but does not execute actions directly. |
| **General-Purpose Agent** | An agent designed to operate across variable capability ecosystems without presuming intrinsic capabilities. Specialization emerges from the agent loop (system prompts, skills, reasoning) — not from embedded tools. |
| **Agent Runtime** | The execution environment for the agent. Belongs to the Cognitive Plane. Responsible for running the agent, managing its lifecycle, validating manifests, and holding MCP client instances. |
| **MCP Server** | A capability provider implementing MCP. |
| **Security Policy** | A set of rules defined by the user (the owner of the Action Plane) that governs what operations an agent is permitted to perform. The policy is bound to an agent identity and enforced at the Action Plane boundary. |
| **Internal Cognitive State (ICS)** | State fully isolated to a single agent execution context, whose modifications or observations have no authoritative effect outside the agent. Examples: planning buffers, ephemeral memory, simulated outputs. |
| **Environment State (ES)** | Any state whose modification or retrieval can influence or be observed by external systems, agents, or principals. Examples: filesystems, databases, network endpoints, MCP servers, shared memory, hardware devices. |
| **Environment-Modifying Operation (EMO)** | Any operation that modifies or retrieves Environment State. This includes both reads and writes. |
| **Capability Manifest** | A machine-readable declaration of the MCP servers, permissions, and optional capabilities required for an agent to operate. The manifest is a capability *requirement*, not an authorization grant. |

---

## 5. Architecture

GPARS defines two planes separated by a hard boundary. The Cognitive Plane is where the agent reasons. The Action Plane is where operations execute. The Action Plane is owned by the user and is the sole authority over what agents are permitted to do within it.

```
┌──────────────────────────────────────────────────────┐
│                   Cognitive Plane                    │
│                                                      │
│   Agent Runtime                                      │
│     ├── Agent (reasoning, planning, goal eval)       │
│     ├── Manifest validation                          │
│     └── MCP Clients (outbound requests)              │
│                                                      │
│               Internal Cognitive State               │
└───────────────────────┬──────────────────────────────┘
                        │
                        │ MCP requests
                        │
        ========================================        
                     Plane Boundary             
           (user-controlled enforcement point)  
        ========================================
┌─────────────────────────────────────────────────────┐
│          Action Plane (owned by user)               │
│                                                     │
│  ┌──────────────────────────────────────────────┐   │
│  │     Security Policy (per-agent, per-server)   │  │
│  │     identity verification · authorization    │   │
│  │     audit logging · scope enforcement        │   │
│  └──────────────────────────────────────────────┘   │
│                                                     │
│  ┌─────────┐  ┌────────────┐  ┌───────────────┐     │
│  │  bash   │  │ filesystem │  │   database    │     │
│  │ server  │  │   server   │  │    server     │     │
│  └────┬────┘  └─────┬──────┘  └──────┬────────┘     │
│       │             │                │              │
│       ▼             ▼                ▼              │
│          OS · network · hardware · storage          │
└─────────────────────────────────────────────────────┘
```

The boundary between planes is the central architectural constraint of GPARS. The Cognitive Plane MUST NOT perform Environment-Modifying Operations directly. All such operations MUST cross the boundary via MCP. The user's security policy governs what the agent is permitted to do once operations reach the Action Plane.

### 5.1 Cognitive Plane

Contains the Agent Runtime and the Agent.

**Agent Runtime**

The Runtime is the execution environment for the agent. It:
- MUST receive and validate the agent manifest before the agent loop begins.
- MUST resolve required MCP registry identifiers.
- MUST manage MCP client instances for outbound requests.

The Runtime belongs to the Cognitive Plane. It is NOT a security enforcement point — it is under the control of the agent developer or framework, not the user. Authorization is enforced at the Action Plane boundary.

**Agent**

The Agent:
- MUST perform all Environment-Modifying Operations exclusively through MCP.
- MUST NOT embed tool implementations that modify or retrieve Environment State.
- MAY embed tools that operate solely within Internal Cognitive State (e.g., sleep/wait, internal scheduling, retry logic, loop control).

The Cognitive Plane is confined to Internal Cognitive State (ICS). Everything the agent does that does not cross the MCP boundary is cognition. Everything that crosses the boundary is action. GPARS does not restrict what happens inside the Cognitive Plane — only what crosses the boundary.

### 5.2 Action Plane

The Action Plane is **owned by the user**. It contains the user's data, systems, and infrastructure. The user is the authority over what agents are permitted to do within it — just as a system administrator controls what users can do on a machine.

The Action Plane contains:
- **Security Policy** — user-defined rules governing agent permissions (see [Section 8](#8-security-policy)).
- **MCP Servers** — capability providers that execute operations and enforce policy.
- **Infrastructure** — OS, network, hardware, storage that MCP servers operate on.

MCP Servers:
- MUST enforce the user's security policy on every operation.
- MUST return a standard error when an operation violates policy or the server is unavailable (see [Section 8.3](#83-standard-errors)).
- MUST NOT be required to interpret GPARS manifests.
- MUST return structured result objects.
- SHOULD provide deterministic execution boundaries.

Servers are reusable across agents and do not need awareness of GPARS. They interact with underlying infrastructure on behalf of the agent — the agent never interacts with infrastructure directly.

### 5.3 Plane Boundary

The boundary between the Cognitive Plane and the Action Plane MUST be enforced by a user-controlled enforcement point. MCP servers MUST NOT accept requests directly from the Cognitive Plane without passing through this enforcement point.

The enforcement point is responsible for:
- Verifying the identity of the requesting agent (so that per-agent security policy can be applied).
- Routing MCP requests to the appropriate MCP servers.
- Ensuring the agent cannot bypass the security policy.

GPARS does not mandate a specific implementation for the enforcement point. Implementers MAY use a reverse proxy, network isolation, Unix socket permissions, process-level sandboxing, or any other mechanism that ensures the Cognitive Plane cannot directly access MCP servers while bypassing the security policy.

The critical requirement is: **the agent must not be able to self-assert its identity to the Action Plane**. Identity verification MUST be performed by infrastructure under the user's control, not by trusting the agent's self-declared `id` field.

---

## 6. Environment-Modifying Operations

### 6.1 EMO Rules

1. All EMOs MUST be executed exclusively through MCP servers.
2. Agents MUST NOT perform EMOs internally.
3. MCP responses MUST be treated as authoritative representations of Environment State.
4. Internal simulations of tool output are permitted but MUST NOT be treated as authoritative.

### 6.2 EMO Classification

**Operations that are EMOs** (MUST go through MCP):
- Reading or writing files
- Network communications (HTTP, RPC, WebSocket, etc.)
- Invoking other agents (see note below)
- Executing shell commands
- Accessing databases or shared stores
- Interacting with hardware devices

**Multi-agent interactions:** In GPARS v0.1, another agent is treated as an MCP server from the invoking agent's perspective. Agent-to-agent communication follows the same plane boundary rules, security policy enforcement, and error handling as any other MCP interaction. Dedicated agent-to-agent protocols (e.g., A2A) are out of scope for v0.1.

**Operations that are NOT EMOs** (MAY remain internal):
- Planning and text generation
- Hypothetical command simulations
- Internal memory summarization
- Token-level reasoning or computation

### 6.3 Edge Cases

- Cached or checkpointed internal memory is ICS unless it is authoritative or shared externally.
- Simulations of tool output are allowed but MUST NOT be treated as authoritative.
- Persistent memory that survives agent restart, is shared across processes, or is backed by disk is Environment State and MUST be accessed through MCP.
- Ephemeral memory confined to the reasoning loop is Internal Cognitive State.

---

## 7. Agent Manifest

### 7.1 Manifest Structure

The manifest MUST be provided to the Agent Runtime before the agent loop begins.

The formal schema is defined in [`agent_capability_manifest_schema.json`](./agent_capability_manifest_schema.json).

Minimal structure:

```json
{
  "id": "org:example/agent:researcher-v1",
  "required_mcp_servers": []
}
```

### 7.2 Required MCP Servers

The field `required_mcp_servers` is REQUIRED.

Each entry MUST include:

- `server_id` — canonical identifier from an official MCP registry.
- `version` — semantic version constraint.
- `permission_scopes` — list of required scopes.

Example:

```json
{
  "required_mcp_servers": [
    {
      "server_id": "registry.openmcp.org/bash",
      "version": ">=1.0.0",
      "permission_scopes": ["execute"]
    },
    {
      "server_id": "registry.openmcp.org/filesystem",
      "version": ">=1.0.0",
      "permission_scopes": ["read", "write"]
    }
  ]
}
```

All declared servers in this field are mandatory for the agent to function as intended. The manifest is declarative — it describes what the agent needs, not a precondition for starting the agent loop. If a required server is unavailable at operation time, the agent receives a standard error and handles it accordingly.

### 7.3 Optional MCP Servers

The field `optional_mcp_servers` is OPTIONAL.

Optional servers follow the same schema as required servers. If an optional server is unavailable, the agent MAY continue operating without it. Degradation behavior for missing optional servers is implementation-defined.

### 7.4 Identity Field

The `id` field is REQUIRED. It MUST uniquely identify the agent within its operational domain.

The format, issuance mechanism, and verification model are implementation-defined in v0.1.

### 7.5 Permission Scopes

Permission scopes in v0.1 are **server-defined and opaque to GPARS**. Each MCP server defines the scopes it recognizes. GPARS does not standardize a scope taxonomy.

The manifest declares which scopes the agent requires. These are capability *requirements* — they describe what the agent needs in order to function. They are NOT authorization grants. Whether the agent is actually permitted to exercise those scopes is determined by the user's security policy on the Action Plane (see [Section 8](#8-security-policy)).

---

## 8. Security Policy

### 8.1 Policy Authority

The **user** is the policy authority. The user owns the Action Plane — its data, systems, and infrastructure — and defines what agents are permitted to do within it.

The security policy:
- MUST be defined by the user (or by a policy management engine acting on the user's behalf).
- MUST be bound to the agent's verified identity as determined by the plane boundary enforcement point (see [Section 5.3](#53-plane-boundary)) — NOT by the agent's self-declared `id` field.
- MUST be enforced at the Action Plane boundary and by MCP servers, not by the agent itself.
- MAY support a default policy that applies when no agent-specific policy is defined.

The agent does not participate in defining, negotiating, or interpreting the security policy. The agent is a subject of the policy, not an author of it. The agent cannot influence how it is identified to the Action Plane.

### 8.2 Policy Scope

The security policy governs what operations an agent may perform within MCP server sessions. This includes but is not limited to:
- Which MCP servers the agent may connect to.
- Which scopes are granted per server.
- Resource-level constraints (e.g., which files, directories, endpoints, or databases the agent may access).

The agent is NOT informed of its effective permissions. The agent discovers its boundaries by receiving authorization denials when it attempts operations that violate the policy. This mirrors how user processes interact with operating system security — a process does not receive its SELinux policy; it receives `EACCES` when it violates it.

### 8.3 Standard Errors

GPARS defines the following standard MCP-level error codes:

| Code | Meaning | Returned by |
|------|---------|-------------|
| `AUTHORIZATION_DENIED` | The operation violates the user's security policy. | Enforcement point or MCP server |
| `SERVER_UNAVAILABLE` | The target MCP server is not reachable. | Enforcement point |

All standard errors MUST include:
- `code` — one of the standard error codes above.
- `message` — a human-readable description.

Errors SHOULD NOT include:
- The full security policy or its rules.
- Details about what the agent *would* be permitted to do.

**Authorization denial example:**

```json
{
  "error": {
    "code": "AUTHORIZATION_DENIED",
    "message": "Write access to /etc/passwd is not permitted for this agent."
  }
}
```

**Server unavailability example:**

```json
{
  "error": {
    "code": "SERVER_UNAVAILABLE",
    "message": "MCP server 'registry.openmcp.org/bash' is not reachable."
  }
}
```

The agent MUST treat authorization denials as authoritative. It MUST NOT retry the same denied operation expecting a different result. It MAY attempt alternative operations to achieve its goal.

The agent MAY retry after receiving `SERVER_UNAVAILABLE`, as the server may become available later.

---

## 9. Activation and Processing Model

1. The Agent provides the manifest to the Agent Runtime.
2. The Agent Runtime validates manifest structure and resolves required MCP registry identifiers.
3. The agent loop begins. The agent's lifecycle is independent of MCP server availability.
4. When the agent issues an MCP request, it passes through the plane boundary enforcement point.
5. The enforcement point verifies the agent's identity using infrastructure under the user's control (not by trusting the manifest's `id` field).
6. The enforcement point evaluates the request against the user's security policy. Operations that violate the policy are denied with `AUTHORIZATION_DENIED`.
7. If the target MCP server is unavailable, the agent receives a `SERVER_UNAVAILABLE` error. The agent MAY retry, wait, or adapt its approach.

The manifest is a declarative statement of what the agent needs — not an activation gate. Manifest interpretation occurs once, at the Agent Runtime (Cognitive Plane). Policy enforcement and availability are discovered at operation time, at the Action Plane boundary.

---

## 10. Enforcement Model

The manifest and the security policy serve different roles:

| Concern | Owner | Where it lives | When it applies |
|---------|-------|----------------|-----------------|
| **Capability Requirements** | Agent developer | Manifest (Cognitive Plane) | Declarative (informational) |
| **Security Policy** | User | Action Plane | Continuously during operation |

The manifest declares what the agent needs. The security policy determines what the agent is allowed to do. These are independent — an agent may declare a requirement for `filesystem: ["read", "write"]` but the user's policy may only permit `read` on specific paths.

Enforcement points:
- **Boundary** — the plane boundary enforcement point verifies agent identity and routes MCP requests. The agent cannot bypass this point.
- **Operation** — MCP servers enforce the user's security policy on every request. Unauthorized operations receive `AUTHORIZATION_DENIED`. Unavailable servers return `SERVER_UNAVAILABLE`.

The agent is never trusted to enforce its own boundaries or assert its own identity. All enforcement occurs outside the Cognitive Plane, under the user's control.

---

## 11. Compliance Levels

| Level | Criteria |
|-------|----------|
| **Compliant** | Agent fully externalizes all EMOs via MCP and publishes a valid capability manifest. |
| **Non-Compliant** | Agent performs EMOs internally or does not declare required MCP servers. |

Partial or transitional compliance levels are not defined in v0.1 and are reserved for future revision.

---

## 12. Security Considerations

- By externalizing EMOs, agents avoid executing untrusted or privileged actions internally.
- The user's security policy is the sole authorization authority. Agents cannot self-authorize.
- Agent identity is verified by infrastructure under the user's control at the plane boundary — agents cannot self-assert their identity to the Action Plane.
- MCP servers act as the enforcement layer, applying policy on every operation.
- Every EMO crosses the MCP boundary, creating a natural audit surface for all agent actions.
- Authorization denials are standard and structured, enabling agents to handle them gracefully without leaking policy details.
- The standard does NOT specify enforcement point implementations; implementers MAY use reverse proxies, network isolation, Unix socket permissions, containers, runtime sandboxes, or OS-level isolation.
- The standard does NOT specify identity verification mechanisms in v0.1; implementers MAY use mTLS, API keys, process-level isolation, or other methods appropriate to their deployment.
- The standard does NOT specify policy management engine implementations; users MAY use static config files, RBAC engines, or interactive approval prompts.

---

## 13. Extension Model

- Agents and MCP servers MAY declare additional metadata (capability descriptions, optional policies, discovery endpoints).
- Extension points MUST NOT violate the ICS/ES separation.
- Versioning and backward compatibility of MCP servers SHOULD follow semantic versioning conventions.
- Extension namespaces SHOULD use reverse-domain notation to avoid collisions.

---

## 14. Rationale

- **Separation of Cognition and Action:** Prevents cognitive bias induced by embedded execution tools. When tools are internal, the agent's reasoning is shaped by their implementation. Externalization ensures the agent reasons about *what* to do, not *how* tools work.
- **User-Owned Security:** The user owns the data and systems the agent operates on. The user — not the agent, not the agent developer — defines what is permitted. This mirrors established security models in operating systems, cloud platforms, and enterprise infrastructure.
- **Declarative Environment Requirements:** Ensures deterministic agent behavior and runtime portability. An agent's manifest fully describes what it needs to function.
- **Discovery-Based Authorization:** Agents are not told their effective permissions. They discover boundaries by receiving denials. This prevents agents from gaming policy boundaries and keeps the security model simple.
- **Modularity:** Enables emergent behavior across heterogeneous MCP capability ecosystems. Different cognitive models, reasoning architectures, and vendors can be composed without rewriting tool logic.
- **General-Purpose Validity:** A truly general-purpose agent operates across environments without presuming intrinsic capabilities. Specialization comes from the agent loop — not from embedded tools.

---

## 15. Non-Goals (v0.1)

GPARS v0.1 does NOT:

- Define identity infrastructure or verification mechanisms.
- Define authentication protocols.
- Define plane boundary enforcement point implementations.
- Define distributed orchestration standards.
- Standardize capability token formats.
- Define degradation or fallback policies for missing optional servers.
- Define runtime failure semantics for MCP server unavailability during a session.
- Define policy management engine implementations or policy rule syntax.
- Define how the security policy is distributed to MCP servers.

These concerns are reserved for future revisions.

---

## 16. References

### Normative

- **[RFC 2119]** Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, March 1997. https://www.rfc-editor.org/rfc/rfc2119

- **[MCP 2025-11-25]** Anthropic, "Model Context Protocol Specification", Revision 2025-11-25. https://modelcontextprotocol.io/specification/2025-11-25/

### Informative

- **[JSON Schema]** JSON Schema: A Media Type for Describing JSON Documents, Draft 2020-12. https://json-schema.org/draft/2020-12/json-schema-core

---

## Appendix A: Worked Example (Non-Normative)

This appendix walks through a complete interaction between a GPARS-compliant coding agent and the Action Plane.

### A.1 Agent Manifest

A coding assistant agent declares the following manifest:

```json
{
  "id": "org:example/agent:coding-assistant-v1",
  "version": "0.1.0",
  "required_mcp_servers": [
    {
      "server_id": "registry.openmcp.org/bash",
      "version": ">=1.0.0",
      "permission_scopes": ["execute"]
    },
    {
      "server_id": "registry.openmcp.org/filesystem",
      "version": ">=1.0.0",
      "permission_scopes": ["read", "write"]
    },
    {
      "server_id": "registry.openmcp.org/git",
      "version": ">=1.0.0",
      "permission_scopes": ["read", "write"]
    }
  ]
}
```

### A.2 Agent Startup

1. The Agent Runtime receives the manifest and validates its structure.
2. The Agent Runtime resolves the three `server_id` values against the MCP registry.
3. The agent loop begins. No MCP server availability check is required.

### A.3 Successful Operation

The agent decides to read a file. It issues an MCP request through the plane boundary:

**Request** (from agent, through enforcement point, to filesystem server):
```json
{
  "method": "read_file",
  "params": {
    "path": "/home/user/project/src/main.py"
  }
}
```

The enforcement point verifies the agent's identity (using infrastructure under the user's control), then forwards the request to the filesystem MCP server. The server checks the security policy, confirms read access is permitted for this path, and returns:

**Response:**
```json
{
  "result": {
    "content": "def main():\n    print('hello world')\n"
  }
}
```

### A.4 Authorization Denied

The agent attempts to write to a protected path:

**Request:**
```json
{
  "method": "write_file",
  "params": {
    "path": "/etc/passwd",
    "content": "malicious content"
  }
}
```

The user's security policy denies write access to `/etc/`. The MCP server returns:

**Response:**
```json
{
  "error": {
    "code": "AUTHORIZATION_DENIED",
    "message": "Write access to /etc/passwd is not permitted for this agent."
  }
}
```

The agent treats this as authoritative, does not retry, and attempts an alternative approach.

### A.5 Server Unavailable

The agent attempts to use the git MCP server, which is not currently running:

**Request:**
```json
{
  "method": "git_status",
  "params": {
    "repo": "/home/user/project"
  }
}
```

The enforcement point cannot reach the git server and returns:

**Response:**
```json
{
  "error": {
    "code": "SERVER_UNAVAILABLE",
    "message": "MCP server 'registry.openmcp.org/git' is not reachable."
  }
}
```

The agent may retry later or continue operating with the capabilities that are available.
