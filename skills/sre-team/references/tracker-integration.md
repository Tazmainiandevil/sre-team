# Tracker Integration

The sre-team plugin integrates with the **beads** tracker (https://github.com/gastownhall/beads) when it is present. All tracker integration fails-safe — projects without beads behave identically to projects with no tracker configured.

> **Status:** All commands below are live-tested. Detection, `bd ready`, `bd prime`, `bd create --silent`, `bd dep add`, `bd dep tree`, `bd remember`, `bd forget`, and `bd close -r` all confirmed working.

## Detection

Run from the project's git root:

```bash
command -v bd > /dev/null 2>&1 && bd where > /dev/null 2>&1
```

Both conditions must pass: `bd` must be in PATH **and** a beads database must be reachable from the current directory (walks up from git root). This avoids false-positive detection when `bd` is installed globally but the project under review has no beads database.

If the check succeeds, beads is the tracker for this run.

## Phase 1 — Recon enrichment

When beads is detected, gather open bugs before spawning reviewers:

```bash
bd ready -t bug
```

Inline the result as a `## Known issues from tracker` section in `recon.md`. This lets reviewers cross-reference against known issues rather than re-reporting them.

Also run the skill self-audit to surface lessons from prior sre-team runs:

```bash
bd prime --memories-only | grep -i 'sre-team\|reliability\|reviewer'
```

If any memory entry contradicts a prescription in SKILL.md, memory wins — flag the contradiction in `recon.md`, follow memory, and record the drift in the report's emergent insights appendix.

## Severity → priority mapping

| Severity | bd priority (`-p`) |
|----------|--------------------|
| Critical | 0 |
| High | 1 |
| Medium | 2 |
| Low | 3 |
| Info | (not filed) |

## Phase 4b — Filing findings

File component findings first, then compound chains.

### Component findings (Critical / High / Medium)

```bash
BEAD_ID=$(bd create "[sre-team] <finding one-liner>" \
  -t bug -p <mapped-priority> \
  -d "Surface: <surface>\nConditions: <conditions>\nEvidence: <evidence>" \
  --silent)
```

`--silent` returns only the bead ID, suitable for capturing in a variable. Update the report's finding block `**Tracker ID**` field with the returned ID.

### Compound chains

```bash
CHAIN_ID=$(bd create "[sre-team][chain] <chain name>" \
  -t bug -p 0 \
  -d "<failure narrative>" \
  --silent)

bd dep add $CHAIN_ID $COMPONENT_BEAD_ID_1
bd dep add $CHAIN_ID $COMPONENT_BEAD_ID_2
```

The chain bead blocks on its component beads. Use `bd dep tree $CHAIN_ID` to verify the dependency graph after filing.

**Important:** Compound chain beads are tracking artifacts, not directly actionable items. On-call fixes the component beads; the chain bead closes automatically when all components close. Do not assign the chain bead to an engineer for direct remediation — assign the component beads instead. The chain bead exists to make the compound risk visible in the tracker and to prevent components from being closed independently without considering the chain.

## Silent-promise guard

Before declaring the report finalized when beads is detected, grep the saved file:

```bash
grep -c 'Tracker ID.*N/A\|Tracker ID:$' report.md
```

If any Critical or High finding has an empty or N/A tracker ID, either:
1. File the bead now and update the report, **or**
2. Demote the finding's severity (with rationale recorded in the report) and re-run the guard.

Do not exit Phase 4 with unfiled C/H findings.

## Emergent-insights write-back

```bash
bd remember "<lesson, ≤200 chars>"
```

Run only when the run surfaced a structural lesson worth persisting. Skip when beads is not detected — the appendix entry in the report is the only handle.
