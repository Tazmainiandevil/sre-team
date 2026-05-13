# Changelog

All notable changes to the sre-team plugin are documented here. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] — 2026-05-13

Initial release.

### Added

**Core protocol**
- Reliability Commander orchestrates all phases: pre-scan, plan card, full recon, parallel fan-out, depth triage, cross-correlation, report preview, tracker filing
- Phase 0 pre-scan populates roster auto-promotions before plan card is shown — plan card is always accurate
- Plan card with roster, per-seat model, scope, and budget. User commands: `go`, `add <seat>`, `drop <seat>`, `promote <seat> to opus`, `narrow scope`, `review as diff`, `defer`, `abort`
- `review as diff` mode: scopes recon and all reviewer spawn prompts to files changed in the current branch vs main; auto-suggested for small PRs (≤ 30 changed files); trigger phrases "review this PR" / "review this branch" / "is this PR safe" pre-select it
- Handshake verify after spawn — spawn health visible, not inferred
- Phase 2.75 depth triage: Commander scans Critical/High findings for thin evidence and spawns targeted Opus deep-dives (max 2 per run, triage order: irreversibility → blast radius → novelty)
- Phase 3 cross-correlation with compound failure chain detection and deduplication
- Report preview before any file write. `accept <finding-id> "<rationale>"` marks findings as risk-accepted, excludes from bead filing, annotates `risk_accepted: true` for future runs
- Stop-early path: partial report with halted status, no beads filed
- All run artifacts co-located in `~/.claude/sre-team/<yyyy-mm-dd>-<slug>/` (recon + report in same directory)

**Seat boundary protocol**
- Each seat owns its domain exclusively; cross-domain signals use `chain_potential: true` + `"defer to <seat>"`
- Commander owns a seat ownership table for routing defers
- Deduplication step before cross-correlation collapses near-duplicate findings, respects defers

**Always-on roster (5 seats, Sonnet broad scan)**
- `sre-reviewer` — blast radius, SLO existence + error budget policy, rollback safety, failure modes, MTTD, retry/backoff behaviour, health check quality, AZ replica distribution, startup resilience, dependency SLA compatibility, cache invalidation failure modes, multi-region blast radius (cross-region shared components), toil
- `chaos-reviewer` — steady state definition, blast radius control for experiments, abort path readiness, observability prerequisites, learning loop, experiment history
- `observability-reviewer` — golden signals, SLI/SLO validity, alert coverage, burn rate alerting, detection latency, structured logging, tracing, dashboard coverage, metric cardinality, alert routing
- `release-reviewer` — deployment blast radius, progressive delivery, rollback mechanics, kill-switch existence + testing, canary analysis automation, post-deploy verification, config safety, connection draining on pod termination
- `on-call-reviewer` — runbook completeness, escalation paths, alert actionability, service ownership, incident communication templates, 3am operability, incident learning loop

**Opt-in roster (3 seats, Sonnet, auto-promotion supported)**
- `capacity-reviewer` — load profile, headroom, autoscaling bounds, cold start latency, CPU throttling detection; auto-promotes on HPA config, traffic SLOs, single replica, new fan-out
- `data-reviewer` — data durability, backup/restore, RPO/RTO, schema migration safety, database failover duration, read replica lag; auto-promotes on migrations, queue/stream references, new data stores
- `network-reviewer` — ingress/LB timeout alignment, DNS TTL, NetworkPolicy correctness, TLS lifecycle, service mesh policy conflicts, mTLS enforcement, connection draining, egress firewall; auto-promotes on NetworkPolicy, Ingress, TLS, service mesh config

**Finding schema**
- Fields: `id`, `severity`, `surface`, `finding`, `conditions`, `evidence`, `suggested_remediation`, `category` (includes `network`), `chain_potential`
- `suggested_remediation` field: specific action, not generic advice
- `id` scoped to run only — cross-run matching uses date + finding one-liner

**Compound failure chain patterns**
- Untested rollback + missing alert → silent failure with no recovery path
- No SLO + cascading blast radius → unmeasured outage duration
- Proxy SLI + unverified capacity → SLO stays green while users suffer
- Toil + paging conditions → on-call fatigue spiral
- Untested chaos abort + production experiment → uncontrolled outage
- Missing runbook + long MTTD → operator blind during blast radius window
- Irreversible schema migration + no rollback → full outage with no recovery
- Single replica + no HPA + capacity signal → single point of failure, no automated recovery
- Error budget >80% consumed + high-risk change → shipping into negative safety margin
- Aggressive retry + no circuit breaker → failure amplification, thundering herd on recovery
- NetworkPolicy change + no network alert + dependency failure symptoms → misattributed outage, wrong layer debugged

**Temporal recon enrichment**
- Prior sre-team run history: unresolved Critical/High findings from most recent run escalate in severity if they recur
- Postmortem content extraction: what failed, contributing factors, action item status (not just existence)
- Error budget posture: burn rate changes severity calibration for all seats
- Dependency reliability signals: which dependencies have SLOs, which are unknown surfaces

**Beads tracker integration (fails-safe)**
- Detection: `command -v bd && bd where` — both must pass, prevents false positive when bd is globally installed
- Phase 1 enrichment: `bd ready -t bug` populates `## Known issues from tracker` in recon brief
- Skill self-audit: `bd prime --memories-only` surfaces lessons that contradict SKILL.md prescriptions
- Filing: component findings first (Critical p0, High p1, Medium p2), compound chains second with `bd dep add`
- Silent-promise guard: every unfiled Critical/High that isn't risk-accepted is caught before finalisation
- Chain beads are tracking artifacts only — actionable work is on the component beads
- Emergent insights write-back: `bd remember` persists structural lessons; Appendix entry always written regardless of tracker status

**Severity guide**: operational impact calibration, compound chain severity rules
**Recon template**: 12 sections including all temporal enrichment fields
**Report template**: frontmatter, findings by severity, compound chains, accepted risks, clean checks, scope gaps, recommendations, emergent insights appendix
