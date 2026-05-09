# Semantic Agent Communication Protocol (SACP)

SACP is a transport-agnostic protocol for orchestrating modular multi-agent systems powered by LLMs or other autonomous components.

It defines a strict, machine-first message contract for:
- Goal expression
- Structured execution planning
- Lifecycle state tracking
- Error and retry control
- Capability-based delegation and agent spawning

## Status

SACP is now specified as **v1.0.0 (stable)** for production implementations.

## Why SACP

- Compact JSON envelope with predictable semantics
- Strict schema validation and state transition discipline
- Output format control (`raw`, `summary`, `human`)
- Dependency-aware step execution (`depends_on`)
- Extensibility without breaking the core protocol

## Canonical Message Shape (v1.0.0)

```json
{
  "proto_ver": "1.0.0",
  "msg_id": "task-001",
  "ts": "2026-05-09T00:00:00Z",
  "sender": "INTERFACE_AGENT",
  "recipient": "EXECUTOR_AGENT",
  "g": "SUMMARIZE_FEEDBACK",
  "ctx": {"lang": "en"},
  "plan": [
    {
      "id": "s1",
      "verb": "extract",
      "tool": "FEEDBACK_DOC",
      "params": {"range": "Q2"}
    },
    {
      "id": "s2",
      "verb": "analyze",
      "tool": "SENTIMENT",
      "params": {},
      "depends_on": ["s1"]
    }
  ],
  "out_fmt": "raw",
  "state": "PENDING"
}
```

## Repository Structure

- `specification.md`: Normative SACP v1.0.0 specification
- `schema/sacp.schema.json`: JSON Schema (draft 2020-12) for message validation
- `examples/`: Valid example messages
- `templates/`: Prompt template for creating SACP-compliant agents
- `whitepaper.md`: Architecture rationale, design decisions, and adoption guidance

## Compatibility Notes

- v0.1 tuple step syntax (`[verb, tool, params]`) is deprecated.
- v1.0.0 canonical `plan` steps are object-based.
- Consumers may support a compatibility adapter, but producers should emit v1.0.0 messages.

## License

MIT License (see [LICENSE](LICENSE)).
