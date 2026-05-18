# Severity Guide

Calibrate severity based on **operational impact**, not theoretical risk. The question is not "could this fail?" but "what happens when it fails, and how bad is it?"

## Critical

Any finding where:
- Failure causes data loss, data corruption, or irrecoverable state
- No detection path exists (failure is silent and blast radius is high)
- No rollback path exists or rollback has never been demonstrated on a production-critical path
- SLO violation would affect the majority of users with no graceful degradation
- An untested schema migration on a live table with no rollback script

**Default treatment:** Do not ship until resolved or explicitly accepted with a documented exception.

## High

Any finding where:
- Failure is detectable but MTTD is longer than acceptable given the blast radius
- Rollback path exists but has never been executed (demonstrated time unknown)
- SLI measures a proxy metric — could stay green while user experience degrades
- Cascading blast radius is mapped but downstream services have no visibility for operators
- An on-call procedure requires the original author or undocumented tribal knowledge

**Default treatment:** Resolve before shipping to production, or ship behind a feature flag with an explicit timeline to resolve.

## Medium

Any finding where:
- Known toil exists with no automation path or timeline
- Runbook exists but is incomplete or has not been followed end-to-end
- Capacity assumptions are unverified but within plausible range
- Alert exists but fires after SLO breach rather than before (no early-warning headroom)
- Chaos experiment framework is absent but the service has low blast radius

**Default treatment:** File a tracked item with a deadline. Ship if risk is explicitly accepted.

## Low

Any finding where:
- Minor observability gaps that would slow diagnosis but not prevent it
- Runbook style issues (missing links, outdated contacts) with correct procedure intact
- Resource requests/limits set by convention rather than measurement on a low-criticality service
- Chaos experiment documentation missing but service has been informally tested

**Default treatment:** File a tracked item. No shipping gate.

## Info

A domain that was checked and found clean. **Always emit info findings for clean domains** — omission is ambiguous. A reviewer that checked and found nothing is different from a reviewer that never ran.

---

## Evidence calibration — absence of inline rationale is not absence of measurement

When a finding's severity rests on what's MISSING from the reviewed scope (no inline comment explaining a threshold, no SLO doc, no runbook found, no postmortem present), distinguish:

- **Documentation gap** (Medium/Low): the rationale, comment, or doc doesn't exist in-repo, but the underlying work may exist out-of-band (in a dashboard, a ticket, a PR comment, the author's measurement notebook).
- **Substantive gap** (High/Critical): positive evidence the underlying work is missing or wrong.

If you have only absence-of-evidence, the finding is the documentation gap. Do not escalate to substantive unless you have positive evidence — e.g., you ran the relevant query and the baseline is at or above threshold, you searched all likely SLO locations and confirmed none records this service, you traced the alert routing and it has no destination team.

Remediation for documentation gaps is cheap (record the rationale inline). Remediation for substantive gaps is expensive (re-derive the value, define the SLO, build the runbook). Calibrate severity to the cost of the actual remediation.

---

## Compound chain severity

Compound chains are rated **one level above the highest component finding**:
- Two High findings that combine → Critical chain
- High + Medium that combine into a worse outcome → High chain (same as the High component; only raise if the combination creates a qualitatively new risk category)

Use judgment: the question is whether the combination opens an attack or failure path that neither component opens alone.
