# simulation-protocol (archived)

This spec now lives in Paver, alongside every other primitive spec.
This repo only ever held v0.1; v0.2 through v0.6 were authored in Paver.

- Current version: `simulation@0.6`.
- Schemas: `backend/app/services/primitives/protocols/schemas/simulation-*.json` in `aohara345/paver-platform`.
- Live registration: `GET https://api.get-paver.com/api/v1/protocol_registrations/simulation`.
- Decision: ADR 0042 in `aohara345/paver-platform`.

---

# simulation-protocol

**A protocol for verifying anything.**

A saved, parameterized declaration of *what a simulation is*: a target
(usually a Composition), typed inputs (scalar or distribution), a
determinism envelope (seed, frozen time, stubs), an oracle (what results
are compared against), assertions, and a run policy (N=1 regression to
N=1000 Monte Carlo).

Registered under [Paver's Protocols framework](https://github.com/aohara345/paver-platform/blob/main/docs/adr/0020-protocols-is-a-library-primitive.md)
as the tenth primitive per [ADR 0022](https://github.com/aohara345/paver-platform/blob/main/docs/adr/0022-simulations-primitive.md).

**Status:** v0.1 draft. Grammar will evolve; every change is
additive-only, older simulation packages keep working forever.

## Why a separate protocol

- **Verification is a category of thing agents and platform engineers
  author as an ongoing act** — not code-once-then-forget. Simulations
  accumulate. That's the [primitive test](https://github.com/aohara345/paver-platform/blob/main/docs/primitives.md#the-authoring-suris-the-load-bearing-one).
- **Run tracking is native behavior** — like Skills over Notes. A
  `SimulationRun` is a first-class artifact you ship with a bug report,
  replay in staging, and diff across deploys.
- **Cross-substrate reach** — one grammar tests platform Ruby, user-app
  TSX flows, agent behavior, and vendor integrations, sharing fixtures
  across every project. No traditional test framework does this.

## Spec version

`SPEC_VERSION = "0.1"`

Supported versions: `["0.1"]`

## Repo shape

```
spec/
  v0.1/
    meta-spec.md        # what a simulation IS
    grammar.md          # target / inputs / determinism / oracle / assertions / run_policy
    oracles.md          # types: literal, baseline-run, mathematical, statistical
    distributions.md    # normal / uniform / discrete / categorical / bootstrap
    determinism.md      # seed, time-freeze, stub protocol
    schema.json         # JSONSchema for simulation packages
simulations/
  ffm-signup-through-90-day/
    simulation.yml      # canonical example
fixtures/
  README.md             # v0.1: fixtures live inline in simulation packages
                        # (Fixtures may extract as its own primitive later)
```

## License

MIT (on publish).
