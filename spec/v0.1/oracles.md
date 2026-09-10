# Oracles v0.1

An **oracle** is how a simulation decides truth. Every simulation
declares one. A simulation without an oracle is not a simulation;
it's a Composition invocation with better observability.

## `kind: literal`

The regression-testing default. Declare expected values or predicate
expressions over `outputs.*` and `steps.<id>.*`.

```yaml
oracle:
  kind: literal
  expect:
    - "outputs.result == \"ok\""
    - "outputs.total_cents == 42500"
    - "outputs.duration_ms < 500"
    - "steps.transcribe.text != null"
```

Each string is a tiny expression: identifier chain on the left, an
operator (`== / != / < / <= / > / >= / in / matches`), and a literal
or identifier on the right. `null` is legitimate.

Every expression must evaluate truthy for the run to pass.

## `kind: baseline-run`

Diff against a pinned prior SimulationRun. Any observed difference in
the declared paths is a fail.

```yaml
oracle:
  kind: baseline-run
  baseline: sr_abc123
  compare:
    - outputs.result
    - outputs.total_cents
    - steps.transcribe.text
```

Used when the expected value is "whatever it was last time — flag
me if it changes." Snapshot testing shape.

Baseline Sns are pinned by public_id, so the oracle
survives platform-wide fixture rotation.

## `kind: mathematical`

An invariant that must hold every run regardless of inputs. Fuzz-
testing shape.

```yaml
oracle:
  kind: mathematical
  invariants:
    - "outputs.total_cents == inputs.a_cents + inputs.b_cents"
    - "outputs.charged_at >= inputs.submitted_at"
    - "steps.route.hops.size >= 1"
```

Every invariant expression combines identifiers across `inputs`,
`outputs`, and `steps`. Same operator grammar as `literal`.

Same expression evaluates over every sampled input tuple. A single
failing tuple fails the run.

## `kind: statistical`

Bounds over N runs. Load-testing shape and Monte Carlo shape.

```yaml
oracle:
  kind: statistical
  bounds:
    - "mean(outputs.latency_ms) < 200"
    - "p99(outputs.latency_ms) < 500"
    - "count(outputs.ok == true) / count(*) >= 0.99"
    - "stdev(outputs.result_score) < 5"
```

Aggregation vocabulary in v0.1: `count`, `mean`, `median`, `p50`,
`p90`, `p95`, `p99`, `stdev`, `min`, `max`. Every aggregate is over
observations across the N runs of this simulation's batch.

Requires `run_policy.n > 1`. Framework rejects if not.

## What a v0.1 oracle does NOT supportross-run correlations (`outputs from run i predict outputs from run i+1`).
  Adjacent Monte Carlo pattern; deferred until a real consumer.
- User-defined aggregation functions. Deferred.
- Temporal predicates (`eventually`, `always`, `until`). Property-
  based test-shape; deferred until a real consumer.
- Assertion negation as a first-class kind. Use `!=` in expressions.

Grammar bumps are additive-only. A simulation authored against v0.1
oracles keeps validating under v0.9.
