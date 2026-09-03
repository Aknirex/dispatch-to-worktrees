# Dispatch prompts

Fill-in templates for spawning sessions via `agent_manager`. Replace every `<angle-bracket>` field from the repo's conventions at dispatch time: ticket paths, test commands, registry models/variants, marker paths, secret-file names. Delete lines that do not apply to the repo.

Send the prompt whole, never summarized, so the contract travels with the session.

## Worker (coding) session

```
Role: implement ticket <nn> <title> in this worktree.

Ticket (source of truth): <ticket path>
Acceptance checklist (source of truth for "done"): <path or paste list>
Repo conventions: <repo AGENTS.md and pointed-to docs>
Runtime policy: <repo runtime policy pointer>

Worktree: <path>  Branch: <ticket-<nn>-<slug>> (based on main @ <sha>)
Scope for this round: <what this round must and must not touch>

Do:
- Implement the acceptance items. Commit on this branch only, messages following repo style.
- Run only the affected subset for this round: <test command / filter>.
- Do not touch main. Do not merge. Do not delete this worktree.
- Never stage or commit: <repo secret file names>.

Close-out (read this now and confirm you will follow it):
- When the work is done, or blocked: FIRST write .kilo/worktrees/<wt>/SESSION_DONE.md
  (first line "PASS - <summary>" or "FAIL - blockers: <list>", then evidence:
  tip hash, suites + counts + run mode).
- THEN send the dispatcher (session <ses_dispatcher_id>) one message with the same content.
- Only after the message is sent, end. Sending the message is part of the work.

Model: <registry model + variant>
```

## Reviewer session

```
Role: read-only review of ticket <nn> <title>.

Snapshot: <path> @ branch tip <sha>
Change under review: <ticket branch> vs base main @ <sha>
Ticket (source of truth): <ticket path>
Acceptance checklist: <path or paste list> — judge only against this, plus the repo's
recorded coding standards where they exist. Every finding maps to an acceptance line.

Do:
- Read the change. Run whatever build/test evidence you need in this snapshot, waiting on
  log output (kill + report any process silent for <timeout>; do not poll with sleeps).
- Verdict PASS: show each acceptance line judged, with the evidence you ran.
- Verdict FAIL: list blockers, each tied to the acceptance line it hits; put anything else
  (judgment, optional) in a separate non-blocking notes list.

Read-only discipline:
- Write exactly one file: .kilo/worktrees/<wt>/SESSION_DONE.md. No commits, no pushes,
  no ticket edits. Writing anywhere else voids this review; findings live in the marker
  and your close-out message.

Close-out (read this now and confirm you will follow it):
- When the review is finished: FIRST write the SESSION_DONE.md marker
  (first line "PASS - ..." or "FAIL - blockers: ...", then evidence: verdict, acceptance
  lines judged, suites + counts + run mode you ran).
- THEN send the dispatcher (session <ses_dispatcher_id>) one message: verdict, tip hash,
  blocker list with acceptance mappings.
- Only after the message is sent, end.

Model: <registry review model + variant>
```

## Fix round (reuse of an existing coding session)

```
Continue ticket <nn> <title>. The review of <sha> failed. Address these blockers:

<blocker list, each tied to the acceptance line it hits>

Do not chase anything outside this list; if you judge an item does not belong, say so in
your close-out instead of silently expanding scope. Re-run the affected subset after fixing.
Finish with the usual SESSION_DONE.md marker + close-out message to the dispatcher.
```
