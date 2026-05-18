---
name: sre-team
description: Production reliability review. Spawn independent specialist reviewers to find every operational risk in a design or change — blast radius, untested rollback, missing alerts, SLO violations, toil, runbook gaps, and chaos coverage. The Reliability Commander cross-correlates findings to surface compound failure chains no single reviewer would catch. Use when the user says "sre review", "reliability review", "production readiness", "is this safe to ship", "blast radius review", "ops review", or "review for reliability". Also triggers proactively (with plan-card pause for approval) when a new service is ready to ship, SLO-critical paths change, or infrastructure changes carry production blast radius.
version: 0.1.0
---

# sre-team

A multi-agent production reliability review skill. Independent specialist agents review from distinct operational angles with **no shared context during the review phase**; the Reliability Commander cross-correlates findings afterward to surface compound failure chains no single reviewer would catch.

## What the user sees

1. **Plan card.** Before any agent spawns, the Commander shows a one-screen card: scope, roster (always-on plus opt-in seats with rationale for each opt-in), per-seat model, rough token/wall-clock budget. Reply `go`, `add capacity-reviewer`, `drop chaos-reviewer`, `narrow scope to src/payments`, or `abort`.
2. **Handshake status.** After spawn: `HANDSHAKE: 5/5 ok | started=[...] | failed=[]`. Spawn health visible, not inferred.
3. **Silent review phase.** Agents run in parallel with no inter-agent communication. The Commander surfaces per-agent progress notifications.
4. **Report preview.** Before any file is written, the Commander posts the draft report to chat. Reply `save`, `amend <note>`, `accept <finding-id> "<rationale>"` (mark as risk-accepted, excluded from bead filing), or `discard`.
5. **Output.** Final report at `~/.claude/sre-team/<yyyy-mm-dd>-<slug>/report.md` (outside the repo under review).
6. **Stop early.** Say "stop the sre review" at any phase — Commander cancels in-flight agents, collects whatever findings are already in, writes a partial report with `status: halted`.

## When to invoke

Trigger phrases: "sre review", "reliability review", "production readiness", "is this safe to ship", "ops review", "blast radius review", "review for reliability", "pre-release review", "operational readiness", "review this PR", "review this branch", "is this PR safe".

**Proactive triggers** (pause for plan-card approval, do not run unprompted):
- New service being shipped for the first time
- Changes to SLO-critical code paths (auth, payment, core data pipelines)
- Infrastructure changes with production blast radius (autoscaler config, database migration, network policy)
- New external dependencies introduced in production code

**Do NOT invoke** for: doc-only changes, local dev tooling, pure refactors with no behavioral change, bug fixes that don't alter failure modes. The token cost isn't earned.

## Phase 0 — Recon + Plan card

Phase 0 has two parts: a quick recon pass to build the full roster, followed by the plan card for user approval. The plan card cannot be accurate without knowing what auto-promotions apply — so recon comes first.

### Phase 0a — Pre-scan (Commander)

Before presenting the plan card, run these specific lightweight checks — file existence and grep only, no deep reads. Save all findings for Phase 1 full recon.

1. Detect tracker: `command -v bd > /dev/null 2>&1 && bd where > /dev/null 2>&1`
2. Check capacity-reviewer auto-promotion signals: `grep -rl 'hpa\|horizontalpodautoscaler\|replicas:' . 2>/dev/null | grep -v '.git/' | head -5`; check for traffic SLO docs; check if replica count is 1
3. Check data-reviewer auto-promotion signals: `find . \( -name '*migration*' -o -name '*schema*' \) 2>/dev/null | grep -v '.git/' | head -5`; `grep -rl 'kafka\|kinesis\|rabbitmq\|pubsub\|nats\|sqs' . 2>/dev/null | grep -v '.git/' | head -5`
4. Check network-reviewer auto-promotion signals: `find . \( -name '*networkpolicy*' -o -name '*ingress*' -o -name '*virtualservice*' -o -name '*destinationrule*' \) 2>/dev/null | grep -v '.git/' | head -5`; check for cert-manager or TLS config files
5. Count expected reviewer seats and estimate token budget (~15k tokens per always-on seat, ~10k per opt-in)
6. **Detect diff context.** Run `git branch --show-current 2>/dev/null`. If the current branch is not `main` or `master`, run `git diff main...HEAD --name-only 2>/dev/null | wc -l`. If the diff is ≤ 30 changed files, note "diff mode available" for the plan card — scoping the review to the branch diff is more cost-efficient than reviewing the whole repo. If the user's trigger phrase was "review this PR" or "review this branch", pre-select diff mode and reflect it in the plan card scope.

### Phase 0b — Plan card

Post to chat. Wait for the user's reply.

```
SRE REVIEW PLAN
===============
Scope:         repo root (default) | <path> if narrowed | diff of <branch> vs main if diff mode
Always-on:     sre-reviewer, chaos-reviewer, observability-reviewer,
               release-reviewer, on-call-reviewer
Opt-in:        [list with one-line rationale per seat]
Auto-promoted: [seat — trigger reason]  (omit row if none)
Models:        broad scan (all seats) → Sonnet | deep-dives → Opus | Commander → Opus
Budget:        ~Nk tokens, ~M minutes wall-clock
Output:        ~/.claude/sre-team/<date>-<slug>/report.md

Reply: go | add <seat> | drop <seat> | promote <seat> to opus | narrow scope to <path> | review as diff | defer | abort
```

User commands:
- `go` — proceed to Phase 1 full recon at the current scope
- `add <seat>` / `drop <seat>` — adjust roster
- `promote <seat> to opus` — upgrade a specific seat from Sonnet to Opus for this run (use when the service has unusual complexity in that domain)
- `narrow scope to <path>` — re-run pre-scan with tighter scope, re-present plan card
- `review as diff` — scope Phase 1 recon and all reviewer spawn prompts to files changed in the current branch vs main; more focused and cost-efficient for PR reviews
- `defer` — acknowledge the trigger but skip this review now; no spawn, no artifact, no re-prompt until the user invokes manually
- `abort` — stop entirely, no spawn, no artifact

## Phase 1 — Recon (Commander, single context)

Build the recon brief. This is the only shared context across all reviewers — it must be precise and bounded.

Steps:

0. **Tracker detection (fails-safe).** Run `command -v bd > /dev/null 2>&1 && bd where > /dev/null 2>&1` from the project's git root. Both must pass: `bd` in PATH and a database reachable from the current directory. This prevents false-positive detection when `bd` is installed globally but the project has no beads database. If the check succeeds, beads is this run's tracker — gather context per `references/tracker-integration.md` (Phase 1 recon enrichment + skill self-audit) and inline into a `## Known issues from tracker` section of `recon.md`. If neither succeeds, skip silently. **When self-audit detects a memory entry that contradicts a prescription in this file, memory wins**; flag the contradiction in `recon.md`, follow memory, and record the drift in the report's `Appendix — Emergent insights`.
1. Read `CLAUDE.md` if present — extract architecture constraints, known risks, out-of-scope areas.
2. Scan project structure:
   - Services involved, their dependencies, and data stores
   - Deployment configuration and rollback approach (if visible in config files)
   - Existing SLO definitions (if documented)
   - Recent commits: `git log --oneline -15` — populate the `## Recent changes` section of the recon brief so reviewers can prioritise new code paths
   - Incident history or postmortem docs (if present in the repo): `git log --oneline --all | grep -i 'incident\|hotfix\|rollback\|revert'`

   **Diff mode** (active when the user replied `review as diff`, or when the trigger phrase was "review this PR" / "review this branch"): run `git diff main...HEAD --name-only` to get the exact changed file list, and `git diff main...HEAD --stat` for a summary. Scope the architecture summary and all reviewer spawn prompts to the changed files and their direct dependencies. Include the diff stat in the recon brief under `## Recent changes` so reviewers know precisely what changed in this branch. Reviewers should focus findings on the changed surfaces and explicitly note if a finding is in unchanged code that the diff exposes.
3. **Search for deployment values outside the service repo.** The service repo alone is not the full blast radius surface. Search broadly — deployment config may live in the same repo or in a sibling GitOps/infra repo. Try:
   ```bash
   # Common deployment config locations — adapt to whatever is present
   find . -name "*.yaml" -o -name "*.yml" 2>/dev/null | \
     xargs grep -l "<service-name>" 2>/dev/null | \
     grep -v '.git/' | head -30
   
   # Also check sibling repos visible from workspace root or mentioned in CLAUDE.md
   find .. -maxdepth 4 \( -name "values*.yaml" -o -name "*.values.yaml" \
     -o -name "kustomization.yaml" -o -path "*/overlays/*" \
     -o -path "*/.github/workflows/*.yml" \) 2>/dev/null | \
     xargs grep -l "<service-name>" 2>/dev/null | head -20
   ```
   Patterns to look for: Helm `values.yaml` files, Kustomize overlays, ArgoCD Application manifests, Flux HelmRelease resources, GitHub Actions / GitLab CI deploy jobs, Terraform/OpenTofu variable files. The specific paths depend on the project — check `CLAUDE.md` or workspace layout for hints. Specifically look for:
   - Replica count and HPA config (single-replica = no blast radius isolation)
   - Alerting and monitoring enable/disable flags
   - Resource limits and requests
   - Feature flags or progressive delivery config (canary weight, traffic splits)
   
   Record the deployment values path (or "not found") in the recon brief. **If you cannot locate deployment values, flag this explicitly as a recon gap** — reviewers must know whether production config was checked. The most operationally significant findings often live in deployment config, not service code.
4. **Prior sre-team run history.** Check `~/.claude/sre-team/` for previous dated run directories matching this service: `ls ~/.claude/sre-team/ 2>/dev/null | grep <slug>`. Look for `report.md` files inside those directories — **not** `recon.md`. Sort by date descending and read the most recent completed `report.md`. Extract all Critical and High findings (excluding any marked `risk_accepted: true`) and write them to a `## Unresolved findings` section of the recon brief with the run date. A finding that recurs across runs is either accepted risk (note it) or a persistent gap (escalate its severity this run). Reviewers who see an unresolved finding should assess whether conditions have changed, not re-discover it from scratch. If no prior run directories exist, note "no prior sre-team history" — this signals the service's first structured reliability review.
5. **Postmortem and failure history.** Go deeper than git log keywords. Search for postmortem documents in the repo (`find . \( -iname "*postmortem*" -o -iname "*post-mortem*" -o -iname "*incident*" \) 2>/dev/null | grep -v '.git/'`). For each postmortem found, extract: what failed, contributing factors, and whether action items are marked resolved. Write a `## Failure history` section summarising the recurring failure modes — even 3-5 bullet points per incident. Reviewers can then treat known failure modes as confirmed risk, not hypothesis. If postmortems exist but action items are unresolved, flag this explicitly — the failure is likely to recur.
6. **Error budget posture.** Search for SLO or error budget documentation (`find . \( -iname "*slo*" -o -iname "*error-budget*" -o -iname "*error_budget*" \) 2>/dev/null | grep -v '.git/'`). If found, extract current error budget consumption or burn rate if documented. Write a `## Error budget posture` section. A service burning >50% of its error budget in the current window is in a different risk posture than one at <10% — this context changes how every seat calibrates severity. If no SLO documentation is found, note the absence: it will surface as a finding for sre-reviewer.
7. **Dependency reliability signals.** For each external dependency identified in step 2, check whether it has a documented SLO or reliability guarantee (search the repo, CLAUDE.md, or any architecture docs). Write a `## Dependency reliability` section noting: dependency name, whether an SLO was found, and any known failure history. "No SLO found for dependency X" is a signal worth surfacing — sre-reviewer should treat that dependency as an unknown reliability surface rather than a safe assumption.
8. Identify which opt-in seats apply — and check auto-promotion conditions:

   **capacity-reviewer** — Add manually when the change affects fan-out, replica count, or query load. **Auto-promote to always-on** when any of the following are found during recon:
   - Traffic SLOs are defined for this service (a traffic SLO implies headroom matters)
   - HPA config is present (autoscaler bounds must be correct)
   - The change introduces a new fan-out pattern (one request → N downstream calls)
   - Replica count is 1 (single-replica services have no blast radius isolation — capacity is the whole story)

   **data-reviewer** — Add manually when the change touches data storage or schema. **Auto-promote to always-on** when any of the following are found during recon:
   - Queue, stream, or messaging system references (Kafka, SQS, Kinesis, RabbitMQ, Pub/Sub, NATS)
   - Batch job or scheduled data processing patterns
   - Schema migration files in the change set
   - New data store introduced (any database, cache, blob storage)

   **network-reviewer** — Add manually when the change touches network configuration. **Auto-promote to always-on** when any of the following are found during recon:
   - Ingress, Gateway, or load balancer configuration files in the change set
   - Kubernetes NetworkPolicy resources added or modified
   - DNS record changes or TTL modifications
   - TLS certificate configuration, renewal settings, or cert-manager resources
   - Service mesh configuration files (VirtualService, DestinationRule, PeerAuthentication, etc.)

   When auto-promoting a seat, note the trigger in the plan card under `Opt-in (auto-promoted)` with the reason. The user can still drop it.

Write the brief to `~/.claude/sre-team/<yyyy-mm-dd>-<slug>/recon.md` — the same dated directory that will hold the report. This keeps recon and report co-located and prevents prior-run lookup from confusing recon files with reports. Every reviewer's spawn prompt points to this path for prompt-cache hits across parallel spawns.

Brief format: see `references/recon-template.md`. Tracker enrichment recipe: `references/tracker-integration.md`.

## Phase 2 — Review fan-out (parallel)

Spawn all selected agents simultaneously in **one multi-tool-call message** via `Agent(... run_in_background: true)`. Each agent receives:

- Its **role brief** loaded from `references/agents/<slug>.md`
- A `Read` instruction for the recon brief at `~/.claude/sre-team/<yyyy-mm-dd>-<slug>/recon.md`
- The **finding schema** (below)
- Full tool access (file reads, bash)
- A **tracker-awareness rule** (when the recon brief contains a `## Known issues from tracker` section): before reporting any finding, check the listed beads. If your finding matches an existing bead, skip it or report with `chain_potential: true` and the bead id in `evidence` — never refile as a fresh finding.

**Seat boundary rule.** Each seat owns its domain exclusively. When a reviewer encounters a signal outside its domain, it includes the signal in the finding's `evidence` field with `"defer to <seat>"` and emits `chain_potential: true` — it does not file the finding itself. The Commander links these defers in Phase 3. This prevents duplicate findings and preserves the independence that makes cross-correlation meaningful.

**Model.** All seats run on Sonnet for the broad scan pass. Coverage is the goal — Sonnet handles breadth reliably. Depth comes from Phase 2.75.

**Critical**: Agents do not communicate during this phase. Independence is the whole point.

### Finding schema

Each agent emits one JSON block per finding:

```json
{
  "id": "<surface-slug>-<6char-hex>",
  "severity": "critical | high | medium | low | info",
  "surface": "the component or operational domain reviewed",
  "finding": "one-sentence description of the reliability risk",
  "conditions": "what operational conditions trigger or expose this risk",
  "evidence": "file path, config section, architecture element, or spec reference",
  "category": "blast-radius | rollback | slo | observability | toil | chaos | runbook | capacity | dependency | network",
  "chain_potential": false,
  "suggested_remediation": "specific action to resolve or mitigate — optional but strongly encouraged for Critical and High findings"
}
```

**Field notes:**
- `chain_potential`: set to `true` when your finding has a signal outside your domain (defer) or when it is likely to combine with another finding for a worse compound outcome. Default `false`.
- `suggested_remediation`: the specific action, not generic advice. "Add `PodAntiAffinity` with `topologyKey: topology.kubernetes.io/zone`" is correct. "Improve availability" is not.
- `id` values are scoped to this run only — they are not stable across runs. Cross-run matching uses run date + finding one-liner, not IDs.

If a reviewer finds nothing in its domain, it must emit one `info`-severity entry naming what was checked:

```json
{
  "id": "sre-clean-001",
  "severity": "info",
  "surface": "production reliability",
  "finding": "No findings. Reviewed blast radius, SLO coverage, rollback path, failure modes, and toil.",
  "conditions": "N/A",
  "evidence": "N/A",
  "category": "blast-radius",
  "chain_potential": false
}
```

A clean check is a real result. Omission is ambiguous.

## Phase 2.5 — Handshake verify

After spawn, count incoming acknowledgements. Emit:

```
HANDSHAKE: N/N ok | started=[...] | failed=[...]
```

For any failed reviewer:
- Retry once with the same role brief
- If second spawn fails, proceed and list the failed surface in the report's "Scope Gaps" section

## Phase 2.75 — Depth triage (Commander)

After all Sonnet findings are in, the Commander scans Tier 1 (Critical/High) findings for depth signals before writing the report.

**Trigger conditions for a deep-dive spawn:**
- Critical or High finding with vague or thin evidence ("may not exist", "unclear if", "assumed")
- Two or more `chain_potential: true` findings pointing at the same component with no corroborating evidence
- A safety-critical surface (payment, auth, core data pipeline) with shallow coverage (single finding, no evidence files cited)
- **Three or more reviewers independently flag the same surface via absence-of-evidence reasoning** (no inline comment, no doc found, no rationale recorded). Cross-reviewer corroboration is normally a confidence signal, but identical absence-based inferences are a shared blind spot, not independent evidence. The deep-dive's job is to convert absence into either positive evidence (escalate the finding) or documentation-gap framing (downgrade it per `references/severity-guide.md`).

**Spawn up to 2 targeted Opus deep-dive agents.** Each receives: the specific finding(s) to validate, the recon brief path, and a directive to either confirm with specific evidence or downgrade with rationale. Hard limit: **max 2 deep-dive agents per run** regardless of how many triggers fire.

When more than 2 surfaces qualify, triage in this order:
1. **Irreversibility first** — findings where being wrong cannot be undone: schema drops, hard deletes, credential revocations, data pipeline truncations. A false-negative here has no recovery path.
2. **Blast radius second** — the surface with the widest confirmed user impact if the finding is real.
3. **Novelty third** — a failure mode not present in prior sre-team runs or postmortems for this service. Known risks are managed; unknown risks are not.

Deep-dive findings replace or augment the Sonnet findings they target. The Commander notes which findings were depth-validated in the report.

If no trigger conditions fire, skip Phase 2.75.

## Phase 3 — Cross-correlation (Commander)

After all reviewers report back, the Commander cross-correlates before writing the report. **This is the phase that catches compound failure chains.**

### Algorithm

0. **Deduplicate and check recurrence.** Reviewers with overlapping domains (e.g. sre-reviewer and observability-reviewer may both note a missing alert via `chain_potential`) can produce near-duplicate findings. Before cross-correlation: group findings by `surface` + `category`, collapse duplicates into the highest-severity instance, and record merged finding IDs in the compound chain. A finding that was deferred (`"defer to <seat>"`) is a cross-correlation signal, not a standalone finding — never refile it. Additionally: cross-reference each finding against the `## Unresolved findings` section of the recon brief. If a current finding matches a prior-run finding, **escalate its severity by one level** and add `"recurs from <date>"` to its evidence. A persistent gap that survived a previous review cycle is operationally worse than a new discovery.
1. Collect all deduplicated findings into a single list, keyed by `id`.
2. For each finding where `chain_potential: true`, ask: which other findings does this combine with to produce a higher-severity outcome?
3. Common compound patterns for reliability:
   - Untested rollback + missing alert → **silent failure with no recovery path** (one level above highest component)
   - No SLO defined + cascading blast radius → **unmeasured production impact, unbounded outage duration**
   - Proxy SLI + capacity assumption unverified → **SLO stays green while users suffer under load**
   - Toil with no automation + paging conditions → **on-call fatigue spiral, increasing MTTR over time**
   - Untested chaos abort path + production chaos experiment → **experiment becomes uncontrolled outage**
   - Missing runbook + long MTTD → **operator blind during blast radius window**
   - Irreversible schema migration + no tested rollback → **full outage with no recovery path**
   - Single replica + no HPA + capacity signal → **single point of failure with no automated recovery — any failure or overload requires manual intervention**
   - Error budget >80% consumed in current window + high-risk change → **shipping into negative safety margin — one more incident breaches SLO for the window**
   - Aggressive retry policy + no circuit breaker → **failure amplification — recovering dependency faces thundering herd, MTTR extends well beyond the original failure duration**
   - Network config change (NetworkPolicy/mesh) + no network-level alert + dependency failure symptoms → **network misconfiguration misattributed as dependency failure — on-call debugs the wrong layer for the duration of the outage**
4. For each chain, create a compound finding with:
   - `severity`: one level above the highest component finding
   - `chain`: list of component finding `id`s
   - `failure_narrative`: prose description of how the failure chain plays out end-to-end (3–5 sentences)

## Phase 4 — Report

Skeleton: `references/report-template.md`. Severity calibration: `references/severity-guide.md`. Tracker filing recipe: `references/tracker-integration.md`.

### 4a — Preview before write

Post the draft report to chat:

```
[draft report markdown]

Reply: save | amend <note> | accept <finding-id> "<rationale>" | discard
```

- `save` — write to `~/.claude/sre-team/<yyyy-mm-dd>-<slug>/report.md` and proceed to 4b
- `amend <note>` — revise per the note, re-preview
- `accept <finding-id> "<rationale>"` — mark the finding as risk-accepted with the stated rationale; it moves to the "Accepted Risks" section of the report, is excluded from bead filing, and is annotated `risk_accepted: true` so it does not escalate in future runs. Multiple `accept` commands can be chained before `save`.
- `discard` — clean up artifacts, no file written, no beads filed

### 4b — File findings to tracker (when beads is detected)

After the report file is written, file findings per the exact syntax and recipe in `references/tracker-integration.md`. Do not improvise commands — use only the live-tested forms documented there.

Order: component findings first (Critical / High / Medium), compound chains second. Update the on-disk report with returned bead IDs: frontmatter `primary-tracker-ids` and each finding's `**Tracker ID**`.

When beads is **not** detected: skip 4b entirely.

### 4c — Silent-promise guard

Before declaring the report finalized, grep the saved file. **Every Critical and High finding that is NOT marked `risk_accepted: true` must have a non-empty `**Tracker ID**`** when beads is detected. Missing IDs: file the bead now and update the report, or demote the finding's severity with rationale recorded. Accepted-risk findings are excluded from this guard.

### 4d — Emergent-insights write-back

If the run surfaced a structural lesson — a SKILL.md prescription contradicted by reality, a recon brief that produced false trust boundaries, a reviewer that consistently re-reports known beads — write it via `bd remember "<lesson, ≤200 chars>"` so the next run picks it up. The lesson also lands in the report's `Appendix — Emergent insights`.

When beads is not detected: skip `bd remember` but **always write the Appendix entry** — the lesson is recorded in the report regardless of tracker status.

## Stop early

User says "stop the sre review" at any phase:

1. Stop waiting on in-flight agents — do not process any further results from background spawns
2. Collect whatever findings have already arrived
3. Skip cross-correlation (or run abbreviated version on whatever is collected)
4. Write partial report with frontmatter `status: halted`, `cancelled_at: <phase>`, `cancellation_reason: user_request`
5. **Skip Phase 4b filing.** Tell the user explicitly: "no beads filed because the run was halted."
6. Clean up

## Roster

### Always-on

| Slug | Model | Mission |
|------|-------|---------|
| `sre-reviewer` | Sonnet | Blast radius, SLO integrity, rollback safety, failure modes, MTTD, toil |
| `chaos-reviewer` | Sonnet | Steady state definition, blast radius control, failure hypothesis quality, safety controls |
| `observability-reviewer` | Sonnet | Signal quality, alert coverage, detection latency, SLI validity |
| `release-reviewer` | Sonnet | Deployment blast radius, progressive delivery strategy, rollback path |
| `on-call-reviewer` | Sonnet | Runbook completeness, escalation clarity, 3am operability |

Always-on seats use **Sonnet** for the broad scan pass. Coverage is the goal — each seat looks across its entire domain for risk signals. Depth on flagged findings comes from Phase 2.75 targeted Opus deep-dives where the signal-to-cost ratio justifies it.

### Opt-in (Commander adds during recon based on detected context)

| Slug | Model | Add when |
|------|-------|---------|
| `capacity-reviewer` | Sonnet | Change affects request rate, fan-out, autoscaling, or replica count |
| `data-reviewer` | Sonnet | Change touches data storage, schema migrations, or data pipelines |
| `network-reviewer` | Sonnet | Change touches ingress, NetworkPolicy, DNS, TLS configuration, or service mesh config |

Opt-in seats use Sonnet by default. To upgrade a seat to Opus for a specific run, reply `promote <seat> to opus` in the plan card. Use this when: the service has complex distributed data guarantees (data-reviewer), unusual capacity constraints with multi-region fan-out (capacity-reviewer), or a dense service mesh with many overlapping policies (network-reviewer).

**Commander uses Opus** for cross-correlation (Phase 3) — synthesising compound failure chains from 60–100 findings across 7 reviewers is the hardest reasoning task in the protocol.

Per-agent methodology lives in `references/agents/<slug>.md`. **Load only when writing that agent's spawn prompt.**

## Notes for the Reliability Commander

**On severity calibration.** Use operational impact, not theoretical risk. An untested rollback on a low-traffic internal service is HIGH. An untested rollback on a payment critical path is CRITICAL. See `references/severity-guide.md`.

**On evidence standards.** Reviewers must cite specific file paths, config sections, or spec references. "Rollback might be hard" is not a finding. "The migration at `db/migrations/0042_add_column.sql` drops the `legacy_id` column with no rollback script" is a finding.

**On clean checks.** A domain with no findings goes in the "Clean Checks" section of the report, not omitted. Omission is ambiguous.

**On seat boundaries.** Each seat owns its domain exclusively. Use this table to route `chain_potential: true` defers from Phase 2:

| Domain | Owner | Others defer with |
|--------|-------|------------------|
| Alert rule existence, metric instrumentation, SLI/SLO definitions, dashboard coverage, detection latency | `observability-reviewer` | `chain_potential: true` + `"defer to observability-reviewer"` |
| Runtime failure modes, blast radius, MTTD, SLO existence, graceful degradation, toil | `sre-reviewer` | `chain_potential: true` + `"defer to sre-reviewer"` |
| Deployment strategy, progressive delivery, rollback mechanics, post-deploy verification, config drift | `release-reviewer` | `chain_potential: true` + `"defer to release-reviewer"` |
| Runbook completeness, escalation paths, alert actionability (links + correct metric names), 3am operability | `on-call-reviewer` | `chain_potential: true` + `"defer to on-call-reviewer"` |
| Chaos experiment safety, steady-state definition, blast radius for experiments, abort path readiness | `chaos-reviewer` | `chain_potential: true` + `"defer to chaos-reviewer"` |
| Load profile, headroom, autoscaling bounds, capacity assumptions | `capacity-reviewer` | `chain_potential: true` + `"defer to capacity-reviewer"` |
| Data durability, backup coverage, RPO/RTO, schema migration safety | `data-reviewer` | `chain_potential: true` + `"defer to data-reviewer"` |
| Network-layer reliability — ingress timeouts, DNS TTL, TLS rotation, NetworkPolicy changes, service mesh traffic policies | `network-reviewer` | `chain_potential: true` + `"defer to network-reviewer"` |

**On tool agnosticism.** This skill assumes nothing about the observability stack, chaos framework, deployment tooling, or incident platform in use. Reviewers assess what should exist and whether it does — not which specific tool implements it.

**On proactive triggering.** When triggered proactively (not from a user trigger phrase), the plan card includes the **proactive trigger reason** — e.g., "Detected new service at `services/billing/`. First production deployment." The user can `defer` or `go`.

**On tracker integration.** When beads is detected, Critical/High/Medium findings auto-file after the user `save`s the report. Full recipe in `references/tracker-integration.md`. Fails-safe.

## Reference files

- `references/agents/<slug>.md` — Per-reviewer methodology + spawn brief (load when writing that reviewer's spawn prompt)
- `references/severity-guide.md` — Operational impact-based severity rubric
- `references/recon-template.md` — Phase 1 brief format
- `references/report-template.md` — Phase 4 markdown skeleton
- `references/tracker-integration.md` — Beads detection, severity → priority map, finding-filing recipe, silent-promise guard
