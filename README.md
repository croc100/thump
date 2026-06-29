# Thump

**The fleet feels danger before it strikes.**

Thump is a closed-loop **predictive evacuation** platform for server and GPU
fleets. It reads the early-warning signals your hardware already emits — long
before a node goes dark — and safely moves workloads off nodes that are about
to fail.

When a rabbit senses a predator, it thumps the ground to warn the whole warren
before danger arrives. Thump does the same for your fleet.

---

## The problem

The usual on-prem / cloud failure loop is reactive:

```
ping fail  →  check device  →  detect fault  →  remediate
```

But a `ping fail` is the *last* signal, not the first. By the time a node stops
answering, its BMC and telemetry have usually been flashing warnings for hours
or days — correctable ECC bursts, GPU Xid errors, NVLink flaps, fan/temp drift,
PSU ripple, SMART reallocations.

For GPU training fleets the cost is brutal: a single dead node can stall an
entire distributed job, forcing a checkpoint rollback that burns thousands of
GPU-hours.

Thump turns this around: **detect the leading signal, then evacuate the
workload before the failure happens.**

---

## Why this is safe even when prediction isn't perfect

100% prediction is physically impossible. Thump is not built on perfect
prediction — it's built on **asymmetric, reversible action**:

- **False positive** (we drain a node that was actually fine) → a little wasted
  capacity. The workload keeps running elsewhere. Recoverable.
- **False negative** (we miss one) → you fall back to exactly today's reactive
  behavior. No worse than the status quo.

> If we're wrong, you're no worse off than today. If we're right, you avoid an
> outage.

This is enforced by four design rules:

1. **Confidence grading** — action aggressiveness is mapped to prediction
   confidence. Only high-confidence + reversible actions run automatically.
2. **Reversible-only automation** — auto actions are limited to `cordon` /
   `drain` / `live-migrate` / `LB drain`. Destructive actions (reboot, power,
   disk) stay human-approved.
3. **Compare to *now*, not to a god** — today's predictive accuracy is 0%
   (you only know after the ping dies). Catching even half the failures early
   is 0→50, not 100→50.
4. **Shadow mode first** — deploy with actions OFF, prove the numbers on the
   customer's own fleet, *then* let them flip on automation at a threshold they
   choose.

---

## Architecture — five cores

```
   ┌───────────┐   ┌──────────┐   ┌──────────┐   ┌────────────┐
   │ ① Collect │ → │② Predict │ → │③ Decide  │ → │④ Actuate   │
   │  signals  │   │  P(fail) │   │ grade    │   │ safe exec  │
   └───────────┘   └──────────┘   └──────────┘   └────────────┘
          ↑________________⑤ Learn (fault→outcome labels)________|
```

| Core | What it does |
|---|---|
| **① Collect** | DCGM, Redfish events + SEL, SMART/nvme, MCE/dmesg. Event-driven first, polling fallback. Scrapes existing Prometheus/DCGM-exporter where present — no "yet another agent." |
| **② Predict** | `node → P(fail within Δt)` with explainable evidence. Rules/statistics first (ECC acceleration, Xid patterns, SMART trends), models later. |
| **③ Decide** | Confidence × action-blast-radius matrix → auto / recommend / observe. Declarative policy (thresholds, blackout windows, "never touch this job"). Full audit trail. |
| **④ Actuate** | Safe adapters: **Slurm** and **K8s** drain first; VM live-migrate, LB drain later. Built-in dry-run. |
| **⑤ Learn** | Labels every prediction with its real outcome. Cross-fleet labels compound into the moat. |

---

## Deployment — hybrid

Thump runs **on-prem inside the customer fleet** (their telemetry never
leaves), while only **anonymized fault→outcome labels** are aggregated
centrally. This keeps security-sensitive operators (finance, sovereign AI,
on-prem GPU) deployable *and* builds the cross-fleet data moat.

---

## First wedge — GPU training (Slurm)

Thump starts narrow: **GPU training clusters on Slurm**, where one dead node
stalls a whole distributed job and the evacuation ROI is overwhelming. CPU
fleets, inference/K8s, and cloud adapters expand from there.

---

## Who it's for

| Operator | What they get |
|---|---|
| **SRE / platform** | Pre-failure evacuation, fewer 3am pages |
| **Neocloud / GPU cloud** | Less node churn, preserved capacity |
| **ISP / IDC / hosting** | Multi-tenant fleet health, SLA evidence |
| **Vendors / OEMs** | Hardware operability certification (CP program, later phase) |

---

## Status

Early development. See [ROADMAP.md](ROADMAP.md) for phases.

---

Part of the **CRODE** family · [crode.net](https://crode.net) · 🐇
