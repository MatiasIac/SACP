# SACP Conformance Tests Checklist

Version target: `1.0.0`  
Status: Operational checklist for implementation certification

## Scope

This checklist validates both:
- Schema conformance (`schema/sacp.schema.json`)
- Runtime conformance (state machine, execution semantics, policy behavior)

Passing schema validation alone is not sufficient for protocol conformance.

## How To Use

1. Build a test harness that can submit messages and capture outputs/transitions.
2. Execute all checklist items as automated tests in CI.
3. Treat unchecked items as non-conformant.
4. Re-run the suite on every protocol or orchestrator change.

## A. Schema Validation

- [ ] `A01` Accepts a minimal valid request with required fields: `proto_ver`, `msg_id`, `ts`, `sender`, `g`, `plan`, `state`.
- [ ] `A02` Rejects message missing each required field (one test per field).
- [ ] `A03` Rejects unknown top-level properties (`additionalProperties=false`).
- [ ] `A04` Rejects `proto_ver` other than `1.0.0`.
- [ ] `A05` Rejects invalid `ts` format.
- [ ] `A06` Rejects non-UTC timestamp not ending in `Z`.
- [ ] `A07` Rejects empty `plan`.
- [ ] `A08` Rejects step missing required fields: `id`, `verb`, `tool`, or `params`.
- [ ] `A09` Rejects unknown plan-step properties.
- [ ] `A10` Rejects `on_error=fallback` step without `fallback_step`.
- [ ] `A11` Rejects `state=DONE` without `result`.
- [ ] `A12` Rejects `state=FAILED` without `error`.
- [ ] `A13` Rejects message containing both `result` and `error`.
- [ ] `A14` Accepts `result` object only when structurally valid.
- [ ] `A15` Accepts `error` object only when structurally valid.

## B. State Machine Conformance

- [ ] `B01` Allows `PENDING -> ACCEPTED`.
- [ ] `B02` Allows `ACCEPTED -> IN_PROGRESS`.
- [ ] `B03` Allows `IN_PROGRESS -> WAITING`.
- [ ] `B04` Allows `WAITING -> IN_PROGRESS`.
- [ ] `B05` Allows terminal transitions: `IN_PROGRESS -> DONE|FAILED|CANCELLED`.
- [ ] `B06` Rejects invalid transition: `PENDING -> DONE` without required lifecycle path (if your implementation enforces full path).
- [ ] `B07` Rejects any transition out of terminal states (`DONE`, `FAILED`, `CANCELLED`).
- [ ] `B08` Ensures `FAILED` transitions include canonical `error.code`.
- [ ] `B09` Ensures `DONE` transitions include `result.outputs`.

## C. Plan Semantics

- [ ] `C01` Executes independent steps in any order allowed by dependencies.
- [ ] `C02` Blocks a step until all `depends_on` prerequisites succeed.
- [ ] `C03` Rejects plan with dependency cycle.
- [ ] `C04` Rejects plan with duplicate step ids.
- [ ] `C05` Rejects `depends_on` references to unknown step ids.
- [ ] `C06` Correctly skips dependent steps when prerequisite fails under `fail_fast`.
- [ ] `C07` Records step-level failure and continues independent work under `continue`.
- [ ] `C08` Invokes `fallback_step` exactly once after retries exhausted under `fallback`.

## D. Retry And Timeout Behavior

- [ ] `D01` Retries up to `retry.max_attempts`.
- [ ] `D02` Stops retrying after max attempts and returns terminal error outcome.
- [ ] `D03` Applies `fixed` backoff behavior when configured.
- [ ] `D04` Applies `exponential` backoff behavior when configured.
- [ ] `D05` Enforces `timeout_ms` and emits `TIMEOUT` on timeout.
- [ ] `D06` Marks timed-out step as failed and propagates according to `on_error`.

## E. Output Discipline

- [ ] `E01` Defaults `out_fmt` to `raw` when omitted.
- [ ] `E02` Produces machine-readable output in `raw` mode.
- [ ] `E03` Produces compact machine-readable output in `summary` mode.
- [ ] `E04` Only emits human prose when `out_fmt=human`.
- [ ] `E05` Intermediate agents do not emit prose in `raw` mode.

## F. Error Contract

- [ ] `F01` Emits `error.code` for all failed terminal outcomes.
- [ ] `F02` Uses canonical codes where applicable (`MALFORMED_MESSAGE`, `UNKNOWN_TOOL`, `TIMEOUT`, etc.).
- [ ] `F03` Includes `step_id` when failure is step-scoped.
- [ ] `F04` Sets `retryable` consistently with policy.
- [ ] `F05` Never returns terminal `FAILED` with empty/placeholder error body.

## G. Result Contract

- [ ] `G01` Emits `result.outputs` on `DONE`.
- [ ] `G02` Preserves step output mapping when `output_key` is present.
- [ ] `G03` Includes optional `result.step_results` with correct step ids when emitted.
- [ ] `G04` Never returns terminal `DONE` with missing or invalid `result`.

## H. Compatibility And Versioning

- [ ] `H01` Rejects unsupported major protocol versions.
- [ ] `H02` Handles `1.0.x` payloads consistently (if minor/patch tolerance is implemented).
- [ ] `H03` Producer emits canonical object-form steps.
- [ ] `H04` If tuple adapter exists, adapter mode is explicit and tested separately.

## I. Security And Policy Gates

- [ ] `I01` Rejects unauthenticated sender (if auth layer integrated).
- [ ] `I02` Rejects unauthorized verb/tool use with `AUTHZ_DENIED`.
- [ ] `I03` Logs lifecycle transitions and tool invocations for audit.
- [ ] `I04` Enforces tool allowlist policy per sender/tenant.
- [ ] `I05` Suppresses side effects in `test_mode=true` unless policy override exists.

## J. Observability

- [ ] `J01` Propagates `trace.trace_id` across hops when provided.
- [ ] `J02` Maintains `trace.attempt` semantics across retries.
- [ ] `J03` Emits `trace.metrics.latency_ms` at terminal states (if metrics enabled).
- [ ] `J04` Correlates per-step failures with message-level terminal outcome.

## Minimum Certification Gate

An implementation is considered SACP-conformant for production only when:
- All `A*` tests pass.
- All `B*` tests pass.
- All `C*` tests pass.
- No `D*`, `E*`, `F*`, or `G*` test fails.

`H*`, `I*`, and `J*` are required for production-grade deployments, even if development-mode environments defer them.

## Recommended Fixture Set

Maintain explicit fixtures:
- `valid_minimal.json`
- `valid_full.json`
- `invalid_missing_required_*.json`
- `invalid_unknown_field.json`
- `invalid_step_shape.json`
- `invalid_transition_*.json`
- `plan_cycle.json`
- `plan_unknown_dependency.json`
- `timeout_case.json`
- `retry_case_fixed.json`
- `retry_case_exponential.json`
- `done_without_result.json`
- `failed_without_error.json`

Place fixtures under `tests/fixtures/` and run them in CI for every commit.
