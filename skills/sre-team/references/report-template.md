---
date: YYYY-MM-DD
slug: <slug>
status: complete | halted
cancelled_at: null | phase-N
cancellation_reason: null | user_request
reviewer_count: N
scope: <scope>
tracker: beads | none
primary-tracker-ids: []
linked-tracker-ids: []
---

# SRE Review: <slug>

**Date:** YYYY-MM-DD
**Scope:** <scope>
**Reviewers:** sre-reviewer, chaos-reviewer, observability-reviewer, release-reviewer, on-call-reviewer [, capacity-reviewer] [, data-reviewer]

---

## Executive Summary

<!-- 3–5 sentences. What is the overall reliability posture? What are the top risks? Is it safe to ship? -->

---

## Critical Findings

<!-- Each finding block: -->

### [CRIT-001] <one-sentence finding>

- **Surface:** <component or domain>
- **Category:** <blast-radius | rollback | slo | observability | toil | chaos | runbook | capacity | dependency>
- **Conditions:** <what triggers or exposes this risk>
- **Evidence:** <file:line, config section, or spec reference>
- **Tracker ID:** <!-- filled after 4b filing, or "N/A (no tracker)" -->

---

## High Findings

<!-- Same format as Critical -->

---

## Compound Failure Chains

<!-- Cross-correlation results. Each chain: -->

### [CHAIN-001] <chain name>

- **Severity:** Critical | High
- **Components:** CRIT-001, HIGH-003
- **Failure narrative:** <3–5 sentences describing how this chain plays out end-to-end>
- **Tracker ID:** <!-- filled after 4b filing -->

---

## Medium Findings

<!-- Same format as Critical -->

---

## Low Findings

<!-- Same format as Critical -->

---

## Clean Checks

<!-- Domains where reviewers found nothing. Never omit — omission is ambiguous. -->

| Reviewer | Domain checked | Result |
|----------|---------------|--------|
| sre-reviewer | Blast radius, SLO, rollback, failure modes, MTTD, toil | No findings |
| ... | ... | ... |

---

## Accepted Risks

<!-- Findings acknowledged by the user during report preview with a documented rationale. -->
<!-- These are NOT filed as beads and do NOT escalate in future runs (risk_accepted: true). -->
<!-- Format per entry: -->

<!-- ### [ORIG-ID] <finding one-liner>
- **Original severity:** High
- **Accepted by:** <user> on YYYY-MM-DD
- **Rationale:** <exact rationale provided>
- **Review date:** YYYY-MM-DD (when this acceptance should be re-evaluated) -->

<!-- "None" if no findings were accepted. -->

---

## Scope Gaps

<!-- Reviewers that failed to start after two spawn attempts. -->
<!-- "None" if all reviewers started successfully. -->

---

## Recommendations

<!-- Prioritised action list. Critical and High findings each need a concrete, specific recommendation. -->

1. **[CRIT-001]** <specific remediation, not "fix the issue">
2. ...

---

## Appendix — Emergent insights

<!-- Structural lessons from this run worth persisting to future runs. -->
<!-- Written to beads memory when tracker is detected. -->
