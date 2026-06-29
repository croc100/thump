# Thump Roadmap

Thump is a closed-loop **predictive evacuation** platform: detect a hardware
failure before it happens, then safely move the workload off the node. This
roadmap is the source of truth for sequencing.

Guiding principle: **earn trust before taking control.** Every phase ships
something usable, and automation is only unlocked once the numbers are proven
on the customer's own fleet.

---

## Phase 0 — Shadow (predict only, actions OFF)

Build trust with hard numbers before touching a single workload.

- **① Collect**: DCGM + Redfish events + SMART (three sources only).
- **② Predict**: rule-based, explainable (`P(fail within Δt)` with evidence).
- **⑤ Learn**: label every prediction with its real outcome.
- Output: *"Over the last 60 days we flagged N nodes; X% actually failed; FP
  rate was Y per node per month."*
- No actuation. Trust first.

**Exit criteria:** a credible precision/false-positive report on a real fleet.

---

## Phase 1 — Recommend (human-in-loop)

- **③ Decide**: confidence grading + declarative policy (thresholds, blackout
  windows, protected jobs).
- **④ Actuate**: Slurm + K8s `drain` adapters, **one-click approval** only.
- Operators feel the action with a human gate before anything moves.

**Exit criteria:** operators routinely approving recommendations; measured
outages avoided.

---

## Phase 2 — Closed-loop (high-confidence auto)

- Auto-execute only `high-confidence × reversible` actions
  (`cordon` / `drain`). Everything else stays a recommendation.
- Full audit trail + dry-run on every path.
- Automation share expands gradually, driven by accumulated label data.

**Exit criteria:** automated evacuation running in production on at least one
fleet with zero destructive-action incidents.

---

## Phase 3 — Multi-fleet data + CP program

- Cross-fleet **fault→outcome** labels compound → better day-1 accuracy for new
  customers (the moat).
- **CP (Conformance) program** formalized:
  - **Hardware certification** — OEMs certify that their server/GPU/BMC emits
    accurate leading signals (`Thump Verified` badge + coverage report).
  - **CP test harness** — IDC/cloud burn-in + fault injection (forced ECC,
    thermal stress) to verify both the hardware *and* Thump's detection.
- Vendors become customers, data suppliers, and a distribution channel.

---

## Beyond

- Additional actuators: VM live-migrate, LB drain.
- Cloud adapters (instance health APIs where BMC is invisible).
- CPU / general-server fleets.
- Multi-environment single pane: on-prem bare metal + cloud + GPU.

---

Part of the **CRODE** family · [crode.net](https://crode.net) · 🐇
