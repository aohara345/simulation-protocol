# Grammar v0.1

The canonical shape of a `simulation.yml` file. Every simulation
package under `simulations/<slug>/` conforms to this grammar.

## Top level

```yaml
slug: my-simulation
name: My Simulation
description: |
  One paragraph on what this verifies and why it exists.

target:
  composition: some-composition-slug
  # or one of:
  #   kernel: chart.line
  #   integration: stripe
  #   action: my-project-action-slug
  #   records:  { op: list, collection: leads }

inputs:
  scalar_example:
    kind: string
    default: hello
  distribution_example:
    kind: normal
    mean: 100
    stdev: 15
  choice_example:
    kind: categorical
    values: [a, b, c]
    weights: [0.5, 0.3, 0.2]

determinism:
  seed: 42
  freeze_time_at: "2026-01-01T00:00:00Z"
  stubs:
    - target: { integration: stripe }
      fixture: { charged: true, amount_cents: 42500 }
    - target: { kernel: chart.line }
      fixture: /* PNG bytes as base64 */

oracle:
  kind: literal      # or: baseline-run, mathematical, statisticalect:
    - "outputs.result == \"ok\""
    - "outputs.duration_ms < 500"

run_policy:
  n: 1
  on_failure: stop   # or: cluster, continue

metadata:
  owner: platform-eng
  upstream_ticket: PLT-1234
```

## Section reference

### `target` (required, exactly one child key)

The primitive to invoke. The invocation runs through the ordinary
executor for that primitive; simulations do NOT reimplement execution.

| Key | What it points at |
|---|---|
| `composition` | A Composition slug in the invocation project. |
| `kernel` | A `<kernel>.<op>` reference (e.g. `chart.line`). |
| `integration` | An Integration connector slug. |
| `action` | A per-project Action slug. |
| `records` | A `{op, collection, ...}` records op. |

Multi-target `try:` lists are NOT supported at the simulation level.
The Composition being tested can carry `try:` internally.

### `inputs` (optional)

Typed input declarations. Two flavors:

**Scalar (regression testing):**

| kind | Shape |
|---|---|
| string / int / float / bool | `{ kind:, default: }` |
| enum | `{ kind: enum, values: [...], default: }` |
| file | `{ kind: file, default: pf_...}` |

**Monte Carlo / fuzz):**

| kind | Shape |
|---|---|
| normal | `{ kind: normal, mean:, stdev: }` |
| uniform | `{ kind: uniform, min:, max: }` |
| discrete | `{ kind: discrete, values: [...] }` |
| categorical | `{ kind: categorical, values: [...], weights: [...] }` |
| bootstrap | `{ kind: bootstrap, from: [array of past values] }` |

At invoke time, each distribution is sampled once per run per the
active seed. Same seed → same sequence of samples across runs of
this simulation.

### `determinism` (optional but strongly recommended)

Reproducibility levers. See `determinism.md` for full semantics.

- **`seed`**: an integer. Threaded into the executor as
  `envelope.random`. Any step that pulls from it gets reproducible
  sequences. Steps that ignore it retain their own non-determinism.
- **`freeze_time_at`**: ISO8601 string. `Time.now` / `Time.current`
  inside the invocation return this value.
- **`stubs`**: an array of `{ target, fixture }` pairs. When the
  executor is about to dispatch a step whose target matches a
  declared stub, it returns the fixture instead. Match is on the
  single-key target hash exact.

### `oracle` (required)

How truth is decided. See `oracles.md` for full semantics. Four
kinds in v0.1:

- **`literal`** — expected values or expressions. Simplest form,
  the regression-testing default` is a list of tiny
  expressions over `outputs.*` and `steps.<id>.*`.
- **`baseline-run`** — diff against a pinned prior SimulationRun.
  Any observed difference is a fail. Regressions get flagged
  automatically as behavior drifts.
- **`mathematical`** — an invariant that must hold every run
  regardless of inputs. `∀ inputs: outputs.total == inputs.a +
  inputs.b`. Fuzz-testing shape.
- **`statistical`** — bounds over N runs. `mean(outputs.latency_ms)
  < 200 && p99 < 500`. Load-testing shape.

### `run_policy` (optional)

| Key | Default | Meaning |
|---|---|---|
| `n` | 1 | Number of runs. N=1 is regression; N=1000+ is Monte Carlo. |
| `on_failure` | `stop` | `stop` halts at the first failure, `cluster` groups similar failures, `continue` runs all N regardless. |

### `metadata` (optional)

Free-form provenance. Same field every primitive-instance carries per
Paver's primitive-instance contract.
