# Fixture Index

This directory contains baseline fixtures referenced by [conformance-tests.md](/conformance-tests.md).

## Expected Schema Outcomes

### Should Pass
- `valid_minimal.json`
- `valid_full.json`

### Should Fail
- `invalid_missing_required_proto_ver.json`
- `invalid_missing_required_msg_id.json`
- `invalid_missing_required_ts.json`
- `invalid_missing_required_sender.json`
- `invalid_missing_required_g.json`
- `invalid_missing_required_plan.json`
- `invalid_missing_required_state.json`
- `invalid_unknown_field.json`
- `invalid_step_shape.json`
- `done_without_result.json`
- `failed_without_error.json`

## Runtime Semantics Fixtures

These may pass schema validation but should be validated by runtime conformance tests:
- `invalid_transition_pending_to_done.json`
- `invalid_transition_terminal_reopen.json`
- `plan_cycle.json`
- `plan_unknown_dependency.json`
- `timeout_case.json`
- `retry_case_fixed.json`
- `retry_case_exponential.json`
