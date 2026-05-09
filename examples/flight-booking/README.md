# Flight Booking Example Payloads

These payloads are extracted from [flight-booking-implementation.md](/examples/flight-booking-implementation.md) for direct schema/runtime testing.

## Files

- `initial-request.json`: Initial `BOOK_FLIGHT` orchestration request.
- `spawn-request.json`: `SPAWN_AGENT` request for missing airline capability.
- `final-success-response.json`: Final terminal success response (`state=DONE` + `result`).

## Suggested Validation

Validate each file against [sacp.schema.json](/schema/sacp.schema.json) with your preferred JSON Schema validator (Ajv, Python jsonschema, etc.).

Example with Ajv CLI:

```bash
ajv validate -s schema/sacp.schema.json -d examples/flight-booking/initial-request.json
ajv validate -s schema/sacp.schema.json -d examples/flight-booking/spawn-request.json
ajv validate -s schema/sacp.schema.json -d examples/flight-booking/final-success-response.json
```

## Runtime Notes

- `initial-request.json` includes fallback behavior where step `s4` may trigger `s10`.
- `spawn-request.json` is expected to be routed to `SPAWNER_SERVICE`.
- `final-success-response.json` is a terminal message and should not transition to non-terminal states.
