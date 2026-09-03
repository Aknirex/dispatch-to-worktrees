# Session protocol

The contract every dispatched session obeys. The dispatcher pastes the relevant clauses into each dispatch prompt (templates in `dispatch-prompts.md`) so the contract travels with the session. Paths and defaults below are overridden by repo conventions.

## Completion marker: `SESSION_DONE.md`

When a session finishes its assigned stage it writes a marker file at:

```
.kilo/worktrees/<worktree-name>/SESSION_DONE.md
```

The first line is exactly one of:

```
PASS - <one-line summary of what was achieved>
FAIL - blockers: <short comma-separated blocker list>
```

Follow the first line with whatever evidence section the role requires (see below). The marker path is local tool state, not a tracked source file, so writing it never counts as a source change.

A marker is written exactly once per completed stage. A session that cannot finish its stage still writes `FAIL -` with blockers and preserves its state (committed work on the branch, WIP notes) before reporting.

## Completion brief

Immediately after writing the marker, the session sends the dispatcher one `agent_manager` prompt containing:

- its session id and worktree name,
- the branch tip hash it finished on,
- what it ran and the results (suite names, passed/total, run mode),
- its verdict (`PASS`/`FAIL`) and, for a review, the findings summary.

The brief is how the dispatcher learns the stage is done — this is what replaces polling. The dispatcher advances the pipeline on receipt.

## Evidence expectations

| Stage | Minimum evidence in marker + brief |
|---|---|
| Worker `PASS` | Branch tip hash; build result; test suites run with counts and run mode; note of anything left undone |
| Worker `FAIL` | Same as `PASS` plus the blocker list |
| Reviewer `PASS` | Verdict with every acceptance line judged; build/test evidence the reviewer itself ran |
| Reviewer `FAIL` | Verdict plus each blocker **mapped to the acceptance line it hits**; non-blocking notes listed separately |

## Reviewer read-only discipline

- **May**: read anything; run builds and tests (their untracked, gitignored artifacts are fine); write the single `SESSION_DONE.md` in its own snapshot worktree.
- **Must not**: modify any tracked file, commit, push, edit tickets, or write any other file anywhere.
- A review session that modifies any file outside its `SESSION_DONE.md` **voids its own review**. The dispatcher resets the snapshot to the branch tip and re-dispatches. Findings live in the marker and the brief — nothing else needs to be written.

## Worker discipline

- **May**: work freely inside its own worktree and branch, commit on the ticket branch.
- **Must not**: touch `main`, merge anything, delete its worktree, modify other worktrees, or write outside its worktree except the marker and the brief to the dispatcher.
- Never stage or commit the repo's secret files.

## Model registry

New sessions are started with an explicit model + variant from the repo's registry (the repo's `AGENTS.md` typically records which model runs code, which runs review, and which variant each uses). Never start a dispatched session on an unregistered default. Sessions created under this policy keep their model for their whole life; a successor session inherits the same registered role model.

## Fix rounds

- A fix round returns the blocker list to the **same coding session** when it is still within budget; otherwise the dispatcher resets a successor worktree to the branch tip and the successor continues the same ticket branch.
- After the fix is committed, the dispatcher resets the review snapshot to the new tip and re-dispatches the review. Rounds are counted by review dispatches, not by fixes.
