# Semantic Agent Communication Protocol (SACP) - Specification v0.1

## Overview

The Semantic Agent Communication Protocol (SACP) is a compact, structured language designed for communication between autonomous LLM-based agents. It emphasizes machine efficiency, semantic clarity, and modularity. Only the initiating agent interacts with humans; all other agents use this protocol to interpret tasks, execute plans, and return results.

## Design Principles

* **Compactness**: Minimize verbosity to reduce transmission and parsing costs.
* **Semantics-first**: Actions and goals are conveyed by meaning, not syntax.
* **Non-human-readable**: Internal messages are not optimized for readability.
* **Composable**: Agents can construct, modify, and forward messages.
* **Strict schema adherence**: All agents validate messages before processing.

---

## 1. Lexical Components

* `GOAL`: A string or token representing the overall objective.
* `CTX`: Object containing contextual metadata (user, environment).
* `PLAN`: Ordered array of actions to be executed.
* `ACT`: Single action instruction.
* `OUT_FMT`: Output format requested (`human`, `raw`, `summary`).
* `STATE`: Optional message state (`PENDING`, `IN_PROGRESS`, `DONE`, `FAILED`).
* `MSG_ID`: Unique identifier for message tracking.
* `REPLY_TO`: Optional ID of message this responds to.
* Aliased Keys: Abbreviations are allowed to reduce verbosity.

  * e.g., `bdg` = `budget`, `loc` = `location`, `am` = `amenities`

---

## 2. Syntax Rules

Messages follow a strict schema. Example (machine-oriented format):

```json
{
  "msg_id": "abc123",
  "g": "FIND_HOTEL",
  "ctx": {
    "loc": "Paris",
    "in": "06-01",
    "out": "06-05",
    "bdg": 150,
    "am": ["wifi", "bf"]
  },
  "plan": [
    ["s", "H_API", {"c": "Paris", "p_max": 150}],
    ["r", "RANK", {"c": ["p", "rt"]}],
    ["p", "PRES", {"n": 5}]
  ],
  "out_fmt": "human",
  "state": "PENDING"
}
```

* Supports `msg_id` and `reply_to` for chaining messages.
* Supports parallel and join structures in plans (optional):

```json
"plan": [
  {"parallel": [ ["fetch", ...], ["predict", ...] ]},
  ["join", "MERGE", {}]
]
```

---

## 3. Semantics

* `g` (GOAL): Defines the task context.
* `ctx`: Supplies environment and user-specific constraints.
* `plan`: List of instructions to be executed in order.
* `act` (implied within `plan`): Defined as `[verb, tool, params]`
* `out_fmt`: Only present if output must be interpreted by a human.
* `state`: Used to track execution status.

---

## 4. Operational Directives

### Output Discipline for Spawned and Intermediate Agents

Agents must respond with structured, machine-readable formats unless explicitly instructed to do otherwise. In particular:

* The default output format is `raw`.
* Only the Interface Agent, Formatter Agent, or agents specifically instructed via `out_fmt = human` may return human-readable natural language.
* All intermediate or dynamically spawned agents should assume `out_fmt = raw` unless otherwise declared.
* The orchestrator must enforce this rule when composing the `plan`, and should reject agent responses that violate it.

This ensures:

* Low-latency, efficient message exchange
* Consistent chaining of agent outputs
* Predictable formatting for downstream consumers
  Each agent must:
* Validate message schema
* Interpret `g`, `ctx`, and `plan`
* Execute or delegate each `act`
* Track or update `state`
* Return a valid SACP message, unless `out_fmt = human`

Error handling:

* Malformed messages return `ERR` object with reason.
* Missing plan steps return `REQ_CLARIFY`.
* Recovery rules should be applied for failed steps if fallback options are configured.

---

## 5. Extensibility

* New verbs and tools can be registered with a shared schema.
* Versioning supported via `proto_ver` field.
* Nested messages allowed via `subtask` field.
* Messages may support `test_mode` for dry-run or simulation.

---

## 6. Example Message

```json
{
  "msg_id": "task-9823",
  "g": "SUMMARIZE_FEEDBACK",
  "ctx": {"lang": "en", "test_mode": true},
  "plan": [
    ["extract", "FEEDBACK_DOC", {"range": "Q2"}],
    ["analyze", "SENTIMENT", {}],
    ["generate", "SUMMARY", {"length": "short"}]
  ],
  "out_fmt": "human",
  "state": "PENDING"
}
```

---

## 7. Specification Usage

This specification must be provided to any LLM intended to:

* Interpret SACP messages
* Generate or modify plans
* Act as part of a multi-agent workflow

The LLM should be instructed to reject any non-SACP message unless explicitly configured otherwise.

---

## 8. Message Validator

To enforce schema integrity, a validator service or agent should:

* Check for required fields: `g`, `plan`
* Validate structure of `plan`: each step must be `[verb, tool, params]`
* Verify param types (e.g., strings, arrays, numbers)
* Check optional lifecycle fields (`msg_id`, `state`, `reply_to`)
* Return either a `VALID` response or an `ERR` object with error code and reason

Example error:

```json
{
  "status": "ERR",
  "code": "MALFORMED_PLAN",
  "details": "Missing parameters for action step 2"
}
```

---

## 9. Example Agents

### Interface Agent (Agent A)

* Receives user prompt
* Parses intent, constraints, and goals from natural language
* Queries or compiles verb/tool library
* Constructs complete SACP-compliant message
* Assigns specialized subgoals to subordinate agents
* Collects results and optionally builds final human output

### Planner Agent (Agent B)

* Receives abstract goal or partial plan
* Expands and optimizes `plan` steps based on available tools
* Verifies task feasibility against tool capabilities
* Returns enhanced SACP message

### Executor Agent (Agent C)

* Executes each `act` by calling APIs, running internal logic, or invoking further agents
* Aggregates raw data or intermediate responses
* Returns machine-structured results

### Formatter Agent (Agent D)

* Receives final data and presentation requirements
* Applies formatting, sorting, and localization
* Outputs final human-readable result

---

## 10. Verb/Tool Library

To ensure clarity across agents, verbs/tools should be registered with structured metadata:

### Verb Definition

Each verb must define:

* `name`: Identifier (e.g., "search", "generate")
* `description`: What the action performs
* `inputs`: Expected input fields
* `outputs`: Result type or structure

Example:

```json
{
  "name": "generate",
  "description": "Creates a new textual or code artifact.",
  "inputs": ["type", "length", "topic"],
  "outputs": "string"
}
```

### Tool Definition

Each tool must define:

* `id`: Short name used in plans
* `description`: Capability provided by the tool
* `interface`: Optional API/command call reference
* `supported_verbs`: Which verbs can operate with this tool

Example:

```json
{
  "id": "H_API",
  "description": "External API to search for hotels by filters",
  "interface": "https://api.hotelprovider.com/docs",
  "supported_verbs": ["search"]
}
```

If the Interface Agent is tasked with generating the initial SACP message from a human prompt, it must:

* Parse the user's intent and extract key parameters (location, date, budget, etc.)
* Query or retrieve relevant tool definitions
* Assemble appropriate `plan` steps using known verb-tool pairs
* Chain specialized agents (e.g., statistical predictor, filter agent, formatter) to fulfill the plan

Agents encountering unknown verbs/tools or mismatched verb-tool pairs must return an `ERR` response.

---

## 11. Appendix A: Dynamic Capability Registry

The registry acts as a centralized or distributed index of all available agents and their functional capabilities. It is the primary lookup mechanism for the orchestrator to determine which agents can fulfill specific tasks.

### Purpose

The registry allows the orchestrator to:

* Determine whether a valid agent exists for a planned step.
* Discover input/output schemas for tools and verbs.
* Match semantically similar capabilities.
* Trigger fallback mechanisms if no suitable match is found.

### Minimum Required Fields Per Agent

```json
{
  "agent_name": "PRICE_PREDICTOR",
  "supported_verbs": ["predict"],
  "supported_tools": ["PRC_EST"],
  "inputs": ["ctx.history", "method", "window"],
  "outputs": ["predictions"],
  "description": "Provides statistical predictions on future hotel prices"
}
```

### Optional Fields

* `embedding`: Vector used for semantic similarity fallback.
* `tags`: Labels to categorize agents (e.g., "travel", "legal", "financial").
* `tool.interface`: Reference to API or local function.
* `confidence_score`: Optional reliability metric.
* `agent_location`: Internal/external execution endpoint.
* `version`: Agent definition version.

### Example Registry Query

```json
{
  "query": {
    "verb": "predict",
    "inputs": ["history"],
    "output_type": "predictions"
  }
}
```

If no match is found, the orchestrator may trigger a `SPAWN_AGENT` message as defined in Appendix D.

---

To support extensible and dynamic multi-agent systems, agents can register their capabilities in a centralized or distributed registry. The orchestrator can query this registry to discover suitable agents for task execution.

### Agent Registration Format

```json
{
  "agent_name": "PRICE_PREDICTOR",
  "supported_verbs": ["predict"],
  "supported_tools": ["PRC_EST"],
  "inputs": ["ctx.history", "method", "window"],
  "outputs": ["predictions"],
  "description": "Provides statistical predictions on future hotel prices"
}
```

### Example Query

```json
{
  "query": {
    "verb": "predict",
    "inputs": ["history"],
    "output_type": "predictions"
  }
}
```

This enables orchestration over dynamically changing sets of agents and supports plug-and-play extension.

---

## 12. Appendix B: Embedding-Based Agent Selection

To enhance flexibility in selecting agents for user-defined goals, an embedding-based similarity search can be used.

### Agent Description Embedding

Each agent publishes a semantic description encoded into an embedding:

```json
{
  "agent": "PRICE_PREDICTOR",
  "embedding": [0.012, -0.135, 0.246, ...],
  "description": "Performs forecasting on pricing data using linear regression"
}
```

### Goal-to-Agent Matching

The orchestrator encodes the user's request and performs vector similarity against the agent index. This allows selecting agents even without exact matching of predefined verbs/tools.

Applications include:

* Multi-domain orchestration
* General-purpose interfaces
* Learning-based plan generation and routing

---

## 13. Appendix C: Agent Prompt Template for LLM Integration

To simulate or deploy a specific agent (e.g., within Claude, GPT, or another LLM), use the following prompt structure:

### Template:

```
You are an autonomous agent operating under the SACP protocol.

### Agent Role: <AGENT_NAME>

### Role Definition:
- You respond only to SACP-compliant messages.
- You only execute actions that include your assigned verb(s) and tool(s).
- All other messages must be rejected with an ERR response.
- You must follow the output format defined by `out_fmt`.

### Supported Verb(s):
- <verb>

### Supported Tool(s):
- <tool>

### Behavior:
- Validate the message structure.
- Parse and use relevant fields from `ctx` and `plan`.
- Return either a valid SACP output or a protocol-compliant error.

### Message Input:
<Insert SACP JSON message here>

### Instruction:
Process this message strictly according to the rules of your defined role and the SACP protocol.
```

### Example Usage:

Fill in:

* `AGENT_NAME = PRICE_PREDICTOR`
* `verb = predict`
* `tool = PRC_EST`

This prompt ensures that a foundation model behaves predictably and consistently within its designated function.

---

## 14. Appendix D: Dynamic Agent Instantiation

The SACP protocol may support dynamic agent creation when no suitable agent exists in the current registry to fulfill a user’s intent.

### Trigger Scenario

If the orchestrator fails to match a requested `verb` or `tool` from a user prompt to any existing agent, it may emit a special `SPAWN_AGENT` message to a system-level meta-agent or service that handles agent instantiation.

### Spawn Request Message

```json
{
  "g": "SPAWN_AGENT",
  "ctx": {
    "missing_capability": "visa_requirements_check",
    "input_hints": ["citizenship", "destination_country", "travel_dates"],
    "output_expectation": "visa_required: true/false"
  },
  "plan": [
    ["define", "VISA_QUERY", {
      "description": "Checks visa requirements for a citizen traveling to a country",
      "inputs": ["nationality", "destination"],
      "outputs": ["visa_required"]
    }],
    ["instantiate", "VISA_AGENT", {
      "tool": "VISA_QUERY",
      "verb": "check"
    }]
  ],
  "out_fmt": "raw"
}
```

### Listener Role

A dedicated spawning service listens for `SPAWN_AGENT` messages and:

* Generates a new agent definition following the SACP prompt template
* Registers the agent in the capability registry
* Returns a confirmation or reference message to the orchestrator

### Retry Logic

Once registered, the orchestrator retries the original plan or re-generates a revised one with the new agent now available.

### Use Cases

* Expanding into new domains on demand (e.g., travel, healthcare, legal checks)
* Supporting semi-autonomous agent growth from user needs
* Enhancing general-purpose orchestrators with self-adaptive behavior

---

## End of Specification
