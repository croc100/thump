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

This hybrid shape keeps security-sensitive operators (finance, sovereign AI,
on-prem GPU) **deployable** while still building the cross-fleet data moat.
Pure SaaS would lock them out; pure on-prem would forfeit the moat.

---

## 3. Tech stack

- **Language: Go everywhere** (agent, controller). Matches the ecosystem —
  NVIDIA `dcgm-exporter` is Go (`go-dcgm`), K8s is `client-go`-native, and a
  single static binary deploys to thousands of nodes with no dependency hell.
  Cross-compiles to linux/amd64 + arm64 (Grace) for free.
- **No kernel / machine code.** Every signal is already exposed via a userspace
  interface — we *consume*, never write drivers:

  | Signal | Read from |
  |---|---|
  | GPU (Xid, ECC, NVLink, temp) | NVML / DCGM |
  | BMC (power, fan, temp, SEL) | Redfish (HTTP) + IPMI |
  | Disk | SMART (smartctl / nvme-cli) |
  | CPU/memory errors | EDAC sysfs / rasdaemon / MCE |
  | Kernel events | dmesg / kmsg |

  (An optional eBPF probe is a *later* possibility, not MVP.)
- **ML later, not now.** Prediction is rule-based first (explainable). When
  models arrive they only re-weight hazards — likely a Python sidecar — without
  changing the pipeline.

---

## 4. Storage

- **Signal time-series**: if the customer already runs **Prometheus**, scrape
  it (avoid duplicate collection). Otherwise an embedded local store in the
  controller.
- **Decisions / audit log / labels**: embedded **SQLite** in the controller
  (reusing CRODE's SQLite expertise / LiteScope assets).
- **Central**: anonymized labels in **Cloudflare D1**.

---

## 5. The core data type — `Signal` (flexible schema)

A rigid `Value float64` can't carry a Redfish health *state* (a string), a
multi-field PSU reading, or a log line — and a fixed `Kind` enum forces a code
change for every new failure mode. So `Signal` is built to extend without
recompiling the pipeline:

```go
type Signal struct {
    SchemaVer uint16            // schema versioning — forward/backward compat

    Node      string            // node id (warren member)
    Source    string            // "dcgm" | "redfish" | "smart" | "edac" | ...
    Kind      string            // NAMESPACED, e.g. "gpu.xid", "mem.ecc.correctable",
                                 //   "psu.status", "disk.smart.reallocated"
                                 //   — new kinds need no enum/code change

    Value     Value             // typed: number | string | bool | struct
    Unit      string            // "C", "rpm", "count", "errors/s", ""  (explicit units)
    Semantics Semantics         // GAUGE | COUNTER | STATE | EVENT
                                 //   (a COUNTER must be rate-differenced, not read raw)

    Severity  Severity          // INFO | WARN | CRIT (vendor-mapped)
    Quality   Quality           // OK | STALE | ESTIMATED | SUSPECT
                                 //   (flaky BMC reads are tagged, not silently trusted)

    Labels    map[string]string // gpu_index, dimm, disk, psu_id, root_id ...
    Time      time.Time         // observation time
    Ingest    time.Time         // when WE saw it (to detect collection lag/staleness)
}

// Value is a small tagged union so non-numeric and structured readings are
// first-class, not crammed into a float.
type Value struct {
    Type ValueType // NUMBER | STRING | BOOL | STRUCT
    Num  float64
    Str  string
    Bool bool
    Obj  map[string]any // multi-field readings (e.g. full PSU record)
}
```

Why each addition matters:

- **Typed `Value`** → states/logs/structs are no longer lossy.
- **`Semantics`** → the engine never reads a monotonic COUNTER as if it were a
  GAUGE (a classic ECC-rate bug).
- **`Unit`** → no implicit-unit mistakes across vendors.
- **`Quality` + `Ingest`** → suspect/stale data is visible to Predict and can
  be down-weighted instead of poisoning a hazard.
- **namespaced string `Kind` + `SchemaVer`** → new sources/failure modes are
  data, not code; mixed-version agents/controllers interoperate.

Collectors are the **only** place that knows vendor specifics — adding a
source touches one collector.

---

## 6. Collection at scale (the BMC bottleneck)

Naively polling thousands of Redfish/IPMI BMCs from one place melts both the
collector and the (often fragile) BMCs. Design rules:

1. **In-band first, OOB to complement.** The host agent reads GPU/EDAC/SMART/
   kmsg *locally* (cheap, high-frequency). The BMC is used only for what the
   host can't see — PSU, fans, board temps, and **signals from a host that is
   already down**. This slashes BMC load by default.
2. **Event-first, poll as fallback.** Subscribe to Redfish events (BMC pushes
   to a `thump-agent` webhook receiver) and DCGM field-watch. Only fall back to
   polling where a BMC lacks usable eventing.
3. **Distributed, sharded collection.** OOB polling is sharded across
   agents/regional collectors so no single process owns thousands of BMCs.
   Consistent-hash nodes → collectors; rebalance on failure.
4. **Per-BMC token bucket + jitter.** Each BMC has a strict rate limit and
   randomized schedule so we never burst a fleet of BMCs in lockstep.
5. **Risk-adaptive cadence.** Healthy nodes are polled slowly; a node with an
   open hazard is polled fast. Polling budget follows risk, not a flat interval.
6. **Decouple with a queue + backpressure.** Collectors enqueue `Signal`s;
   the engine consumes at its own pace. Collection lag is itself a `Quality`
   signal (stale data is flagged, §5).

Net: BMC traffic scales with *risk and necessity*, not with fleet size × flat
poll rate.

---

## 7. ② Predict — correlation-aware, horizon-aware hazard

We do **not** start with a black-box ML model. We start with named detectors,
each auditable, each emitting a *hazard*: "this evidence says the node will
fail within Δt, with this strength."

```go
type Hazard struct {
    Rule       string        // "ecc_acceleration", "xid_fatal", "smart_reallocate"
    Cause      CauseClass    // THERMAL | MEMORY | GPU | DISK | POWER | LINK
                             //   — used to de-correlate (see below)
    Strength   float64       // h ∈ [0,1]
    Confidence float64       // c ∈ [0,1] — trust in this evidence (uses Quality)
    Horizon    Horizon       // IMMINENT(min) | SHORT(hours) | LONG(days)
    Polarity   Polarity      // AGGRAVATING | INHIBITING (negative evidence)
    Evidence   []Signal      // raw signals that fired it (audit trail)
}
```

### Example detectors

- **`xid_fatal`** (GPU, IMMINENT) — fatal Xid codes ≈ near-certain hard
  failure → `h≈0.95, c≈0.99`.
- **`ecc_acceleration`** (MEMORY, SHORT) — correctable ECC rate *accelerating*
  (2nd derivative > 0), not merely high → `h≈0.6`.
- **`smart_reallocate`** (DISK, LONG) — reallocated/pending sectors rising →
  `h≈0.5`.
- **`thermal_runaway`** (THERMAL, SHORT) — inlet temp rising while fan RPM
  already maxed → `h≈0.7`.
- **`selftest_pass`** (INHIBITING) — a recent clean diagnostic *lowers* hazard.

### Why naive noisy-OR is wrong, and what we do instead

Plain `P = 1 − Π(1 − hᵢ·cᵢ)` assumes hazards are independent. They are not, and
that creates three failure modes we explicitly fix:

1. **Common-cause double counting.** A thermal event raises temp *and* ECC
   errors *and* fan RPM. Naive noisy-OR multiplies these as if independent and
   inflates P. → **Group hazards by `Cause` class. Within a class, combine
   correlated hazards by `max` (or a correlated-OR), not product. Only combine
   *across independent classes* with noisy-OR.**

   ```
   P_class(k)  = noisy_max over hazards in cause-class k     // de-correlated
   P(fail)     = 1 − Π_k (1 − P_class(k))                    // across classes
   ```

2. **Weak-signal stacking / noise.** Many flapping low-strength signals could
   otherwise stack across the line. → **hysteresis + debounce** on each
   detector (a hazard must persist, not flicker), a **corroboration rule**
   (a LONG/weak hazard alone can't trigger AUTO; it needs a second class or
   sustained trend), and `Quality`-driven confidence so SUSPECT data is
   discounted.

3. **Horizon blending.** A "fails in days" disk trend and a "fails in minutes"
   GPU Xid must not be mashed into one scalar. → **P is computed per horizon
   bucket** (`P_imminent`, `P_short`, `P_long`). Decide consumes the bucket
   that matches the action's urgency.

4. **Negative evidence.** `INHIBITING` hazards subtract from the relevant
   class before combination, so a clean self-test can pull P back down.

```go
type Prediction struct {
    ID         string
    Node       string
    ByHorizon  map[Horizon]float64 // P per horizon bucket
    ByCause    map[CauseClass]float64
    Hazards    []Hazard            // the *why* — always attached
}
```

Hazard weights start hand-set and are later **calibrated** from §9's labels
(isotonic/Platt) so reported P matches observed failure frequency — keeping
explainability (per-class contributions) while gaining accuracy.

---

## 8. ③ Decide — confidence × blast-radius matrix (safety core)

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

### The asymmetry — "100%" means safety, not prediction accuracy

- **False positive** (drain a fine node) → a little wasted capacity,
  self-healing. Recoverable.
- **False negative** (miss one) → falls back to today's reactive ping-fail
  loop. **No worse than the status quo.**
- AUTO only ever runs **reversible** actions.
- We compare to **now (0% predictive accuracy)**, not to a perfect oracle.

> If we're wrong, you're no worse off than today. If we're right, you avoid an
> outage.

---

## 9. ⑤ Learn — closing the loop *without poisoning it*

The naive loop has a fatal flaw: **once we act, we may prevent (or hide) the
very outcome we're trying to learn from** — an interventional feedback loop.
"Drained, then fine" must NOT be filed as a false positive, or the model learns
to never act. We handle this explicitly.

**Key lever — reversible actions don't heal silicon.** A `drain` moves the
*workload*; it does not repair the chip. So after evacuating, we **keep
observing the now-empty node** and recover the *detection* ground truth
cleanly: if it really was failing, it still fails (on an empty node, harming
nothing); if it never fails over an extended window, that prediction is a
genuine FP candidate. Evacuation removes the *risk*, not the *evidence*.

```go
type Outcome struct {
    PredictionID  string
    Detection     DetectResult // FAILED | SURVIVED | CENSORED
    ActionTaken   Action
    OutageAvoided bool         // did evacuation save a real job? (value metric)
    Propensity    float64      // P(this node was acted on) — for de-biasing
}
```

Four anti-contamination mechanisms:

1. **Censored, not "healthy."** If a *destructive/irreversible* path was taken
   (rare, human-approved), the outcome is `CENSORED`, never `SURVIVED` —
   it cannot be used as a negative label.
2. **Post-drain observation.** Reversible-action nodes are watched on the empty
   node to obtain unbiased `FAILED`/`SURVIVED` (see above).
3. **Hold-out / control lane.** A small, *safe* fraction of qualifying
   predictions is deliberately left `OBSERVE`-only (e.g. nodes already idle, or
   randomized low-criticality) to get an uncontaminated base rate — a built-in
   control group, like a continuous mini-RCT. Never on protected/critical jobs.
4. **Propensity weighting.** Because acted-on nodes are non-random, training
   uses inverse-propensity weighting (and provenance tags) so the calibration
   in §7 isn't biased by our own interventions.

**Two truths kept separate:**
- **Detection truth** — did the hardware fail? (recoverable via post-drain
  observation, even after we act)
- **Action value** — did evacuation save a workload? (`OutageAvoided`)

Mixing these is what creates the feedback trap; keeping them apart is what
escapes it.

Outputs:
1. **Phase-0 trust report** — precision, FP rate, outages avoided, on the
   customer's *own* fleet, in shadow mode, before any action is enabled.
2. **Cross-fleet calibration** — anonymized labels feed the central aggregator
   (CF Workers + D1). More fleets → better-calibrated hazards → new customers
   get good predictions day 1. Big-tech see only their own fleet; vendors see
   signals but not outcomes — **only an independent product holds this
   cross-fleet label set.** The compounding moat.

---

## 10. Who runs it (four faces of one product)

| Actor | What they want | Surface |
|---|---|---|
| **SRE / platform** | pre-failure evacuation, fewer 3am pages | closed-loop core |
| **Vendors / OEMs** | "our hardware emits accurate leading signals" proof | CP / conformance program (later) |
| **ISP / IDC / hosting** | multi-tenant fleet health, SLA evidence | multi-tenant + reports |
| **Cloud / Neocloud** | less node churn, preserved capacity | orchestrator adapters |

Vendors are simultaneously customers, **data suppliers, and a distribution
channel** — the lever that fills the cross-fleet dataset fastest.

---

## 11. First wedge — GPU training (Slurm)

Start narrow. GPU training on Slurm is where one dead node stalls a whole
distributed job and the evacuation ROI is overwhelming, and where later GPU
operators lack big-tech-grade fleet automation. CPU fleets, inference/K8s, and
cloud adapters expand from there.

---

## 12. Why this shape (recap)

- **Flexible `Signal`** → new sources/failure modes are data, not code; quality
  and semantics are explicit, not assumed.
- **In-band-first, risk-adaptive, sharded collection** → BMC load scales with
  risk, not fleet size.
- **Correlation- and horizon-aware combination** → no common-cause inflation,
  no noise stacking, no horizon blending; with negative evidence.
- **Decide separated from Predict** → safety policy evolves independently of
  detection accuracy.
- **Reversible-only auto + guardrails** → never worse than the status quo.
- **Censoring + post-drain observation + control lane + IPW** → we learn ground
  truth *without* the intervention poisoning the labels.
- **Hybrid deployment** → deployable to sensitive fleets *and* builds the moat.
- **Shadow → recommend → closed-loop** → earn trust with numbers before taking
  control.
