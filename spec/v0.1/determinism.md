# Determinism v0.1

Reproducibility is the point. A SimulationRun with the same seed +
frozen time + fixtures produces the same outputs + step log + oracle
result, byte-for-byte.

Real-world non-determinism enters through three doors. The envelope
closes each.

## Door 1: RNG

`seed:` in the simulation package pins the executor's `envelope.random`
to a specific Mersenne Twister seed. Any step that pulls random
values from `envelope.random` gets reproducible sequences.

Steps that reach for `SecureRandom.random_number` or `Random.new`
bypass the envelope in v0.1 — deliberately. Full-process RNG
monkeypatching is out of scope; it adds risk without earning value
at v0.1 scale. Followup: land a `random_v` step arg that steps opt
into for seed-aware RNG.

## Door 2: Wall clock

`freeze_time_at:` in the package freezes `Time.now` / `Time.current`
inside the invocation usingActiveSupport's `travel_to`. Everything
downstream — including database `created_at`/`updated_at` timestamps
written during the run — sees the frozen value.

Cleared automatically when the invocation exits, even under raise.

## Door 3: External calls

`stubs:` in the package short-circuit specific step targets with
fixture data instead of hitting the real primitive. Match rules:

- **`{ target: { integration: <slug> } }`** — any step whose target
  is exactly `{ integration: <slug> }` returns the fixture instead
  of hitting the connector.
- **`{ target: { kernel: <slug.op> } }`** — matches by `<slug>.<op>`
  reference the executor already uses.
- **`{ target: { action: <slug> } }`** — matches by per-project
  Action slug.
- **`{ target: { composition: <slug> } }`** — matches by Composition
  slug. Rare, but useful for a simulation that stubs a nested
  composition invocation without descending into it.
- **`{ target: { records: <op> } }`** — matches by records op
  (`list` / `read` / `create` / `update`). Rare in v0.1.

Match is on the single-key target hash alone in v0.1. Args-aware
matching (e.g. "stub Stripe's charge endpoint only when amount > $500")
is a follow-up.

## What the envelope does NOT try to control in v0.1

- Non-determinism inside kernel workers running in Modal subprocesses.The freeze is in-Ruby-process only. Workers get controlled inputs
  (via stubs) but their internal RNG / clock is their own problem.
- LLM sampling. Vendors handle this differently (OpenAI's `seed`,
  Anthropic's temperature=0). The stub protocol replaces the vendor
  call entirely; users who want real-vendor deterministic sampling
  can pass seed args through in the composition step and skip the stub.
- Concurrency and race conditions inside external systems. If the
  target composition invokes an integration that's inherently
  non-deterministic under load, stub it.

## The bet the framework makes

**Envelope + stubs is enough.** Full process-level determinism (VM-
level virtualization, syscall interception, etc.) is out of scope
forever. If your simulation's determinism story needs more than
envelope + stubs, the target is wrong for a Simulation — it belongs
in a different verification tier (property-based fuzzer, chaos
runner, formal verifier).

That bet is validated by external precedent: FoundationDB's DST and
TigerBeetle's simulator use exactly this shape — seeded RNG + frozen
time + stubbed I/O — and findugs traditional testing can't.
