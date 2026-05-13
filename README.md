# sre-team

A production reliability review plugin for Claude Code. Parallel independent specialist reviewers assess blast radius, rollback safety, observability coverage, chaos readiness, on-call operability, and more — then the Reliability Commander cross-correlates findings to surface compound failure chains no single reviewer would catch.

No cross-talk during the review phase. Independence is the whole point.

## When to use

Say any of: **"sre review"**, **"reliability review"**, **"production readiness"**, **"is this safe to ship"**, **"blast radius review"**, **"ops review"**, **"review for reliability"**.

Also triggers proactively (with plan-card approval) when a new service is ready to ship, SLO-critical paths change, or infrastructure changes carry production blast radius.

## What you get

1. **Plan card** — roster, models, scope, budget. Reply `go` / `add <seat>` / `drop <seat>` / `promote <seat> to opus` / `defer` / `abort`.
2. **Handshake status** — confirms all reviewers started before the silent phase.
3. **Silent review** — parallel reviewers, no cross-talk. Each seat owns its domain exclusively; cross-domain signals are deferred, not double-filed.
4. **Depth triage** — Commander scans Critical/High findings for thin evidence and spawns targeted Opus deep-dives where the stakes justify it.
5. **Cross-correlation** — compound failure chains that no single reviewer would catch: untested rollback + missing alert, proxy SLI + unverified capacity, depleted error budget + high-risk change, and more.
6. **Report preview** — full draft in chat before any file is written. Reply `save` / `amend <note>` / `accept <finding-id> "<rationale>"` / `discard`.
7. **Output** — `~/.claude/sre-team/<date>-<slug>/report.md` (outside the repo under review).
8. **Tracker filing** — when [beads](https://github.com/gastownhall/beads) is detected, Critical/High/Medium findings auto-file after save. Fails-safe — projects without beads behave identically.

## Roster

### Always-on

| Seat | Owns |
|------|------|
| `sre-reviewer` | Blast radius, SLO existence + error budget policy, rollback safety, failure modes, MTTD, retry behaviour, health checks, AZ distribution, startup resilience, dependency SLA compatibility |
| `chaos-reviewer` | Steady state definition, blast radius control for experiments, abort path readiness, observability prerequisites, learning loop |
| `observability-reviewer` | Golden signals, SLI/SLO validity, alert coverage, burn rate alerting, detection latency, structured logging, tracing, dashboard coverage, metric cardinality, alert routing |
| `release-reviewer` | Deployment blast radius, progressive delivery, rollback mechanics, canary analysis, post-deploy verification, config safety, connection draining |
| `on-call-reviewer` | Runbook completeness, escalation paths, alert actionability, service ownership, incident communication, 3am operability, incident learning loop |

### Opt-in (auto-promoted when signals are detected)

| Seat | Add when | Auto-promotes on |
|------|----------|-----------------|
| `capacity-reviewer` | Load profile changes, fan-out, autoscaling | HPA config, traffic SLOs, single replica, new fan-out pattern |
| `data-reviewer` | Data storage, schema, pipelines | Migrations, queues/streams, new data stores |
| `network-reviewer` | Network config changes | NetworkPolicy, Ingress, TLS, service mesh config, DNS changes |

## Two-tier model

All seats run **Sonnet** for the broad scan pass — coverage is the goal. The Commander then triages Critical/High findings for depth signals and spawns targeted **Opus** deep-dives (max 2 per run) where the stakes justify it: irreversibility first, blast radius second, novelty third. The Commander itself runs on **Opus** for cross-correlation.

Use `promote <seat> to opus` in the plan card to upgrade a specific seat for the whole run.

## Temporal recon

Before spawning reviewers, the Commander builds a brief that includes:

- Prior sre-team run history for this service (unresolved Critical/High findings escalate in severity if they recur)
- Postmortem content extraction — what failed, contributing factors, action item status
- Error budget posture — current burn rate changes how every seat calibrates severity
- Dependency reliability signals — which dependencies have SLOs and which are unknown reliability surfaces

## Accepted risks

At report preview, reply `accept <finding-id> "<rationale>"` to acknowledge a finding without fixing it. Accepted risks move to a dedicated section of the report, are excluded from bead filing, and are annotated `risk_accepted: true` so they do not escalate in future runs.

## Tool-agnostic

No specific observability stack, chaos framework, deployment tooling, or incident platform is assumed. Reviewers assess what should exist and whether it does — not which tool implements it.

## SRE Workbook alignment

This plugin's seat roster aligns with the core operational concerns in the [Google SRE Workbook](https://sre.google/workbook/table-of-contents/): SLO implementation and error budget policy, monitoring and alerting, eliminating toil, on-call practices, chaos engineering, canarying releases, managing load and overload, and data pipeline resilience. Seats are scoped to the technical production readiness concerns from the Workbook; org-process chapters (engagement model, change management) are intentionally out of scope.

## Installation

```bash
npx claude install https://github.com/Tazmainiandevil/sre-team.git
```

## Related

- [sjsyrek/red-team](https://github.com/sjsyrek/red-team) — adversarial security review (same parallel independent pattern, security domain)
- [sjsyrek/design-council](https://github.com/sjsyrek/design-council) — cross-domain design debate, includes `sre-engineer` opt-in seat for design-time reliability input
