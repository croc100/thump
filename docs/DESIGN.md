# Thump — Design (how it works)

This is the runtime/operational design: how a raw hardware signal becomes a
safe evacuation. ROADMAP.md says *when* we build things; this says *how the
machine runs*.

---

## 1. The pipeline (life of a signal)

```
 source            agent                controller
 ───────           ─────                ──────────
 DCGM      ┐
 Redfish   ├─▶ collect ─▶ normalize ─▶ ┌─▶ ② Predict ─▶ ③ Decide ─▶ ④ Actuate
 SMART     ┘   (raw)      (Signal)     │       │            │           │
 EDAC/MCE                              │       ▼            ▼           ▼
                                       │   hazards      verdict      Slurm/K8s
                                       │                              drain
                                       └──────────── ⑤ Learn ◀────────────┘
                                              (prediction + real outcome → label)
```

One sentence: **agents turn vendor-specific telemetry into a uniform `Signal`
stream; the controller scores failure hazard, grades it against action risk,
and either acts, recommends, or just watches — then records whether it was
right.**

---

## 2. The core data type — `Signal`

Everything downstream speaks one language. Collectors are the only place that
knows about DCGM vs Redfish vs SMART.

```go
type Signal struct {
    Node      string            // node id (warren member)
    Source    Source            // DCGM | REDFISH | SMART | EDAC | KMSG
    Kind      Kind              // ECC_CORRECTABLE | GPU_XID | NVLINK_FLAP |
                                // FAN_RPM | INLET_TEMP | PSU | SMART_REALLOC ...
    Value     float64           // numeric reading (rate, temp, count)
    Severity  Severity          // INFO | WARN | CRIT (vendor-mapped)
    Labels    map[string]string // gpu_index, dimm, disk, psu_id ...
    Time      time.Time
}
```

Collectors are **event-driven first** (Redfish event subscription, DCGM field
watch), **polling only as fallback** — so we don't hammer thousands of BMCs.

---

## 3. ② Predict — rule-based hazard, explainable

We do **not** start with a black-box ML model. We start with named detectors,
each of which is auditable and emits a *hazard*: "this evidence says the node
will fail within Δt, with this strength."

```go
type Hazard struct {
    Rule       string    // "ecc_acceleration", "xid_fatal", "smart_reallocate"
    Strength   float64   // h ∈ [0,1]  — how strongly this predicts failure
    Confidence float64   // c ∈ [0,1]  — how much we trust this evidence
    Horizon    time.Duration // estimated time-to-failure
    Evidence   []Signal  // the raw signals that fired it (for the audit trail)
}
```

### Example detectors

- **`xid_fatal`** — certain GPU Xid codes (e.g. uncorrectable ECC, GPU fallen
  off the bus) are near-certain hard failure → `strength≈0.95, confidence≈0.99,
  horizon≈minutes`.
- **`ecc_acceleration`** — correctable ECC rate is *accelerating* (2nd
  derivative > 0), not just high. A DIMM trending toward uncorrectable →
  `strength≈0.6, horizon≈hours`.
- **`smart_reallocate`** — reallocated/pending sector count rising →
  `strength≈0.5, horizon≈days`.
- **`thermal_runaway`** — inlet temp rising while fan RPM already maxed →
  `strength≈0.7`.

Each detector is ~20 lines of Go over a short window of `Signal`s. New failure
modes = new detectors, no retraining.

### Combining hazards → P(fail)

Detectors are largely **independent** ("any one of these can independently doom
the node"), so we combine with **noisy-OR**:

```
P(fail within Δt) = 1 − Π (1 − hᵢ · cᵢ)      for all hazards i
```

- One strong hazard (xid_fatal) alone pushes P high.
- Several weak hazards (mild ECC + warm + one SMART tick) *stack* — together
  they cross the line even though none alone would.
- Each hazard's `confidence` discounts noisy evidence, so a flaky BMC reading
  can't dominate.

Output of Predict:

```go
type Prediction struct {
    Node       string
    P          float64   // P(fail within horizon)
    Horizon    time.Duration
    Hazards    []Hazard  // the *why* — always attached
}
```

> The `Hazards` slice is the explainability. Every prediction can answer "why?"
> with the exact signals that drove it — required for operator trust and the
> audit trail.

---

## 4. ③ Decide — confidence × blast-radius matrix

This is the safety core. A prediction never directly causes an action. It's
graded against **how destructive the proposed action is.**

```go
type Action int
const (
    Observe   Action = iota // do nothing, keep watching
    Cordon                  // stop NEW work landing — fully reversible
    Drain                   // evict running work        — reversible
    Migrate                 // live-migrate VM            — reversible
    // (reboot/power/disk = NEVER auto; recommend only)
)

type Verdict struct {
    Action   Action
    Mode     Mode    // AUTO | RECOMMEND | OBSERVE
    Reason   string  // human-readable, from Hazards
}
```

Decision logic (conceptually a 2-axis table):

```
                       P low        P mid          P high
 reversible action   observe      recommend       AUTO
 (cordon/drain)
 destructive action  observe      observe        RECOMMEND   (never auto)
```

Plus declarative **policy guardrails** the customer sets:

- `min_auto_confidence` — only auto-act above this (starts very high, lowers as
  Learn proves out).
- blackout windows ("no drains during the 02:00 batch").
- protected workloads ("never touch job X / this priority class").
- rate limits ("≤ N concurrent drains per fleet" — never evacuate so many nodes
  you cause the outage yourself).

### The asymmetry, made concrete

- AUTO only ever runs **reversible** actions → worst case = wasted capacity,
  self-healing.
- Below the bar → **RECOMMEND** (one-click human approval).
- Miss entirely → falls back to today's reactive ping-fail loop. No worse.

---

## 5. ④ Actuate — safe adapters

Thin, well-tested adapters that translate a `Verdict` into the orchestrator's
own primitives:

- **Slurm**: `scontrol update nodename=… state=DRAIN reason="thump:<rule>"`
  (or slurmrestd). First wedge.
- **K8s**: `client-go` cordon + eviction API (respects PodDisruptionBudgets).
- Every action: **dry-run capable**, fully logged, and **undo-able** (uncordon
  / resume) since we restrict auto to reversible ops.

---

## 6. ⑤ Learn — close the loop

For every `Prediction`, we later record what actually happened:

```go
type Outcome struct {
    PredictionID string
    Actual       Result   // FAILED | HEALTHY_AFTER_TIMEOUT
    ActionTaken  Action
    OutageAvoided bool    // did the evacuation prevent a real job kill?
}
```

This produces the trust report (Phase 0) — *precision, false-positive rate,
outages avoided* — and, anonymized, feeds the central **cross-fleet label
aggregator** (CF Workers + D1). More fleets → better-calibrated hazard weights
→ new customers get good predictions on day 1. That's the compounding moat.

---

## 7. Why this shape

- **Rules before ML** → explainable, trustworthy, debuggable from day 1; ML
  refines hazard weights later without changing the pipeline.
- **One `Signal` type** → adding a vendor/source touches only a collector.
- **Decide separated from Predict** → safety policy evolves independently of
  detection accuracy.
- **Reversible-only auto** → the whole system can never make things worse than
  the status quo.
