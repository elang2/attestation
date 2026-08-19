# Predicate type: AI Agent Action

Type URI: https://in-toto.io/attestation/ai-agent-action/v0.1

Version: 0.1.0

Authors: Elankumaran Srinivasan (@elang2)

Predicate Name: ai-agent-action

## Purpose

This predicate describes actions performed by AI agents through tool-calling
protocols. As AI agents increasingly execute real-world operations autonomously
(file modifications, API calls, infrastructure changes, financial transactions),
organizations need cryptographically verifiable records of what an agent did,
when, and with what outcome.

Existing in-toto predicates cover software supply chain operations (builds,
scans, deployments). AI agent actions represent a new class of automated
operation that produces side effects in external systems but lacks standardized
attestation. This predicate fills that gap by recording tool invocations made
through protocols like the Model Context Protocol (MCP), enabling the same
verification workflows that in-toto provides for build and release pipelines.

## Use Cases

### Compliance auditing for autonomous AI systems

EU AI Act Article 14 requires human oversight capabilities for high-risk AI
systems, which implies the ability to review and verify historical agent actions.
An AI agent that approves expenses, modifies cloud infrastructure, or sends
communications on behalf of a user needs verifiable records that cannot be
fabricated or selectively deleted by the agent itself.

This predicate enables an intermediary (gateway, proxy, or audit sidecar) to
produce signed attestations of each tool call. A compliance team can later
verify the complete history of agent actions without trusting the agent's own
logs.

### AI supply chain integrity

When AI agents participate in software development workflows (code generation,
PR creation, deployment approval), their actions become part of the software
supply chain. This predicate integrates AI agent actions into the same
attestation framework used for builds, scans, and releases, enabling end-to-end
verification policies that span both human and AI contributors.

### Incident investigation

When an AI agent causes an incident (deleted a production database, sent an
unauthorized email, approved an invalid transaction), investigators need
tamper-evident records. Application logs written by the same process whose
behavior is under investigation create a conflict of interest. Third-party
attestation from a protocol intermediary provides independent evidence.

## Prerequisites

Familiarity with the in-toto Attestation Framework (v1 Statement format),
AI agent architectures (tool-calling patterns), and the Model Context Protocol
(MCP) is helpful but not required to understand this predicate.

## Model

The predicate models a single tool invocation as observed by an intermediary
sitting between an AI client and a tool server. The intermediary does not
execute the tool; it observes the request and response passing through it and
records the metadata.

Key entities:

- **Agent**: The AI system that initiated the tool call (identified by model,
  session, or principal)
- **Tool**: The capability being invoked (identified by name and namespace)
- **Intermediary**: The entity producing the attestation (identified in the
  statement's signing metadata)
- **Outcome**: Whether the tool call succeeded or failed, its duration, and
  any error classification

The predicate does not include the tool's input arguments or output content by
default, as these may contain sensitive data. The `contentDigest` field allows
optional binding to the actual request/response payloads without including them
inline.

## Schema

```jsonc
{
  // Standard attestation fields:
  "_type": "https://in-toto.io/Statement/v1",
  "subject": [{
    "name": "<agent-session-identifier>",
    "digest": { "sha256": "<chain-head-hash>" }
  }],

  // Predicate:
  "predicateType": "https://in-toto.io/attestation/ai-agent-action/v0.1",
  "predicate": {
    "action": {
      "type": "tool_call",
      "protocol": "mcp",
      "method": "tools/call",
      "toolName": "<string>",
      "namespace": "<string or null>",
      "timestamp": "<RFC 3339 timestamp>",
      "durationMs": "<integer>",
      "success": "<boolean>",
      "errorCode": "<integer or null>",
      "errorClass": "<string or null>"
    },
    "agent": {
      "principal": "<string or null>",
      "model": "<string or null>",
      "sessionId": "<string or null>"
    },
    "upstream": {
      "name": "<string or null>",
      "transport": "<string or null>"
    },
    "chain": {
      "previousHash": "<hex string or 'genesis'>",
      "sequenceNumber": "<integer>"
    },
    "contentDigest": {
      "request": { "sha256": "<hex>" },
      "response": { "sha256": "<hex>" }
    },
    "metadata": {
      "attestorVersion": "<string>",
      "configHash": "<hex or null>"
    }
  }
}
```

### Parsing Rules

This predicate follows the in-toto attestation parsing rules. Summary:

- Consumers MUST ignore unrecognized fields.
- The `predicateType` URI includes the major version number and will always
  change whenever there is a backwards incompatible change.
- The `predicate.chain` object is OPTIONAL. When absent, the attestation
  stands alone without ordering guarantees.
- The `predicate.contentDigest` object is OPTIONAL. When absent, the
  attestation does not bind to specific request/response payloads.

### Fields

#### `predicate.action`

| Field        | Type              | Required | Description                                               |
| ------------ | ----------------- | -------- | --------------------------------------------------------- |
| `type`       | string            | Yes      | Action type. Currently only `"tool_call"` is defined.     |
| `protocol`   | string            | Yes      | Protocol through which the action was observed. e.g. `"mcp"` |
| `method`     | string            | Yes      | Protocol method. e.g. `"tools/call"`                      |
| `toolName`   | string            | Yes      | Name of the tool as invoked by the agent                  |
| `namespace`  | string or null    | No       | Tool namespace (for multi-server routing)                 |
| `timestamp`  | string (RFC 3339) | Yes      | When the action was observed by the intermediary          |
| `durationMs` | integer           | Yes      | Wall-clock duration of the tool execution in milliseconds |
| `success`    | boolean           | Yes      | Whether the tool call completed without error             |
| `errorCode`  | integer or null   | No       | Protocol-level error code (e.g., JSON-RPC error code)     |
| `errorClass` | string or null    | No       | Human-readable error classification                       |

#### `predicate.agent`

| Field       | Type           | Required | Description                                         |
| ----------- | -------------- | -------- | --------------------------------------------------- |
| `principal` | string or null | No       | Identity of the requesting agent or user            |
| `model`     | string or null | No       | AI model identifier (e.g., `"claude-sonnet-4-20250514"`)  |
| `sessionId` | string or null | No       | Session or conversation identifier                  |

#### `predicate.upstream`

| Field       | Type           | Required | Description                                      |
| ----------- | -------------- | -------- | ------------------------------------------------ |
| `name`      | string or null | No       | Name of the upstream tool server                 |
| `transport` | string or null | No       | Transport type (e.g., `"stdio"`, `"streamable-http"`) |

#### `predicate.chain`

| Field            | Type              | Required | Description                                                    |
| ---------------- | ----------------- | -------- | -------------------------------------------------------------- |
| `previousHash`   | string            | Yes      | SHA-256 hex of the preceding attestation. First = `"genesis"`  |
| `sequenceNumber` | integer           | Yes      | Monotonically increasing counter within this chain             |

#### `predicate.contentDigest`

| Field      | Type                          | Required | Description                                    |
| ---------- | ----------------------------- | -------- | ---------------------------------------------- |
| `request`  | ResourceDescriptor (digests)  | No       | Digest of the canonical tool call request      |
| `response` | ResourceDescriptor (digests)  | No       | Digest of the tool call response content       |

#### `predicate.metadata`

| Field            | Type           | Required | Description                                        |
| ---------------- | -------------- | -------- | -------------------------------------------------- |
| `attestorVersion`| string         | Yes      | Version of the attestation-producing software      |
| `configHash`     | string or null | No       | Hash of the attestor's policy configuration        |

## Example

A complete attestation for an AI agent creating a pull request via the GitHub
MCP server:

```json
{
  "_type": "https://in-toto.io/Statement/v1",
  "subject": [{
    "name": "session:agent-workspace-4f2a",
    "digest": {
      "sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
    }
  }],
  "predicateType": "https://in-toto.io/attestation/ai-agent-action/v0.1",
  "predicate": {
    "action": {
      "type": "tool_call",
      "protocol": "mcp",
      "method": "tools/call",
      "toolName": "github/create_pull_request",
      "namespace": "github",
      "timestamp": "2026-08-18T14:32:01.998Z",
      "durationMs": 1247,
      "success": true,
      "errorCode": null,
      "errorClass": null
    },
    "agent": {
      "principal": "user:alice@example.com",
      "model": "claude-sonnet-4-20250514",
      "sessionId": "conv_abc123"
    },
    "upstream": {
      "name": "github-server",
      "transport": "stdio"
    },
    "chain": {
      "previousHash": "a1b2c3d4e5f6...",
      "sequenceNumber": 42
    },
    "contentDigest": {
      "request": { "sha256": "7d865e959b2466918c9863afca942d0fb89d7c9ac0c99bafc3749504ded97730" },
      "response": { "sha256": "2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824" }
    },
    "metadata": {
      "attestorVersion": "mcp-audit-gateway/0.1.0",
      "configHash": "9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08"
    }
  }
}
```

## Changelog and Migrations

This is the initial version (v0.1) of the AI Agent Action predicate. No
migrations are required.
