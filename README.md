# Depwire Action Token (DAT)

**An open standard for cryptographically signing AI agent actions.**

[![License: CC0-1.0](https://img.shields.io/badge/License-CC0_1.0-lightgrey.svg)](https://creativecommons.org/publicdomain/zero/1.0/)
[![Version](https://img.shields.io/badge/version-0.1.0--draft-blue.svg)](spec/dat-v0.1.0-draft.md)
[![Status](https://img.shields.io/badge/status-RFC-orange.svg)](spec/dat-v0.1.0-draft.md)
[![Open for Comments](https://img.shields.io/badge/RFC-open%20for%20comments-orange?style=flat)](https://github.com/depwire/dat-spec/issues)

---

**Renamed from AIT to DAT (May 2026)**

This specification was originally published as **AIT (Agent Identity Token)**. It has been renamed to **DAT (Depwire Action Token)** to avoid acronym collision with the IETF AgentID draft also using the term "AIT". The technical content, structure, and three token types (Session, Action, Delegation) are unchanged — only the name has been updated for clarity and distinctiveness.

---

> "DAT provides a tamper-proof, cryptographic audit trail for autonomous agents, ensuring that every AI-driven code change is tied to a verified identity, a specific model context, and an authorized security scope."

---

## The Problem

When an AI agent modifies your production codebase, calls an external API, or spawns a child agent — current systems cannot answer:

- **Who** authorized this action?
- **What model** executed it, at what version?
- **What context** did the model have when it acted?
- **Was the action within authorized scope?**
- **Has the audit record been tampered with?**

Existing logging systems record that an action occurred. They do not cryptographically bind that action to the agent's identity, model state, and authorization context at the moment of execution.

**DAT closes this gap.**

---

## What DAT Is

DAT is a JWT-based token standard (built on [RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519)) that defines three token types forming a verifiable chain of custody for every AI agent action:

| Token Type | Purpose |
|------------|---------|
| **DAT-S** Session | Who is this agent, what model, what scope |
| **DAT-A** Action | What did it do, to what target, with what context |
| **DAT-D** Delegation | What did the parent agent grant to the child |

Every DAT-A traces back through the delegation chain to a root DAT-S authorized by a human. No orphan actions. No identity laundering.

```
Human Authorization
        │
        ▼
   DAT-S (Session)
        │
        ├──► DAT-A (Action)        ← parent agent actions
        └──► DAT-D (Delegation)    ← spawns child agent
                  │
                  ├──► DAT-S (Session)
                  └──► DAT-A (Action)  ← child agent actions
```

---

## Quick Example

A DAT Action Token for a file modification:

```json
{
  "header": { "alg": "RS256", "typ": "DAT-A" },
  "payload": {
    "dat": "0.1",
    "jti": "550e8400-e29b-41d4-a716-446655440000",
    "iss": "depwire-cli@1.1.8",
    "iat": 1715123120,
    "exp": 1715123420,
    "session_ref": "parent-session-jti",
    "agent_id": "agent-uuid-v4",
    "action": {
      "type": "file_write",
      "tool": "edit_file",
      "target": "src/auth/token.ts"
    },
    "context": {
      "hash": "sha256-of-active-context-window"
    },
    "state": {
      "prev_hash": "sha256-of-file-before",
      "post_hash": "sha256-of-file-after"
    }
  }
}
```

Signed with RS256. Verifiable offline. No central authority required.

---

## Add a DAT Badge to Your Project

Show that your tool generates or verifies DAT tokens:

```markdown
[![DAT-Enabled](https://img.shields.io/badge/DAT-enabled-00d4aa?style=flat&labelColor=555)](https://github.com/depwire/dat-spec)
[![DAT-Verifiable](https://img.shields.io/badge/DAT-verifiable-00d4aa?style=flat&labelColor=555)](https://github.com/depwire/dat-spec)
[![DAT-Signed](https://img.shields.io/badge/agent-✓%20signed-00d4aa?style=flat&labelColor=555)](https://github.com/depwire/dat-spec)
```

[![DAT-Enabled](https://img.shields.io/badge/DAT-enabled-00d4aa?style=flat&labelColor=555)](https://github.com/depwire/dat-spec)
[![DAT-Verifiable](https://img.shields.io/badge/DAT-verifiable-00d4aa?style=flat&labelColor=555)](https://github.com/depwire/dat-spec)
[![DAT-Signed](https://img.shields.io/badge/agent-✓%20signed-00d4aa?style=flat&labelColor=555)](https://github.com/depwire/dat-spec)

---

## Specification

The full specification is in **[spec/dat-v0.1.0-draft.md](spec/dat-v0.1.0-draft.md)**.

It covers:

- Three token types and chain architecture
- Complete payload schemas with required/optional fields
- Cryptographic requirements (RS256 mandatory)
- Context and file state hashing algorithms
- Verification rules for single tokens and chains
- Scope inheritance and delegation rules
- Security considerations (context substitution, replay attacks, scope escalation)
- Comparison to JWT, OAuth, OpenTelemetry, W3C Provenance

---

## Machine-Readable Schemas

JSON Schema files for all three token types are in [`schemas/`](schemas/):

- [`dat-session.schema.json`](schemas/dat-session.schema.json) — DAT-S
- [`dat-action.schema.json`](schemas/dat-action.schema.json) — DAT-A
- [`dat-delegation.schema.json`](schemas/dat-delegation.schema.json) — DAT-D

---

## Reference Implementation

A reference implementation in [Depwire CLI](https://github.com/depwire/depwire) is in progress. It will provide:

- DAT token generation for every MCP tool call
- DAT chain verification
- DAT audit log export (JSON, SARIF)

Tracking: [Issue #1](https://github.com/depwire/dat-spec/issues/1)

Any tool, platform, or AI provider can implement DAT independently — the spec is CC0 (public domain) and requires no coordination with Depwire.

---

## Design Principles

- **Offline verifiable** — no network call required to verify a token
- **Model-agnostic** — works with Claude, GPT, Gemini, Llama, or any model
- **Composable** — builds on JWT; compatible with OAuth, OpenTelemetry, W3C Provenance
- **Implementation-defined key management** — use your existing PKI or HSM
- **Human authority at the root** — every chain traces to a human authorization event

---

## Status

DAT v0.1.0 is a **draft RFC** open for public comment.

Feedback is welcome via [GitHub Issues](https://github.com/depwire/dat-spec/issues). The goal is an open standard, not a Depwire-proprietary one.

**NIST alignment:** This specification is designed to align with NIST's February 2026 concept paper on *"Accelerating the Adoption of Software and AI Agent Identity and Authorization"* and the NIST AI Risk Management Framework.

---

## Governance

DAT is stewarded by Depwire but governed by the community. The spec is published under CC0 1.0 Universal (public domain) — Depwire holds no special rights over implementations or derivative standards.

Decisions about the spec are made through the public issue tracker. Any proposal with community consensus will be incorporated regardless of origin. The steward's role is to maintain the spec document and coordinate releases — not to control the direction.

If the community decides a different steward is more appropriate, the CC0 license ensures the spec can be forked and continued independently.

---

## Contributing

This specification is stewarded by Depwire under **CC0 1.0 Universal** (public domain). All contributions, implementations, and feedback are welcome.

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

---

## Related Work

| Standard | What it does | How DAT relates |
|----------|-------------|-----------------|
| JWT (RFC 7519) | General token format | DAT is built on JWT |
| OAuth 2.0 | Human identity delegation | DAT extends to agent identity |
| OpenTelemetry | Distributed tracing | DAT adds cryptographic signing |
| W3C Provenance | Data lineage | DAT can sign provenance records |
| NIST AI RMF | AI risk management | DAT implements the identity layer |

---

## Authors

**Atef Ataya** — ATEF ATAYA LLC, UAE — atef@depwire.dev

---

*[github.com/depwire/dat-spec](https://github.com/depwire/dat-spec) — CC0 1.0 Universal*
