# SACP Whitepaper

Version: 1.0.0
Date: 2026-05-09

## Abstract

Multi-agent AI systems are now common in production workflows, but most implementations still rely on ad hoc message formats, inconsistent lifecycle handling, and weak execution guarantees.

The Semantic Agent Communication Protocol (SACP) addresses this by defining a strict, compact, transport-agnostic contract for agent-to-agent coordination. SACP standardizes how goals, plans, state transitions, errors, and outputs are represented so independently developed agents can interoperate reliably.

This whitepaper explains the design rationale behind SACP v1.0.0, the production requirements it targets, and the operational model required for safe, observable, and scalable deployments.

## 1. The Problem

### 1.1 Multi-Agent Systems Are Growing Faster Than Their Contracts

Organizations increasingly use specialized agents for planning, retrieval, execution, validation, and formatting. In many deployments, these components evolve independently and are assembled through orchestration code.

Without a strict protocol:
- Messages drift over time and silently break consumers.
- Error handling is inconsistent across agents.
- State transitions are implicit and hard to audit.
- Output shape is unpredictable, which breaks downstream chaining.
- Observability data is fragmented across tools.

### 1.2 Why Existing Integrations Fail in Production

Typical failure patterns include:
- Free-form JSON with no versioning contract
- Missing lifecycle semantics (no clear `in_progress` vs `failed` distinction)
- No standard retry or timeout metadata
- Mixed human and machine output in the same execution path
- No compatibility discipline for protocol upgrades

The result is brittle orchestration, expensive debugging, and high mean time to recovery.

## 2. SACP Design Goals

SACP v1.0.0 is designed around the following goals:

1. Determinism
A consumer should be able to validate and interpret every message unambiguously.

2. Interoperability
Independent agent implementations should communicate without custom glue code.

3. Operational Safety
State transitions, error handling, and execution policies should be explicit.

4. Efficiency
The message shape should stay compact enough for high-volume chaining.

5. Extensibility
New capabilities should be added without breaking the core contract.

## 3. Non-Goals

SACP intentionally does not define:
- Transport protocols (HTTP, gRPC, Kafka, queues)
- Organization-specific auth systems
- Domain-specific tool interfaces
- Prompting strategies for any particular model vendor

SACP is the application-layer coordination contract, not a full platform stack.

## 4. Core Protocol Model

A SACP message carries:
- A goal (`g`)
- Optional contextual constraints (`ctx`)
- A dependency-aware execution plan (`plan`)
- Lifecycle state (`state`)
- Optional terminal payload (`result` or `error`)

This gives each agent enough structure to:
- Validate input
- Execute or delegate steps
- Emit machine-readable state and outcome updates

## 5. Why Object-Based Plan Steps

Earlier protocol drafts used tuple steps (`[verb, tool, params]`). This was compact but limited:
- No stable place for step ids
- Hard to represent retry and timeout policy
- Weak dependency expression
- Poor forward compatibility

v1.0.0 uses object-based steps:
- `id`, `verb`, `tool`, `params` are required
- `depends_on`, `retry`, `timeout_ms`, and error policy are optional

This preserves compactness while enabling production-grade control.

## 6. Lifecycle State Machine as a First-Class Contract

Many systems treat lifecycle as implementation detail. SACP makes it protocol-level behavior.

Standard states:
- `PENDING`
- `ACCEPTED`
- `IN_PROGRESS`
- `WAITING`
- `DONE`
- `FAILED`
- `CANCELLED`

Benefits:
- Clear automation triggers
- Deterministic dashboards and SLAs
- Better incident diagnostics
- Easier handoffs across teams

Terminal state requirements:
- `DONE` must include `result`
- `FAILED` must include `error`

## 7. Output Discipline and Chain Reliability

SACP enforces output discipline through `out_fmt`:
- `raw` (default): machine chaining
- `summary`: compact machine-readable summary
- `human`: user-facing prose

Only interface/formatter stages should use `human` routinely.

This separation prevents a common reliability issue where intermediate agents emit prose that downstream systems cannot parse safely.

## 8. Error Model and Recovery

SACP standardizes error envelopes with canonical codes and optional retryability metadata.

Why this matters:
- Orchestrators can apply policy based on `error.code`.
- Recovery logic can be centralized.
- Dashboards can group failures by actionable category.

Recommended code families:
- Validation (`MALFORMED_MESSAGE`, `UNSUPPORTED_VERSION`)
- Capability (`UNKNOWN_VERB`, `UNKNOWN_TOOL`)
- Runtime (`TOOL_INVOCATION_FAILED`, `TIMEOUT`, `DEPENDENCY_FAILED`)
- Control plane (`AUTHZ_DENIED`, `RATE_LIMITED`)
- System (`INTERNAL_ERROR`, `CANCELLED`)

## 9. Capability Discovery and Dynamic Expansion

SACP supports a capability registry model where agents advertise:
- Supported verbs/tools
- Input/output expectations
- Version and optional confidence metadata

If no agent can satisfy a step, orchestrators may emit a `SPAWN_AGENT` goal.

This allows controlled, policy-governed expansion without rewriting the orchestration layer.

## 10. Security Considerations

SACP itself is transport-agnostic, so production deployments must define a security profile.

Minimum controls:
- Sender authentication
- Authorization per tool/action
- Parameter validation and sanitization
- Audit logging of lifecycle and tool calls

Recommended controls:
- Signed envelopes at trust boundaries
- Secrets redaction in `ctx`, `result`, and logs
- Per-sender allowlists for tools and verbs
- Approval workflow for side effects in `test_mode`

## 11. Observability and Operations

The optional `trace` object enables cross-service correlation and cost/performance analysis.

Useful metrics include:
- End-to-end latency
- Queue delay
- Tool call counts
- Token usage and cost

Operationally, this supports:
- SLA reporting
- Capacity planning
- Cost attribution
- Incident triage

## 12. Interoperability and Governance

To prevent protocol drift, teams should adopt:

1. Schema-first validation
Every inbound and outbound message is validated against `schema/sacp.schema.json`.

2. Conformance test suites
Test valid and invalid transition sequences, retry behavior, and terminal envelopes.

3. Version gates
Reject unsupported `proto_ver` values explicitly.

4. Change control
Protocol changes require semantic versioning and migration guidance.

## 13. Migration from Draft Systems

A practical migration path from informal or v0.x formats:

1. Add normalization adapters
Convert legacy tuples and field aliases into v1 object shape.

2. Shadow-validate
Run strict validation in monitor-only mode first.

3. Enforce producer conformance
Require all emitters to send v1.0.0 payloads.

4. Remove legacy paths
Turn off compatibility mode after telemetry confirms stability.

## 14. Expected Outcomes

Teams that adopt SACP should expect:
- Lower integration overhead across agent teams
- Fewer silent contract breaks
- Faster incident resolution due to standard states/errors
- Better parallel execution through explicit dependencies
- Cleaner separation between machine and human outputs

## 15. Limitations and Future Work

Current limitations:
- JSON Schema cannot enforce all semantic rules (e.g., cycle detection, globally unique step ids) without runtime checks.
- Security profile is intentionally external to the core spec.
- Tool interface definitions are still ecosystem-specific.

Potential future extensions:
- Standard policy attachment model
- Formal cancellation handshake semantics
- Streaming partial result contract
- Signed payload profile and trust chain spec

## 16. Conclusion

SACP v1.0.0 provides the minimum strong contract needed to make multi-agent systems dependable in production. It does not try to solve every platform concern, but it enforces the pieces that most often fail in real deployments: message validity, execution semantics, lifecycle transitions, and machine-readable outcomes.

By separating protocol guarantees from transport and model choice, SACP enables teams to evolve infrastructure and model stacks independently while preserving interoperability.

## Appendix A: Production Adoption Checklist

- Validate every message against `schema/sacp.schema.json`.
- Enforce lifecycle transition rules at runtime.
- Reject unknown/unauthorized verbs and tools.
- Implement per-step timeout and retry handling.
- Emit canonical `error.code` values.
- Track `trace_id` across every hop.
- Keep intermediate outputs machine-readable.
- Gate protocol changes through semver and compatibility tests.

## Appendix B: Reference Artifacts in This Repository

- `specification.md` (normative protocol contract)
- `schema/sacp.schema.json` (machine validation)
- `examples/message-example.json` (standard request)
- `examples/spawn-example.json` (dynamic agent spawn flow)
- `templates/agent-template.md` (agent behavior template)
