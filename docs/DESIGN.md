# Thump — Design (how it works)

This is the runtime/operational design: how a raw hardware signal becomes a
safe evacuation, and every architectural decision behind it. ROADMAP.md says
*when* we build things; this says *how the machine runs and why it's shaped
this way*.

---

## 0. Thesis

The standard on-prem / cloud failure loop is **reactive**:

```
ping fail  →  check device  →  detect fault  →  remediate
```

A `ping fail` is the *last* signal, not the first. By the time a node stops
answering, its BMC and telemetry have been flashing warnings for hours or days.
For GPU training, one dead node stalls an entire distributed job and forces a
checkpoint rollback worth thousands of GPU-hours.

Thump inverts the loop: **read the leading signal, then evacuate the workload
before the failure happens** — a closed loop of detect → predict → decide →
act → learn.

### The market gap we fill

Signals and alarms are crowded; the *bridge between them* is empty:

| Layer | Who's there | Limit |
|---|---|---|
| Signal source | DCGM, Redfish, DCIM | emits signals only, workload-blind |
| Observability / AIOps | Datadog, Grafana, Prometheus | alarms a human, no closed loop |
| Orchestration | K8s, Slurm, Run:ai | drains only on *manual* input |

Nobody productizes the **"signal → confidence → safe orchestrator action"**
layer in between. That decision+actuation bridge — plus a cross-fleet
fault→outcome dataset nobody else is positioned to hold — is Thump.

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
and either acts, recommends, or watches — then records whether it was right.**

---

## 2. Deployment topology — hybrid

One logical pipeline, three physical units:

```
 [nodes]                    [inside customer fleet]          [central / cloud]
 thump-agent  ──signals──▶  thump-controller    ──anon labels──▶ label-aggregator
 (collect)                  (predict/decide/                     (cross-fleet
                            actuate + storage)                    calibration)
```

- **thump-agent** — one per node, lightweight static Go binary. Collect only.
- **thump-controller** — one per fleet (HA), Go. Predict / Decide / Actuate +
  local storage. **Customer telemetry never leaves this boundary.**
- **label-aggregator** — central, receives **only anonymized fault→outcome
  labels**. Reuses existing CRODE infra: **Cloudflare Workers + D1**.

agent and controller ship as **one binary, two modes** (`thump agent` /
`thump controller`) → a single build/release pipeline.

This hybrid shape is deliberate: it keeps security-sensitive operators
(finance, sovereign AI, on-prem GPU) **deployable** while still building the
cross-fleet data moat. Pure SaaS would lock them out; pure on-prem would
forfeit the moat.

---

## 3. Tech stack

- **Language: Go everywhere** (agent, controller). Matches the ecosystem —
  NVIDIA `dcgm-exporter` is Go (`go-dcgm` bindings), K8s is `client-go`-native,
  and a single static binary deploys to thousands of nodes with no dependency
  hell. Cross-compiles to linux/amd64 + arm64 (Grace) for free.
- **No kernel / machine code.** Every signal is already exposed via a userspace
  interface — we *consume*, never write drivers:

  | Signal | Read from |
  |---|---|
  | GPU (Xid, ECC, NVLink, temp) | NVML / DCGM |
  | BMC (power, fan, temp, SEL) | Redfish (HTTP) + IPMI |
  | Disk | SMART (smartctl / nvme-cli) |
  | CPU/memory errors | EDAC sysfs / rasdaemon / MCE |
  | Kernel events | dmesg / kmsg |

  (An optional eBPF probe for deeper signals is a *later* possibility, not MVP.)
- **ML later, not now.** Prediction is rule-based first (explainable,
  trustworthy). When models arrive in a later phase they only re-weight
  hazards — likely a Python sidecar — without changing the pipeline.

---

## 4. Storage

- **Signal time-series**: if the customer already runs **Prometheus**, scrape
  it (avoid duplicate collection). Otherwise an embedded local store in the
  controller.
- **Decisions / audit log / labels**: embedded **SQLite** in the controller
  (reusing CRODE's SQLite expertise / LiteScope assets).
- **Central**: anonymized labels in **Cloudflare D1**.

---

## 5. The core data type — `Signal`

Everything downstream speaks one language. Collectors are the only place that
knows about DCGM vs Redfish vs SMART — adding a vendor/source touches only a
collector.

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
watch), **polling only as fallback** — so we don't hammer thousands of
(often fragile) BMCs.

---

## 6. ② Predict — rule-based hazard, explainable

We do **not** start with a black-box ML model. We start with named detectors,
each auditable, each emitting a *hazard*: "this evidence says the node will
fail within Δt, with this strength."

```go
type Hazard struct {
    Rule       string        // "ecc_acceleration", "xid_fatal", "smart_reallocate"
    Strength   float64       // h ∈ [0,1]  — how strongly this predicts failure
    Confidence float64       // c ∈ [0,1]  — how much we trust this evidence
    Horizon    time.Duration // estimated time-to-failure
    Evidence   []Signal      // the raw signals that fired it (audit trail)
}
```

### Example detectors

- **`xid_fatal`** — certain GPU Xid codes (uncorrectable ECC, GPU fallen off
  the bus) ≈ near-certain hard failure → `strength≈0.95, confidence≈0.99,
  horizon≈minutes`.
- **`ecc_acceleration`** — correctable ECC rate is *accelerating* (2nd
  derivative > 0), not merely high → `strength≈0.6, horizon≈hours`.
- **`smart_reallocate`** — reallocated / pending sector count rising →
  `strength≈0.5, horizon≈days`.
- **`thermal_runaway`** — inlet temp rising while fan RPM already maxed →
  `strength≈0.7`.

Each detector is ~20 lines of Go over a short window of `Signal`s. New failure
modes = new detectors, no retraining.

### Combining hazards → P(fail)

Detectors are largely **independent** ("any one can independently doom the
node"), so we combine with **noisy-OR**:

```
P(fail within Δt) = 1 − Π (1 − hᵢ · cᵢ)      for all hazards i
```

- One strong hazard (`xid_fatal`) alone pushes P high.
- Several weak hazards (mild ECC + warm + one SMART tick) *stack* — together
  they cross the line even though none alone would.
- Each hazard's `confidence` discounts noisy evidence, so a flaky BMC reading
  can't dominate.

```go
type Prediction struct {
    ID      string
    Node    string
    P       float64       // P(fail within horizon)
    Horizon time.Duration
    Hazards []Hazard      // the *why* — always attached
}
```

> The `Hazards` slice is the explainability. Every prediction answers "why?"
> with the exact signals that drove it — required for operator trust and the
> audit trail.

---

## 7. ③ Decide — confidence × blast-radius matrix (safety core)

A prediction **never directly causes an action.** It is graded against how
destructive the proposed action is.

```go
type Action int
const (
    Observe Action = iota // do nothing, keep watching
    Cordon                // stop NEW work landing — fully reversible
    Drain                 // evict running work     — reversible
    Migrate               // live-migrate VM         — reversible
    // reboot / power / disk = NEVER auto; recommend only
)

type Mode int
const ( AUTO Mode = iota; RECOMMEND; OBSERVE )

type Verdict struct {
    Action Action
    Mode   Mode
    Reason string // human-readable, derived from Hazards
}
```

Decision logic (a 2-axis table):

```
                       P low        P mid          P high
 reversible action   observe      recommend       AUTO
 (cordon/drain)
 destructive action  observe      observe        RECOMMEND   (never auto)
```

### Policy guardrails (declarative, customer-set)

- `min_auto_confidence` — only auto-act above this. **Starts very high; lowers
  as ⑤ Learn proves the numbers.**
- **blackout windows** — "no drains during the 02:00 batch."
- **protected workloads** — "never touch job X / this priority class."
- **rate limits** — "≤ N concurrent drains per fleet" so Thump never evacuates
  so aggressively it causes the outage itself.

### The asymmetry, made concrete

This is the whole safety argument — **"100%" means safety, not prediction
accuracy:**

- **False positive** (drain a node that was fine) → a little wasted capacity,
  self-healing. Recoverable.
- **False negative** (miss one) → falls back to today's reactive ping-fail loop.
  **No worse than the status quo.**
- AUTO only ever runs **reversible** actions → worst case is never destructive.
- We compare to **now (0% predictive accuracy)**, not to a perfect oracle.

> If we're wrong, you're no worse off than today. If we're right, you avoid an
> outage.

---

## 8. ④ Actuate — safe adapters

Thin, well-tested adapters translating a `Verdict` into the orchestrator's own
primitives:

- **Slurm** *(first wedge)*:
  `scontrol update nodename=… state=DRAIN reason="thump:<rule>"` (or
  slurmrestd).
- **K8s**: `client-go` cordon + eviction API (respects PodDisruptionBudgets).
- *Later*: VM live-migrate, LB drain, cloud instance-health adapters.

Every action is **dry-run capable**, fully logged, and **undo-able**
(uncordon / resume) — guaranteed because auto is restricted to reversible ops.

---

## 9. ⑤ Learn — close the loop (the moat)

For every `Prediction` we later record what actually happened:

```go
type Outcome struct {
    PredictionID  string
    Actual        Result // FAILED | HEALTHY_AFTER_TIMEOUT
    ActionTaken   Action
    OutageAvoided bool   // did the evacuation prevent a real job kill?
}
```

This drives two things:

1. **The trust report (Phase 0)** — precision, false-positive rate, outages
   avoided — computed on the customer's *own* fleet, in shadow mode, before any
   action is enabled.
2. **Cross-fleet calibration** — anonymized labels feed the central aggregator
   (CF Workers + D1). More fleets → better-calibrated hazard weights → new
   customers get good predictions on day 1. Big-tech see only their own fleet;
   vendors see signals but not outcomes — **only an independent product can
   hold this cross-fleet label set.** That's the compounding moat.

---

## 10. Who runs it (four faces of one product)

| Actor | What they want | Surface |
|---|---|---|
| **SRE / platform** | pre-failure evacuation, fewer 3am pages | closed-loop core |
| **Vendors / OEMs** | "our hardware emits accurate leading signals" proof | CP / conformance program (later phase) |
| **ISP / IDC / hosting** | multi-tenant fleet health, SLA evidence | multi-tenant + reports |
| **Cloud / Neocloud** | less node churn, preserved capacity | orchestrator adapters |

Vendors are simultaneously customers, **data suppliers, and a distribution
channel** — the lever that fills the cross-fleet dataset fastest.

---

## 11. First wedge — GPU training (Slurm)

Start narrow. GPU training on Slurm is where one dead node stalls a whole
distributed job and the evacuation ROI is overwhelming, and where later GPU
operators have no big-tech-grade fleet automation of their own. CPU fleets,
inference/K8s, and cloud adapters expand from there.

---

## 12. Why this shape (recap)

- **Rules before ML** → explainable, trustworthy, debuggable from day 1; ML
  later only re-weights hazards.
- **One `Signal` type** → adding a vendor/source touches only a collector.
- **Decide separated from Predict** → safety policy evolves independently of
  detection accuracy.
- **Reversible-only auto + guardrails** → the system can never make things
  worse than the status quo.
- **Hybrid deployment** → deployable to sensitive fleets *and* builds the moat.
- **Shadow → recommend → closed-loop** → earn trust with numbers before taking
  control.
