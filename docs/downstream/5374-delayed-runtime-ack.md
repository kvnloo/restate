# Downstream regression candidate: runtime acknowledgement latency

Status: **HOLD**. Test-only candidate, not compiled or executed. No production fix,
no observed RED/GREEN transition, and no claim of readiness to merge.

## Provenance and scope

- Upstream: https://github.com/restatedev/restate/issues/5374
- Base: `752283ec6600c45ed8a60c97d132508c5a45f3af`
- Original race report and timing evidence: **muhamadazmy**.
- Protocol-state invariant / delayed-ack test suggestion: **kvnloo**,
  https://github.com/restatedev/restate/issues/5374#issuecomment-5975615453
- Reviewed prior timer correction: https://github.com/restatedev/restate/pull/4604
  (keeps a persistent inactivity timer; does not track pending runtime ACKs).

The new paused-time test invokes the actual V7 protocol runner: accept a requested
run-completion proposal, hold the ACK for 200 ms against a 100 ms inactivity
budget, deliver the matching ACK, and require the resumed SDK inactivity timer
to terminate the still-idle stream. It uses no HTTP server or real-time sleeps.
The current runner is expected to terminate before the delayed ACK; that
expectation is source-based and has not been confirmed by executing the test.

## Validation on 2026-10-10

Rust 1.98.1 installed successfully. Builds used two jobs, no debug information,
and no incremental compilation.

- `cargo check -p restate-invoker-impl --tests`: blocked during dependency
  resolution by a CONNECT tunnel 403 fetching `restatedev/rust-sasl`.
- `cargo check --offline -p restate-invoker-impl --tests`: blocked because that
  pinned dependency is absent locally.
- `cargo nextest run --offline -p restate-invoker-impl --all-features runtime_ack_tests`:
  not run; `cargo-nextest` is not installed.
- `cargo fmt --all -- --check`: passed.
- `cargo clippy --offline --all-features --all-targets --workspace -- -D warnings`:
  blocked by the same absent dependency.

The pinned dependency is `sasl2-sys` from
`https://github.com/restatedev/rust-sasl`, revision
`d3e87c54e87327ddbd226881d7546ab7696bf683`. This is a baseline environment
blocker, not an observed regression failure. No raw build logs are included.

## Next steps

1. Use an authorized environment with the pinned dependencies and cargo-nextest.
2. Compile and run this regression against the base, recording the actual RED.
3. Decide a bounded runtime-ACK waiting policy before changing production code.
   Disabling inactivity indefinitely or silently reusing the SDK abort timeout
   would introduce unreviewed semantics.
4. Match ACK identifiers, account for multiple outstanding requests, preserve the
   V7 gate, and test command ACKs plus missing/unrelated ACK controls.
5. Run GREEN, format, lint and integration checks before promotion.

Issue #5448 is deliberately deferred: no object-offload design or benchmark result
is claimed without an executable test and measurement environment.
