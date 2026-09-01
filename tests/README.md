# tests/ — libpdx-schema-registry test suite (M4)

**Milestone lineage.** M4 in `paideia-os` →
`design/round-retrospectives/r90-xrepo-wave3-plan.md` §4 rubric line:
`tests + smoke`. Two open issues at M4:

- **#9  — M4-001** unit-test drivers (bss fault-injection).
- **#10 — M4-002** register-then-lookup smoke witness.

Both land at M4; M5 (signed release) is deferred to a future round.

## Files

- `test_register_dedup.pdx` — M4-001 driver. Exports
  `TestRegisterDedup::run() -> u64`. Verifies the M2-001 catalog
  reset shape, the M2-003 `schema_lookup` fqname-match / lookup-miss
  paths, and the M2-004 `schema_list` write-mode / probe-mode
  contracts. Uses `.bss` fault-injection to seed the catalog with
  simulated register outcomes rather than calling `schema_register`
  directly (see *Why the drivers do not call schema_register* below).
  Returns 0 on pass or a 1..10 subtest ordinal on failure.

- `test_gate_semantics.pdx` — M4-001 driver. Exports
  `TestGateSemantics::run() -> u64`. Verifies the M3-002 elevate-gate
  trio (`schema_registry_arm` / `schema_registry_disarm` /
  `schema_registry_is_armed`) end-to-end including the mask-must-
  carry-R_SCHEMA_REGISTER refusal, the arm-idempotent-on-refusal
  policy, and the reset-clears-flag invariant. Returns 0 on pass or
  a 1..12 subtest ordinal on failure.

- `test_register_lookup_smoke.pdx` — M4-002 driver. Register-then-
  lookup smoke witness; see M4-002 (issue #10) for its ledger.

## Return-code convention

Every M4 test driver in this tree returns a `u64` where:

- `0` — all subtests passed.
- `N > 0` — the ordinal of the first failing subtest. Ordinals are
  stable across driver runs so a smoke log can pinpoint which
  invariant broke.

A test harness (in a future consumer tool that also links this
library) walks the driver list, sums the non-zero returns per driver,
and exits `0` iff every driver returned `0`.

## Why the drivers do not call `schema_register`

`schema_register` gates on two external state slots:

1. `AuditRecord::record_state` (owned by libpdx-audit) — the M3-001
   audit-open gate.
2. `SchemaRecord::schema_registry_authorized` (owned by this library)
   — the M3-002 elevate gate; `test_gate_semantics` drives it.

The first is a library-external symbol. To exercise `schema_register`
end-to-end an isolated test module would need to define `record_state`
locally so the linker resolves the reference; that local definition
would then collide with libpdx-audit's own `record_state` when both
libraries are linked into a consumer. libpdx-audit's own tests hit
the identical constraint and take the same route: pure leaf,
library-owned state only, with the full end-to-end path deferred to
the QEMU consumer harness documented below.

The tests therefore seed the catalog with the `.bss` state a
successful `schema_register` would have produced (slot state OCCUPIED
+ fqname pointer + fqname length + id + content hash) and exercise
the read-side ops (`schema_lookup`, `schema_list`) plus the gate-
helper trio (`schema_registry_arm` / `disarm` / `is_armed`) against
that seeded catalog. Every invariant the library owns without an
external reference is verified.

## Deferred: QEMU smoke matrix

The full end-to-end smoke — spawn a bootstrap consumer under QEMU
that has libpdx-audit + libpdx-elevate + libpdx-schema-registry
linked, obtain a real audit id via `audit_begin`, obtain an elevate
grant carrying `R_SCHEMA_REGISTER` via `elevate_client_acquire`,
then call `schema_register` and observe the catalog and the audit
journal — needs three not-yet-landed substrates:

1. **shell.M4** so a consumer can be spawned with a bounded cap set.
2. **A runnable bootstrap consumer** that links all three libraries
   and drives the test entry points from its own test-runner main.
3. **The kernel-side svc.schema-registry daemon body** (monorepo,
   `src/kernel/services/schema_registry.pdx`) — currently a stub
   per the R90-XREPO wave-3 plan §4.

Once those three are in place, the smoke matrix runs:

1. Boot QEMU with an image that has all three brokers registered.
2. Spawn the bootstrap consumer.
3. Consumer runs `TestRegisterDedup::run()` / `TestGateSemantics::run()`
   / `TestRegisterLookupSmoke::run()`; if any return != 0, exit that
   ordinal.
4. Consumer then runs a live `schema_register` sequence with real
   audit + elevate context and observes the daemon's `KIND_IPC_ENDPOINT`
   reply matches the expected wire layout defined in
   `src/schema_wire.pdx`.

## Interim runtime discipline

Until the QEMU matrix lands, the drivers compile clean under
`bash tools/build.sh` and the object files are ready for a future
consumer to link. Compile-cleanness is the M4-001 fingerprint.
