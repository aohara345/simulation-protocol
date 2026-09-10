# Simulation Meta-Spec v0.1

A **simulation** is a saved, named, parameterized declaration of a
verification: what to run, what inputs to run it under, in what
environment, and how to decide whether the run behaved right.

The one-sentence model:

> A simulation targets a Composition (or another primitive), applies
> a determinism envelope, runs it under a declared input distribution
> N times per its run policy, evaluates each run against an oracle,
> and aggregates a pass/fail verdict with observations.

## The seven parts

| Part | Required | What it says |
|---|---|---|
| `target` | yes | Which primitive to invoke (composition / kernel / integration / action / records). |
| `inputs` | no | Typed inputs. Scalars for regression tests; distributions for Monte Carlo / fuzz. |
| `determinism` | no | Seed, freeze_time_at, and stub declarations that make the run reproducible. |
| `oracle` | yes | How truth is decided: literal (`expect`), baseline-run (diff a prior run), mathematical (invariant), statistical (bounds over N). |
| `assertions` | yes-ish | Concrete pass/fail predicates. Everyracle type declares these differently; see oracles.md. |
| `run_policy` | no | N=1 (default) for regression. N>1 for Monte Carlo, fuzz, chaos. `on_failure: cluster / stop / continue`. |
| `metadata` | no | Free-form provenance: why this exists, who owns it, upstream ticket. Same field every primitive-instance carries per [Paver's primitive contract](https://github.com/aohara345/paver-platform/blob/main/docs/primitives.md#the-primitive-instance-contract). |

## Invariants

- **Additive-only across versions.** A simulation authored under v0.1
  keeps validating under v0.9. Grammar bumps only add keys; removing
  a key requires a new major version.
- **The target is a primitive reference, not code.** Simulations
  target existing primitives — they don't reimplement execution.
- **Oracle is required.** A simulation without an oracle is not a
  simulation; it's an ordinary invocation with better observability.
  That's a Composition invocation, not a Simulation. Guard this
  boundary jealously.
- **Determinism is opt-in but strongly recommended.** Without a seed
  + time-freeze, results aren't reproducible; a failing run can't be
  replayed with fidelity. The framework does not force determinism
  because some legitimate simulations exercise real-world non-
  determinism (aaos test explicitly wants random failures). But
  the default posture is "seed everything."
- **Runs persist as SimulationRun rows** in the running Paver
  substrate. The framework guarantees this observability shape;
  it's the native behavior that lifts Simulations above Compositions.

## Coordination shape

Under Paver's [primitive taxonomy](https://github.com/aohara345/paver-platform/blob/main/docs/primitives.md#coordination-shapes), Simulations is
**Vocabularies** (registered under the Protocols framework alongside
WDM and Engagements). The Meta-Spec is Paver-external substrate;
per-project simulation packages are Meta-Spec-conformant instances.

## What this spec does NOT do

- **Not a test runner.** The framework hosts simulations; the runner
  is [`Primitives::Simulations::Executor`](https://github.com/aohara345/paver-platform) inside Paver.
  Other runtimes (a Cursor session, a CI worker, a Modal function)
  are welcome to host their own runners so long as they honor the
  determinism envelope and record SimulationRun rows.
- **Not a stub library.** The determares stubs;
  the stub *content* lives elsewhere (inline in the simulation
  package at v0.1, potentially a Fixtures primitive later).
- **Not a UI.** The `Primitives::Simulations` facade exposes
  find/list/slugs + Executor invocation. Everything else composes on
  top.
