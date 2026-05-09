# Semantic Agent Communication Protocol (SACP) - Specification v1.0.0

Status: Stable
Published: 2026-05-09

## 1. Scope

SACP is a transport-agnostic protocol for machine-to-machine coordination between autonomous agents.

This specification defines:
- Message envelope fields and validation rules
- Execution plan semantics
- Lifecycle state transitions
- Error and recovery behavior
- Compatibility and extension rules

This specification does not define:
- A transport protocol (HTTP, queue, event bus, etc.)
- A security transport profile (TLS, mTLS, etc.)
- Domain-specific tool contracts

## 2. Conformance Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" are to be interpreted as described in RFC 2119 and RFC 8174.

An implementation is conformant only if it satisfies all normative requirements in this document.

## 3. Protocol Model

SACP assumes an orchestrated multi-agent system:
- An interface or orchestrator agent receives human requests.
- Planner and executor agents process machine-readable plans.
- Formatter agents are the only agents expected to produce user-facing text.

SACP messages are the only required inter-agent contract.

## 4. Message Envelope

### 4.1 Required Fields

A valid SACP message MUST include:
- `proto_ver`
- `msg_id`
- `ts`
- `sender`
- `g`
- `plan`
- `state`

### 4.2 Field Definitions

| Field | Type | Required | Constraints |
|---|---|---|---|
| `proto_ver` | string | yes | Semantic version. In this spec, MUST be `1.0.0`. |
| `msg_id` | string | yes | Unique per sender for at least 24h. |
| `reply_to` | string | no | References prior `msg_id`. |
| `ts` | string | yes | RFC3339 `date-time` in UTC (suffix `Z`). |
| `sender` | string | yes | Stable agent/service identifier. |
| `recipient` | string | no | Target agent/service identifier. |
| `g` | string | yes | Goal identifier, 1..128 chars. |
| `ctx` | object | no | Context object for constraints and runtime hints. |
| `plan` | array | yes | Ordered execution steps, min length 1. |
| `out_fmt` | string | no | `raw`, `summary`, `human`. Defaults to `raw`. |
| `state` | string | yes | Lifecycle state from section 5. |
| `result` | object | conditional | REQUIRED when `state = DONE`. |
| `error` | object | conditional | REQUIRED when `state = FAILED`. |
| `trace` | object | no | Correlation and observability metadata. |
| `test_mode` | boolean | no | When true, side effects SHOULD be suppressed. |
| `ext` | object | no | Namespaced extensions. |

### 4.3 Envelope Rules

- Unknown top-level fields MUST be rejected.
- `ctx` and `result.outputs` MUST be JSON objects.
- `out_fmt` MUST default to `raw` when omitted.
- `result` and `error` MUST NOT both be present.
- `test_mode=true` MUST be propagated to all spawned/subordinate messages unless explicitly overridden by policy.

## 5. Lifecycle State Machine

### 5.1 States

- `PENDING`: Message created, not yet validated by recipient.
- `ACCEPTED`: Recipient validated message and accepted execution responsibility.
- `IN_PROGRESS`: Plan is actively executing.
- `WAITING`: Execution paused for dependency, external event, or human approval.
- `DONE`: Execution completed successfully.
- `FAILED`: Execution completed unsuccessfully.
- `CANCELLED`: Execution stopped by policy, user, or orchestrator.

### 5.2 Allowed Transitions

- `PENDING -> ACCEPTED | FAILED | CANCELLED`
- `ACCEPTED -> IN_PROGRESS | FAILED | CANCELLED`
- `IN_PROGRESS -> WAITING | DONE | FAILED | CANCELLED`
- `WAITING -> IN_PROGRESS | FAILED | CANCELLED`

`DONE`, `FAILED`, and `CANCELLED` are terminal states.

### 5.3 Transition Requirements

- Any terminal transition SHOULD include `trace.metrics` when available.
- `FAILED` MUST include an `error` object.
- `DONE` MUST include a `result` object.

## 6. Plan Semantics

### 6.1 Step Shape

Each `plan` item MUST be an object with the following required fields:
- `id` (unique step identifier within the message)
- `verb`
- `tool`
- `params`

Optional fields:
- `depends_on` (array of prior step ids)
- `timeout_ms`
- `retry`
- `on_error`
- `fallback_step`
- `output_key`

### 6.2 Step Fields

| Field | Type | Constraints |
|---|---|---|
| `id` | string | 1..64 chars, `[A-Za-z0-9._:-]+` |
| `verb` | string | 1..64 chars |
| `tool` | string | 1..64 chars |
| `params` | object | Tool-specific arguments |
| `depends_on` | array<string> | Each id MUST refer to another step id |
| `timeout_ms` | integer | 1..3,600,000 |
| `retry.max_attempts` | integer | 1..10 |
| `retry.backoff_ms` | integer | 0..600,000 |
| `retry.strategy` | string | `fixed` or `exponential` |
| `on_error` | string | `fail_fast`, `continue`, `fallback` |
| `fallback_step` | string | REQUIRED when `on_error=fallback` |
| `output_key` | string | Key for result output map |

### 6.3 Execution Rules

- Steps with no `depends_on` MAY execute immediately.
- Independent steps SHOULD be executed in parallel where possible.
- `depends_on` graphs MUST be acyclic.
- A step MUST NOT execute until all dependencies are in successful terminal state.
- If a dependency fails and no dependency policy allows continuation, dependent steps MUST be skipped.

### 6.4 Error Policy Rules

- `fail_fast`: first non-recoverable step error transitions message to `FAILED`.
- `continue`: step error is recorded; remaining independent steps continue.
- `fallback`: executor MUST attempt `fallback_step` exactly once after retries are exhausted.

### 6.5 Legacy Plan Form

The v0.1 tuple form `[verb, tool, params]` is deprecated.

- Producers conforming to v1.0.0 MUST emit object-form steps.
- Consumers MAY accept tuple-form steps through a compatibility adapter.
- Native v1.0.0 validators MUST treat tuple-form as invalid unless adapter mode is explicitly enabled outside this core spec.

## 7. Result and Error Objects

### 7.1 Result Object

`result` fields:
- `outputs` (object, REQUIRED): final named outputs
- `summary` (string, OPTIONAL): concise execution note
- `step_results` (object, OPTIONAL): keyed by step id

### 7.2 Error Object

`error` fields:
- `code` (string, REQUIRED)
- `message` (string, REQUIRED)
- `step_id` (string, OPTIONAL)
- `retryable` (boolean, OPTIONAL, default false)
- `details` (object, OPTIONAL)

### 7.3 Standard Error Codes

Implementations SHOULD use these canonical codes:
- `UNSUPPORTED_VERSION`
- `MALFORMED_MESSAGE`
- `INVALID_STATE_TRANSITION`
- `UNKNOWN_VERB`
- `UNKNOWN_TOOL`
- `TOOL_INVOCATION_FAILED`
- `TIMEOUT`
- `DEPENDENCY_FAILED`
- `AUTHZ_DENIED`
- `RATE_LIMITED`
- `INTERNAL_ERROR`
- `CANCELLED`

## 8. Output Discipline

- Default output mode is `raw`.
- Agents MUST produce machine-readable content unless `out_fmt=human`.
- Intermediate agents SHOULD NOT emit prose responses.
- Formatter or interface agents MAY emit human output only in `human` mode.

## 9. Capability Discovery and Spawning

### 9.1 Capability Registry Contract

A registry entry SHOULD include:
- `agent_name`
- `supported_verbs`
- `supported_tools`
- `inputs`
- `outputs`
- `version`
- `description`

### 9.2 Selection

Agent/tool selection SHOULD prioritize:
1. Exact verb-tool match
2. Input/output compatibility
3. Policy and trust constraints
4. Semantic similarity fallback (optional)

### 9.3 Dynamic Spawning

If no candidate exists, orchestrators MAY emit a goal `SPAWN_AGENT` message.
Spawned agents MUST be registered before retrying the blocked plan.

## 10. Security and Trust Requirements

SACP implementations MUST define and enforce a security profile that includes:
- Authentication of message sender
- Authorization for tool and action execution
- Input validation for `params`
- Audit logging for message lifecycle and tool calls

Recommended controls:
- Signed envelopes at transport boundary
- Policy-based tool allowlists per sender
- Redaction policy for sensitive `ctx` and `result` fields

## 11. Observability

The optional `trace` object SHOULD include:
- `trace_id`
- `span_id`
- `parent_span_id`
- `attempt`
- `metrics`

`trace.metrics` MAY include:
- `latency_ms`
- `queue_ms`
- `tool_calls`
- `tokens_in`
- `tokens_out`
- `cost_usd`

## 12. Compatibility and Versioning

- `proto_ver` governs compatibility.
- Breaking field or semantic changes require major version bump.
- Additive optional fields require minor version bump.
- Patch version is for clarifications and non-breaking corrections.

For `1.0.x`:
- Producers MUST emit fields defined by this spec.
- Consumers MUST reject unknown required semantics.

## 13. Conformance Test Expectations

A production implementation SHOULD include automated tests for:
- Schema validation success/failure cases
- State transition validity
- Dependency graph cycle detection
- Retry and timeout behavior
- Error code and terminal state mapping
- `out_fmt` discipline enforcement

## 14. Reference Message Examples

### 14.1 Request Message

```json
{
  "proto_ver": "1.0.0",
  "msg_id": "hotel-search-001",
  "ts": "2026-05-09T00:00:00Z",
  "sender": "INTERFACE_AGENT",
  "recipient": "EXECUTOR_AGENT",
  "g": "FIND_HOTEL",
  "ctx": {
    "loc": "Queenstown",
    "check_in": "2026-07-20",
    "check_out": "2026-07-25",
    "budget": 200,
    "amenities": ["wifi", "breakfast"]
  },
  "plan": [
    {
      "id": "s1",
      "verb": "fetch",
      "tool": "HISTO_API",
      "params": {"loc": "Queenstown", "dates": ["2026-07-20", "2026-07-25"]},
      "timeout_ms": 8000
    },
    {
      "id": "s2",
      "verb": "predict",
      "tool": "PRC_EST",
      "params": {"method": "linear", "window": 7},
      "depends_on": ["s1"]
    },
    {
      "id": "s3",
      "verb": "filter",
      "tool": "BUD_MATCH",
      "params": {"max_price": 200, "amenities": ["breakfast"]},
      "depends_on": ["s2"]
    },
    {
      "id": "s4",
      "verb": "format",
      "tool": "PRESENT",
      "params": {"top": 3},
      "depends_on": ["s3"],
      "output_key": "hotel_options"
    }
  ],
  "out_fmt": "human",
  "state": "PENDING"
}
```

### 14.2 Error Response

```json
{
  "proto_ver": "1.0.0",
  "msg_id": "hotel-search-001-r1",
  "reply_to": "hotel-search-001",
  "ts": "2026-05-09T00:00:06Z",
  "sender": "EXECUTOR_AGENT",
  "recipient": "INTERFACE_AGENT",
  "g": "FIND_HOTEL",
  "plan": [
    {
      "id": "s1",
      "verb": "fetch",
      "tool": "HISTO_API",
      "params": {"loc": "Queenstown", "dates": ["2026-07-20", "2026-07-25"]}
    }
  ],
  "out_fmt": "raw",
  "state": "FAILED",
  "error": {
    "code": "TOOL_INVOCATION_FAILED",
    "message": "HISTO_API returned HTTP 503",
    "step_id": "s1",
    "retryable": true,
    "details": {"http_status": 503}
  }
}
```

## 15. Migration from v0.1

### 15.1 Breaking Changes

- `proto_ver`, `msg_id`, `ts`, `sender`, and `state` are now mandatory.
- `plan` steps are object-based in canonical v1 format.
- `state` values expanded and transition rules are normative.
- `result` and `error` envelopes are now standardized.

### 15.2 Recommended Migration Path

1. Add an adapter to normalize tuple steps to object steps.
2. Enforce v1 schema validation in non-blocking mode.
3. Switch emitters to v1 object steps.
4. Enable strict validation and reject non-v1 payloads.

## End of Specification
