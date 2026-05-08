# Agent Identity Token (AIT) Specification

**Version:** 0.1.0-draft  
**Status:** Request for Comments  
**Date:** May 8, 2026  
**Repository:** github.com/depwire/ait-spec  
**Reference Implementation:** Depwire CLI (github.com/depwire/depwire)  
**License:** CC0 1.0 Universal (Public Domain)

---

## Abstract

The Agent Identity Token (AIT) is an open standard for cryptographically signing the actions of autonomous AI agents. AIT provides a tamper-proof, offline-verifiable audit trail ensuring that every AI-driven action — every tool call, file modification, and agent delegation — is tied to a verified identity, a specific model and context, and an explicitly authorized scope.

AIT is designed to be model-agnostic, platform-agnostic, and composable with existing PKI infrastructure. It requires no central authority, no cloud dependency, and no coordination between AI providers.

> "AIT provides a tamper-proof, cryptographic audit trail for autonomous agents, ensuring that every AI-driven code change is tied to a verified identity, a specific model context, and an authorized security scope."

---

## 1. Problem Statement

### 1.1 The Identity Gap

Modern AI coding agents — Claude Code, GitHub Copilot Agent, Cursor, and others — can autonomously read files, write code, execute commands, and call external APIs. When such an agent modifies a production file at 2am, current systems cannot answer:

- **Who** authorized this action?
- **What model** executed it, with what version?
- **What context** did the model have when it acted?
- **Was the action within authorized scope?**
- **Has the audit record been tampered with?**

Existing logging systems record that an action occurred. They do not cryptographically bind that action to the agent's identity, model state, and authorization context at the moment of execution.

### 1.2 The Delegation Problem

Multi-agent systems introduce a compounding problem: identity laundering. A parent agent (Orchestrator) can spawn child agents (Workers) and delegate capabilities. Without a cryptographic chain, a malicious or compromised child agent can:

- Claim permissions it was never granted
- Execute actions outside its authorized scope
- Deny that its parent delegated the authority
- Break the audit trail entirely

Current systems have no mechanism to verify that a child agent's actions were within the scope delegated by its parent, or that the parent was itself authorized by a human.

### 1.3 The Audit Gap

Regulatory frameworks — NIST AI RMF, ISO/IEC 42001, SOC 2 Type II — increasingly require organizations to demonstrate control over automated systems. An AI agent that modifies source code, infrastructure configuration, or business logic is a control point that requires the same auditability as a human operator.

AIT closes this gap by making every agent action a first-class auditable event with cryptographic non-repudiation.

---

## 2. Design Principles

**2.1 Offline Verifiable**  
Any AIT token MUST be verifiable using only the issuer's public key. No network call, no central authority, no cloud service is required for verification.

**2.2 Model-Agnostic**  
AIT makes no assumptions about which AI model or provider is in use. Claude, GPT, Gemini, Llama, and any future model are equally supported.

**2.3 Minimal by Default**  
The core payload is small. Extended claims are optional. Implementations are not required to include fields they cannot populate.

**2.4 Composable**  
AIT builds on JWT (RFC 7519) and standard cryptographic primitives. Any system that can verify a JWT can verify an AIT token.

**2.5 Implementation-Defined Key Management**  
AIT mandates the cryptographic algorithm and verification logic but does not prescribe key management. Adopters may use existing PKI, hardware security modules (HSMs), or any key management system appropriate to their environment.

**2.6 Human Authority at the Root**  
Every AIT chain MUST be traceable to a human authorization event. Fully autonomous agent chains with no human root are explicitly out of scope for AIT v0.1 and SHOULD be rejected by compliant verifiers.

---

## 3. Token Architecture

AIT v0.1 defines three token types that chain together to form a complete, verifiable execution record.

### 3.1 Session Token (AIT-S)

Issued when an agent session begins. Defines the agent's identity, model, and authorized scope for the duration of the session.

**The Session Token answers:** Who is this agent, what model is it running, and what is it authorized to do?

### 3.2 Action Token (AIT-A)

Issued for each atomic action: a tool call, a file read, a file write, an API call, or a command execution. References either an AIT-S (direct human-authorized session) or an AIT-D (delegated session).

**The Action Token answers:** What did this agent do, to what target, with what context, and what was the state before and after?

### 3.3 Delegation Token (AIT-D)

Issued when a parent agent spawns a child agent and delegates a subset of its capabilities. References the parent's AIT-S or AIT-D, creating a verifiable chain of delegation.

**The Delegation Token answers:** What capabilities did the parent grant to the child, what was explicitly withheld, and for how long?

### 3.4 The Chain Structure

```
Human Authorization
        │
        ▼
   AIT-S (Session)          ← parent agent session
        │
        ├──► AIT-A (Action) ← parent agent actions
        │
        └──► AIT-D (Delegation) ← spawns child agent
                  │
                  ├──► AIT-S (Session) ← child agent session
                  │
                  └──► AIT-A (Action) ← child agent actions
```

Every AIT-A traces back through at most one AIT-D chain to a root AIT-S that was authorized by a human. This prevents identity laundering: a child agent cannot claim permissions not present in its AIT-D, and its AIT-D cannot claim permissions not present in the parent AIT-S.

---

## 4. Payload Schemas

All AIT tokens are JSON Web Tokens (JWT) as defined in RFC 7519. The `typ` header field identifies the token type.

### 4.1 Session Token (AIT-S)

```json
{
  "header": {
    "alg": "RS256",
    "typ": "AIT-S",
    "kid": "key-identifier"
  },
  "payload": {
    "ait": "0.1",
    "jti": "uuid-v4-unique-token-id",
    "iss": "depwire-cli@1.1.8",
    "iat": 1715123054,
    "exp": 1715126654,
    "agent_id": "uuid-v4",
    "model": "claude-sonnet-4-20250514",
    "model_provider": "anthropic",
    "human_ref": "sha256-of-human-authorization-event",
    "scope": {
      "read":    ["src/**", "package.json"],
      "write":   ["src/**"],
      "execute": ["npm", "git"],
      "deny":    ["*.env", "secrets/**"]
    },
    "environment": {
      "repo": "github.com/acme/api",
      "commit": "sha256-of-current-HEAD",
      "branch": "feature/auth-refactor"
    }
  }
}
```

**Required fields:** `ait`, `jti`, `iss`, `iat`, `exp`, `agent_id`, `model`, `model_provider`, `scope`  
**Optional fields:** `human_ref`, `environment`

### 4.2 Action Token (AIT-A)

```json
{
  "header": {
    "alg": "RS256",
    "typ": "AIT-A",
    "kid": "key-identifier"
  },
  "payload": {
    "ait": "0.1",
    "jti": "uuid-v4-unique-token-id",
    "iss": "depwire-cli@1.1.8",
    "iat": 1715123120,
    "exp": 1715123420,
    "session_ref": "jti-of-parent-AIT-S-or-AIT-D",
    "agent_id": "uuid-v4-must-match-session",
    "action": {
      "type": "file_write",
      "tool": "edit_file",
      "target": "src/auth/token.ts",
      "params": {
        "line_start": 42,
        "line_end": 47,
        "operation": "replace"
      }
    },
    "context": {
      "hash": "sha256-of-active-context-window",
      "files_in_context": [
        "src/auth/token.ts",
        "src/auth/middleware.ts"
      ],
      "token_count": 4821
    },
    "state": {
      "prev_hash": "sha256-of-file-before-action",
      "post_hash": "sha256-of-file-after-action"
    }
  }
}
```

**Required fields:** `ait`, `jti`, `iss`, `iat`, `exp`, `session_ref`, `agent_id`, `action`  
**Optional fields:** `context`, `state`

**Note on `state`:** For file write actions, `prev_hash` and `post_hash` SHOULD be included. For tool calls that do not modify files, `state` MAY be omitted.

### 4.3 Delegation Token (AIT-D)

```json
{
  "header": {
    "alg": "RS256",
    "typ": "AIT-D",
    "kid": "key-identifier"
  },
  "payload": {
    "ait": "0.1",
    "jti": "uuid-v4-unique-token-id",
    "iss": "depwire-cli@1.1.8",
    "iat": 1715123200,
    "exp": 1715126800,
    "parent_session_ref": "jti-of-parent-AIT-S-or-AIT-D",
    "parent_agent_id": "uuid-v4-of-parent",
    "child_agent_id": "uuid-v4-of-child",
    "child_model": "claude-haiku-4-5-20251001",
    "child_model_provider": "anthropic",
    "delegated_scope": {
      "read":    ["src/tests/**"],
      "write":   ["src/tests/**"],
      "execute": ["npm test"],
      "deny":    ["src/auth/**", "*.env", "secrets/**"]
    },
    "delegation_reason": "Generate unit tests for auth module"
  }
}
```

**Required fields:** `ait`, `jti`, `iss`, `iat`, `exp`, `parent_session_ref`, `parent_agent_id`, `child_agent_id`, `delegated_scope`  
**Optional fields:** `child_model`, `child_model_provider`, `delegation_reason`

**Scope inheritance rule:** `delegated_scope` MUST be a strict subset of the parent's scope. A compliant verifier MUST reject any AIT-D where `delegated_scope` grants permissions not present in the referenced parent token.

---

## 5. Cryptographic Requirements

### 5.1 Signing Algorithm

- **Mandatory:** RS256 (RSASSA-PKCS1-v1_5 with SHA-256) as defined in RFC 7518
- **Recommended minimum key size:** RSA 2048-bit
- **Recommended:** RSA 4096-bit for long-lived signing keys
- **Future versions** MAY add ES256 (ECDSA with P-256) as an alternative

### 5.2 Context Hashing

The `context.hash` field in AIT-A tokens MUST be computed as:

```
SHA-256(sorted(active_context_window_content))
```

Where `active_context_window_content` is the complete text content of all files, messages, and tool outputs in the model's active context at the time of action. Sorting ensures deterministic hashing regardless of insertion order.

### 5.3 File State Hashing

The `state.prev_hash` and `state.post_hash` fields MUST be computed as:

```
SHA-256(file_content_bytes)
```

Where `file_content_bytes` is the raw byte content of the file before and after the action respectively. For binary files, the raw bytes are hashed directly.

### 5.4 Token ID Uniqueness

Every `jti` field MUST be a UUID v4 generated with a cryptographically secure random number generator. Duplicate `jti` values from the same issuer MUST be rejected by compliant verifiers.

---

## 6. Verification

### 6.1 Single Token Verification

A compliant AIT verifier MUST:

1. Decode the JWT header and validate `alg` is RS256
2. Verify the JWT signature using the issuer's public key identified by `kid`
3. Verify `exp` is in the future (token has not expired)
4. Verify `iat` is in the past (token was not issued in the future)
5. Verify `ait` version is supported
6. Verify `agent_id` is a valid UUID v4
7. For AIT-A: verify `session_ref` references a valid, non-expired AIT-S or AIT-D

### 6.2 Chain Verification

To verify a complete AIT chain, a verifier MUST:

1. Verify each token individually per Section 6.1
2. Verify that `agent_id` in each AIT-A matches the `agent_id` (or `child_agent_id`) in the referenced session token
3. Verify that each AIT-D's `delegated_scope` is a strict subset of the referenced parent token's scope
4. Verify that every chain terminates at an AIT-S (no circular references)
5. Verify that the root AIT-S `human_ref` references a valid human authorization event (implementation-defined)

### 6.3 Scope Verification for Actions

For each AIT-A, a compliant verifier MUST verify that the `action.type` and `action.target` are within the `scope` of the referenced session or delegation token:

- `file_read` actions MUST match a pattern in `scope.read`
- `file_write` actions MUST match a pattern in `scope.write`
- `execute` actions MUST match a value in `scope.execute`
- Any target matching a pattern in `scope.deny` MUST be rejected regardless of other scope grants

---

## 7. Security Considerations

### 7.1 Context Substitution Attacks

Without context pinning, a malicious actor could show an agent safe data during planning but substitute malicious context at execution time. The `context.hash` field in AIT-A prevents this by cryptographically binding the action to the exact context the model observed.

Verifiers that require context integrity SHOULD store and verify the `context.hash` against a locally computed hash of the actual context window.

### 7.2 Replay Attacks

Short-lived tokens (`exp` within minutes of `iat`) combined with unique `jti` values mitigate replay attacks. Verifiers SHOULD maintain a cache of recently seen `jti` values for the duration of their validity period.

Recommended maximum token lifetime: 1 hour for AIT-S, 5 minutes for AIT-A, 24 hours for AIT-D.

### 7.3 Scope Escalation

A child agent MUST NOT be able to grant itself permissions beyond what its AIT-D allows. Verifiers MUST enforce strict scope subsetting at every delegation level. Implementations SHOULD default to denying any action not explicitly permitted rather than permitting any action not explicitly denied.

### 7.4 Key Compromise

AIT does not define a key revocation mechanism in v0.1. Implementations that require revocation SHOULD use short-lived tokens and implement their own revocation list. A future version of this specification MAY define a standard revocation mechanism.

### 7.5 Model Provider Impersonation

The `model` and `model_provider` fields are self-reported by the issuer and are not cryptographically verified by the model provider in AIT v0.1. A future version MAY define a model attestation mechanism in collaboration with model providers.

---

## 8. Action Types

AIT v0.1 defines the following standard action types for the `action.type` field:

| Type | Description |
|------|-------------|
| `file_read` | Read a file or directory listing |
| `file_write` | Create, modify, or delete a file |
| `tool_call` | Call an MCP tool or external API |
| `command_execute` | Execute a shell command or subprocess |
| `agent_spawn` | Spawn a child agent (paired with AIT-D) |
| `agent_terminate` | Terminate a child agent session |

Implementations MAY define additional action types using reverse-domain notation (e.g. `com.depwire.graph_query`).

---

## 9. Comparison to Related Work

| Standard | Purpose | Cryptographic | Agent-Specific | Offline Verifiable |
|----------|---------|---------------|----------------|-------------------|
| JWT (RFC 7519) | General token format | ✅ | ❌ | ✅ |
| OAuth 2.0 | Human identity delegation | ✅ | ❌ | ❌ |
| OpenTelemetry | Distributed tracing | ❌ | ❌ | ✅ |
| W3C Provenance | Data lineage | ❌ | ❌ | ✅ |
| **AIT v0.1** | **AI agent action signing** | ✅ | ✅ | ✅ |

AIT is not a replacement for any of the above. It is composable with all of them. An AIT token MAY be included as a claim in an OAuth token, an AIT chain MAY be exported as an OpenTelemetry trace, and AIT MAY be used to sign W3C Provenance records.

---

## 10. Reference Implementation

The reference implementation of AIT v0.1 is provided by Depwire CLI:

```bash
npm install -g depwire-cli
```

The reference implementation provides:

- AIT token generation for Depwire MCP tool calls
- AIT chain verification
- AIT audit log export (JSON, SARIF)

Source: github.com/depwire/depwire

---

## 11. Versioning

This specification uses semantic versioning. Breaking changes increment the major version. The `ait` claim in all tokens MUST reflect the spec version used.

Version `0.1` is a draft for public comment. Feedback is welcome via GitHub Issues at github.com/depwire/ait-spec.

---

## 12. IANA Considerations

This specification requests no IANA actions at this time. A future version MAY register AIT JWT claims with IANA.

---

## 13. Contributing

This specification is stewarded by Depwire under CC0 1.0 Universal license (public domain). Contributions, feedback, and alternative implementations are welcome. The goal is an open standard, not a Depwire-proprietary one.

To contribute: open an issue or pull request at github.com/depwire/ait-spec.

---

## Authors

Atef Ataya  
ATEF ATAYA LLC, UAE  
atef@depwire.dev

---

## References

- RFC 7519: JSON Web Token (JWT)
- RFC 7518: JSON Web Algorithms (JWA)
- NIST AI RMF: AI Risk Management Framework
- ISO/IEC 42001: AI Management Systems
- NIST: Accelerating the Adoption of Software and AI Agent Identity and Authorization (February 2026)

---

*AIT Specification v0.1.0-draft — May 8, 2026*  
*github.com/depwire/ait-spec*
