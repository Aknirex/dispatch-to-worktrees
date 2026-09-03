# Session protocol

The close-out contract every dispatched session obeys. The dispatcher pastes the relevant clauses into each dispatch prompt (templates in `dispatch-prompts.md`), so the contract travels with the session. Paths and defaults below are overridden by repo conventions.

## The close-out

A dispatched session's last action is its close-out: it tells the dispatcher the stage finished, and only then ends. Ordering is the point — the message precedes the end, never the reverse. A session that ends without a close-out has not finished, even if its work is committed.

Close-out has two parts:

1. A `SESSION_DONE.md` marker written in the session's own worktree (below).
2. One `agent_manager` prompt to the dispatcher session (below).

## 1. The marker file: `SESSION_DONE.md`

Path:

```
.kilo/worktrees/<worktree-name>/SESSION_DONE.md
```

First line is exactly one of:

```
PASS - <one-line summary of what was achieved>
FAIL - blockers: <short comma-separated blocker list>
```

Below the first line goes the evidence section the role requires (see Evidence). The marker path is local tool state, not a tracked source file, so writing it never counts as a source change. A session that cannot finish its stage still writes `FAIL -` with blockers and preserves its state (committed work, WIP notes).

## 2. The message to the dispatcher

Immediately after writing the marker, the session sends the dispatcher one `agent_manager` prompt containing:

- session id and worktree name,
- branch tip hash,
- what it ran and the results (suite names, passed/total, run mode),
- verdict (`PASS`/`FAIL`) and, for a review, the findings summary.

The dispatcher advances the pipeline on receipt of this message. Nothing polls.

## Evidence expectations

| Stage | Minimum evidence in marker + message |
|---|---|
| Worker `PASS` | Branch tip hash; build result; suites run with counts and run mode; anything left undone |
| Worker `FAIL` | Same plus the blocker list |
| Reviewer `PASS` | Every acceptance line judged; build/test evidence the reviewer itself ran |
| Reviewer `FAIL` | Every blocker mapped to the acceptance line it hits; non-blocking notes listed separately |

## Reviewer read-only discipline

- May: read anything; run builds and tests (their untracked, gitignored artifacts are fine); write the single `SESSION_DONE.md` in its own snapshot worktree.
- Must not: modify any tracked file, commit, push, edit tickets, or write any other file anywhere.
- A reviewer that writes anywhere else voids its own review: the dispatcher resets the snapshot to the branch tip and re-dispatches. Findings live in the marker and the close-out message — nothing else needs to be written.

## Worker discipline

- May: work freely inside its own worktree and branch, commit on the ticket branch.
- Must not: touch main, merge anything, delete its worktree, modify other worktrees, or write outside its worktree except the marker and the close-out message.
- Never stage or commit the repo's secret files.

## Model registry

New sessions are started with an explicit model + variant from the repo's registry (the repo's `AGENTS.md` usually records which model codes, which reviews, and which variant each uses). Never start a dispatched session on an unregistered default. A session keeps its model for its whole life; a successor inherits the same registered role model.

## Fix rounds

- A fix round returns the blocker list to the same coding session when it is still within budget; otherwise the dispatcher resets a successor worktree to the branch tip and the successor continues the same ticket branch.
- After the fix is committed, the dispatcher resets the review snapshot to the new tip and re-dispatches the review. Rounds are counted by review dispatches, not by fixes.
