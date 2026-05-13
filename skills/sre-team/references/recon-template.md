# SRE Review Recon Brief

<!-- Commander writes this during Phase 1. All reviewer spawn prompts point to this file. -->

## Scope

<!-- What is being reviewed: repo root, subtree, specific service, or PR diff -->

## Architecture summary

<!-- Services involved, their dependencies, data stores, external integrations -->

## Deployment context

<!-- Deploy strategy visible in config (rolling, canary, blue-green, feature-flagged) -->
<!-- Rollback approach if documented -->

## Deployment values (production)

<!-- Actual values from GitOps/argo-apps/helm-values files for this service -->
<!-- Include: replica count, HPA bounds, alerting flags, resource limits, feature flag state -->
<!-- Source path: <path to values file, or "not found — recon gap"> -->
<!-- If not found: reviewers must assume production config is unknown -->

## Existing SLOs

<!-- SLO definitions found in docs, config, or code — or "none found" -->

## Recent changes

<!-- Last 10–15 commits on the main branch — helps reviewers prioritise new code paths -->
<!-- git log --oneline -15 -->

## Incident signals (git log)

<!-- Surface-level scan: recent incidents, rollbacks, or hotfixes visible in commit history — or "none found" -->
<!-- git log --oneline --all | grep -i 'incident\|hotfix\|rollback\|revert' -->
<!-- For deeper postmortem content, see Failure history below -->

## Failure history

<!-- Summaries extracted from postmortem documents found in the repo -->
<!-- For each postmortem: what failed, contributing factors, action item status (resolved / unresolved) -->
<!-- Unresolved action items are a finding signal for on-call-reviewer -->
<!-- "No postmortem documents found" if absent — note whether this is expected (new service) or a gap -->

## Unresolved findings (prior sre-team runs)

<!-- Critical and High findings from the most recent prior report at ~/.claude/sre-team/ for this service -->
<!-- Format: [date] [severity] [finding one-liner] [original finding id] -->
<!-- "No prior sre-team history" if this is the first run -->
<!-- Reviewers: treat these as known risk — assess whether conditions have changed, not re-discover -->

## Error budget posture

<!-- Current error budget consumption or burn rate if documented — or "SLO found but no budget tracking" or "no SLO found" -->
<!-- High burn rate (>50% consumed in current window) escalates severity calibration for all seats -->

## Dependency reliability

<!-- For each external dependency: whether an SLO or reliability guarantee was found -->
<!-- Format: dependency name | SLO found (yes/no) | known failure history (if any) -->
<!-- Dependencies with no SLO are unknown reliability surfaces — flag for sre-reviewer -->

## Known issues from tracker

<!-- Only present when beads is detected. Lists open bugs and in-progress items. -->
<!-- Reviewers: if your finding matches a listed bead, do not refile — report chain_potential: true with the bead id in evidence instead. -->

## Constraints from CLAUDE.md

<!-- Architecture constraints, known risks, out-of-scope areas extracted from CLAUDE.md -->
<!-- "No CLAUDE.md present" if absent -->

## Opt-in seats activated

<!-- List of opt-in seats added for this run and why -->
