# Session protocol

The close-out contract for dispatched sessions. The dispatcher puts the relevant parts into each dispatch prompt. Repo rules override the defaults here.

## Close-out

Order matters: message first, end second. A session that ends without a close-out did not finish.

Two parts:

1. Write `SESSION_DONE.md` in its worktree.
2. Send one `agent_manager` prompt to the dispatcher.

### 1. The marker: `SESSION_DONE.md`

Path: `.kilo/worktrees/<worktree-name>/SESSION_DONE.md`

First line, exactly one of:

```
PASS - <one-line summary>
FAIL - blockers: <list>
```

Add evidence below the first line. The marker is local tool state, not a tracked file. Writing it is not a code change. A session that cannot finish still writes `FAIL -` with blockers, then keeps its state.

### 2. The message

After the marker, send one prompt with:

- session id, worktree name
- branch tip hash
- tests run: suites, passed/total, mode
- verdict: PASS or FAIL
- review: findings summary

The dispatcher moves the pipeline when this message arrives. Nothing polls.

## Evidence needed

| Stage | Minimum evidence |
|---|---|
| Worker PASS | tip hash, build result, test counts, what is left undone |
| Worker FAIL | same, plus blockers |
| Reviewer PASS | every acceptance line judged, plus evidence you ran |
| Reviewer FAIL | every blocker mapped to an acceptance line; notes separate |

## Reviewer rules

- Can: read files, run builds and tests, write its own `SESSION_DONE.md`.
- Cannot: change files, commit, push, edit tickets, write any other file.
- Writing anywhere else voids the review. The dispatcher resets the snapshot and reviews again.

## Worker rules

- Can: work in its worktree, commit on the ticket branch.
- Cannot: touch main, merge, delete its worktree, change other worktrees, write outside its worktree (except the marker and the close-out message).
- Never commit secret files.

## Models

Start new sessions with the model + variant from the repo registry. Do not guess. A session keeps its model. A successor uses the same role model.

## Fix rounds

- Send blockers to the same session if it can still work. Else reset a new worktree to the branch tip and continue the same branch.
- After the fix, reset the review snapshot to the new tip and review again. Count review rounds, not fix rounds.
