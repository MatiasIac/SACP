# Semantic Agent Communication Protocol (SACP)

SACP is a formal specification and practical messaging protocol for enabling modular, multi-agent systems powered by LLMs or other autonomous reasoning components. It defines a simple, strict, and extensible way for agents to:

* Interpret semantic goals
* Share structured execution plans
* Dynamically spawn new agents when capability gaps are found
* Exchange compact, machine-readable messages with predictable semantics

## Key Features

* Structured `[verb, tool, params]` task planning format
* Support for dynamic agent discovery and embedding-based capability matching
* Built-in message lifecycle control, traceability, and error handling
* Designed to reduce verbosity and maximize agent chaining
* Model-agnostic (usable with GPT, Claude, Mistral, etc.)

## Example Use Cases

* Travel planning (e.g., hotel search, visa prediction)
* Document summarization and analysis
* Multi-step code generation workflows
* AI-based research and knowledge retrieval agents

## Repository Structure

* `specification.md`: Full SACP protocol definition
* `whitepaper.md`: Introductory overview and architectural rationale
* `examples/`: Sample SACP messages and agent behaviors
* `templates/`: LLM prompt templates for compliant agent creation
* `schema/`: JSON schemas for validating message structure

## License

MIT License (see LICENSE file).

## Status

SACP is currently in draft version `v0.1`. Contributions, critiques, and integrations are welcome via GitHub Issues.

---

For more, see `specification.md` or the white paper.
