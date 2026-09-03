---
name: dispatch-to-worktree
description: "Run a review-gated ticket pipeline from a resident dispatcher session: turn ready tickets into isolated git worktree agent sessions, supervise them via completion briefs (分派票据/任务到 worktree 会话的调度员工作流), gate every merge behind a read-only review snapshot, and adjudicate review findings before merging or dispatching a fix round. Use when the user asks to dispatch/分派/派发 tickets or tasks into worktree sessions, supervise coding sessions to completion, 派审查/放行合并 or review-and-merge a finished branch, adjudicate review blockers against an acceptance checklist, or merge and clean up completed worktree sessions."
---

# Dispatch to Worktree

Orchestrate ticket-to-merge work across isolated git worktree agent sessions from a single resident **dispatcher** session in kilocode Agent Manager (AM). The dispatcher is the hub: the user talks to the dispatcher, the dispatcher spawns and supervises worktree sessions, and nothing reaches `main` without passing a read-only review gate.

This skill defines the **process**. Project specifics — ticket source, naming, test commands, model registry, runtime policy, marker paths — come from the target repository's own conventions (its `AGENTS.md` and pointed-to docs). Where a repo convention exists it overrides every default below; where it is silent, the defaults apply.

Read the two disclosed references when their step says so:

- [`references/session-protocol.md`](references/session-protocol.md) — the completion-brief contract every dispatched session obeys.
- [`references/dispatch-prompts.md`](references/dispatch-prompts.md) — fill-in prompt templates for worker and reviewer sessions.

## Roles

- **Dispatcher** — this session. A resident AM *local* session (not a worktree). Only role that talks to the user. Advances the pipeline on evidence.
- **Worker** — a coding session in a ticket worktree. Implements one ticket, commits on its branch, reports completion.
- **Reviewer** — a read-only session on a **snapshot** worktree at the coding branch tip. Judges the change against the ticket's acceptance checklist and reports a verdict. May write no file except its `SESSION_DONE.md`.

## Pipeline at a glance

```
ready tickets
   │ dispatch                      (worker worktree)
   ▼
IMPLEMENT ──PASS brief──▶ VERIFY ──▶ REVIEW ──PASS──▶ MERGE (no-ff) ──▶ cleanup
   │                          (snapshot @ tip)          + record
   │  FAIL brief                  │ FAIL
   ▼                             ▼
 block/report ──────◀── ADJUDICATE ──▶ fix round ──▶ reset snapshot ──▶ re-review
                      (line-by-line vs acceptance;        ▲
                       convergence cap at 3 rounds)       │
                      └────────── loop ───────────────────┘
```

## Steps

Each step ends with a completion criterion.

### 1. Orient

- Read the repo's governing conventions: `AGENTS.md` and whatever it points to (issue tracker, triage labels, test/runbook, model registry, runtime policy). These override every default in this skill.
- `agent_manager` action=list and take inventory: every live session, its worktree, its stage. Identify stale worktrees (created for a purpose already satisfied), sessions waiting on a question, and finished-but-unmerged branches.

**Done when:** you can name every live session/worktree and its purpose, and you know the repo's test commands, ticket location, and per-role review model registry.

### 2. Dispatch a ticket

For each ready ticket (repo tracker status says it is agent-ready, it has an acceptance checklist, it is not blocked):

- Create one worktree session per ticket via `agent_manager` (worktree mode), branch and worktree named per repo convention. Defaults: worktree name `ticket <nn> <title>`, branch `ticket-<nn>-<slug>`.
- Build the worker prompt from the template in `references/dispatch-prompts.md`. It must carry: ticket reference + acceptance checklist, the test scope for this round, the runtime policy pointer, and the explicit model + variant from the repo registry (never an unregistered default).
- After creation: `agent_manager` action=list to confirm the session; then, in the new worktree, `git reset --hard` to the repo baseline and run the repo's environment bootstrap if one exists (e.g. shared build cache junction).

**Done when:** every dispatched ticket has a live session, confirmed by list, sitting on a branch at the baseline, with a prompt that names the acceptance checklist and the required model/variant.

### 3. Supervise to completion

No polling. Workers and reviewers **signal** completion: they write their `SESSION_DONE.md` marker, then send the dispatcher a completion brief via `agent_manager` prompt (contract in `references/session-protocol.md`). The dispatcher processes briefs as they arrive and advances the pipeline.

- On a `PASS` brief: move to **Verify before gate**.
- On a `FAIL` brief: read the blockers. A worker that stopped with blockers it cannot resolve is the dispatcher's decision, not the worker's — decide whether to send a fix round, preserve its state (branch tip, WIP notes), or report to the user with the blocker list.
- On silence: a dispatched session that goes quiet beyond the repo's grace period is a problem the worker cannot see. Check it via list/prompt once; if it is stuck, stop it and preserve its worktree state for a successor.
- Maintain a running ledger per ticket (stage, tip hash, test counts, verdicts) so you never have to re-derive state from the repo.

**Done when:** every in-flight ticket has reached a `PASS` brief, a fix round, or a user-visible stopped report with its state preserved.

### 4. Verify before gate

A brief is a claim, not evidence. Before any review or merge, confirm on the repo itself:

- The branch tip hash in the brief exists on the branch.
- The worktree is clean (no stray tracked changes left by the worker).
- Test evidence is present and specific: suite name, passed/total counts, run mode.
- No credentials or the repo's secret files appear in the change set.

**Done when:** the ticket is either cleared to review or sent back to the worker with the exact gaps listed.

### 5. Dispatch read-only review

- Create a review **snapshot** at the coding branch tip, not a new development branch: worktree name `review <nn>`, branch `review-<nn>-<slug>` (defaults), then `git reset --hard` to the coding branch tip.
- Dispatch the reviewer session(s) with the prompt template in `references/dispatch-prompts.md`: judge only against the ticket's acceptance checklist plus the repo's recorded standards; report verdict and blockers mapped to acceptance lines; follow the read-only discipline.
- Use the review model/variant from the repo registry. If the repo policy calls for double review, dispatch the two registered review models in parallel against the same snapshot.
- Reviewer writes exactly one file: its `SESSION_DONE.md`. Any other write voids the review.

**Done when:** the review is dispatched at the exact tip with registered reviewer model(s), and the reviewer prompt states the read-only discipline.

### 6. Adjudicate the verdict

On the reviewer's completion brief:

- **PASS** → merge gate (step 7).
- **FAIL** → read the blocker list **line-by-line against the ticket's acceptance checklist**:
  - A blocker that hits an acceptance item → **fix round**: prompt the same coding session (or a successor worktree reset to the branch tip if the session is spent) with the blocker list.
  - A blocker that hits no acceptance item (typically demands deeper verification than the acceptance requires, e.g. an automation level the ticket never asked for) → **downgrade** to a judgment/manual note, record it on the ticket, and do not let it block the merge.
- After a fix round, reset the review snapshot to the new tip and re-dispatch the review.
- **Convergence cap**: if the same ticket has not converged after 3 review rounds (repo may set another number), stop looping. Exercise the adjudication authority with the user in the loop and re-examine the review boundary — are the reviewers judging the right thing against the right checklist?

**Done when:** every outstanding review round ends in `PASS`, a recorded downgrade, or an explicit convergence decision visible to the user.

### 7. Merge and record

For each ticket that cleared the gate:

- Merge **no-ff** into `main` from the repo's main checkout: `git merge --no-ff <ticket-branch>`.
- Test layering: the full suite runs **once** at this gate, not on every fix round (fix rounds run the affected subset; documentation-only rounds run nothing). If the gate run did not happen at the review stage, run it now before merging.
- Write the repo's merge record (default: `docs/temp/merge-<date>-ticket-<nn>.md`) capturing: branch and tip hash, merge hash, verification summary, review verdict.
- Advance the ticket status per the repo tracker conventions.

**Done when:** the branch is merged no-ff into `main`, the merge record exists, and the ticket status is advanced.

### 8. Clean up

- For each session whose work is merged, voided, or superseded: `agent_manager` stop the session, then remove its worktree and branch.
- Before deleting branches, re-check with `git branch --merged main` (or the repo's supersession list) and confirm the owning session is stopped via list. Never delete from memory.
- If the repo tracker records status separately from git (e.g. `Status:` lines in ticket files), reconcile it.

**Done when:** only genuinely in-flight work remains; merged and stale worktrees and branches are gone.

## Dispatcher iron rules

These hold on every pass through the pipeline:

- **IDs from list, not memory.** Resolve every session and worktree from `agent_manager` action=list output at the moment you act.
- **Review snapshots are frozen.** Reviewers write only `SESSION_DONE.md`. Any other file change voids the review: reset the snapshot to the branch tip and re-dispatch.
- **Evidence before merge.** Merge only on verified evidence (step 4). Never merge on a claim in a brief.
- **Reset + bootstrap every new worktree.** A fresh worktree is `reset --hard` to its baseline and put through the repo's environment bootstrap before its session starts.
- **Logs, not blind sleep.** When the dispatcher itself runs long child processes (builds, test runs), start them in log mode and wait on log output. No log progress within the repo's timeout (default 120s) means the process is hung: kill it and report. Never poll with sleeps.
- **Test layering.** Fix rounds run the affected subset; the full suite runs once at the review/merge gate; documentation-only rounds run no tests and no build.
- **Credentials stay out.** Never let the repo's secret files reach any branch, commit, log, or report.
- **User talks to the dispatcher.** Worktree sessions never turn to the user; questions and blockers flow back through the dispatcher.
