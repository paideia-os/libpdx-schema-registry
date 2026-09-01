# libpdx-schema-registry

Schema registry service definition and client library for `paideia-os` semantic pipes.

## Status

M1 scaffolding + M2-001 (SchemaRecord types + catalog storage) landed. Owned by round **R90-XREPO.012** (parent issue: `paideia-os/paideia-os#2000`); sub-issues tracked in this repo.

## What this repo will contain

- The `svc.schema-registry` service definition (opaque 32-byte handles, `PdxFsDirEntry`, `RawByteChunk`, extensibility rule).
- Client-side bindings that consumers link to obtain a `KIND_SCHEMA_HANDLE` from the kernel-side daemon at `src/kernel/services/schema_registry.pdx` (monorepo).
- Tests exercising register/lookup round-trips against a mock kernel.

## Design source of truth

See the wave plan and the initial schema-record design doc in the monorepo:

- `paideia-os/paideia-os` → `design/round-retrospectives/r90-xrepo-wave3-plan.md` (this wave)
- `paideia-os/paideia-os` → `design/terminal/schema-registry.md` (consolidated design, filed under R90-XREPO.012.M5-002)

## License

MIT. See `LICENSE`.
