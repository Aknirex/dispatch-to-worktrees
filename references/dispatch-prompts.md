# Dispatch prompts

Fill-in templates the dispatcher uses when spawning sessions via `agent_manager`. Replace every `<angle-bracket>` field from the repo's conventions at dispatch time: ticket paths, test commands, registry models/variants, marker paths, secret-file names. Delete any line that does not apply to the repo.

The prompt template is shipped whole — never summarized — so the contract travels with the session.

## Worker (coding) session

```
Role: implement ticket <nn> <title> in this worktree.

Ticket (source of truth): <ticket path>
Acceptance checklist (source of truth for "done"): <path or paste list>
Repo conventions: <repo AGENTS.md and pointed-to docs>
Runtime policy: <repo runtime policy pointer, e.g. build cache/bootstrap, run mode>

Worktree: <path>  Branch: <ticket-<nn>-<slug>> (based on main @ <sha>)
Scope for this round: <what this round must and must not touch>

Do:
- Implement the acceptance items. Commit on this branch only, messages following repo style.
- For this round's changes run only the affected subset: <test command / filter>. Do not run the full suite.
- Do not touch main. Do not merge. Do not delete this worktree.
- Never stage or commit: <repo secret file names>.

Done means:
- Every acceptance item is implemented and its evidence recorded (commits, test counts).
- Write .kilo/worktrees/<wt>/SESSION_DONE.md, first line exactly
  "PASS - <one-line summary>"  or  "FAIL - blockers: <list>",
  then below it: branch tip hash, what you ran with passed/total counts and run mode.
- Then prompt the dispatcher (session <ses_dispatcher_id>) once with a brief:
  session id, tip hash, suites + counts, verdict, blockers if any.

Model: <registry model + variant>
```

## Reviewer session

```
Role: read-only review of ticket <nn> <title>.

Snapshot: <path> @ branch tip <sha>
Change under review: <ticket branch> vs base main @ <sha>
Ticket (source of truth): <ticket path>
Acceptance checklist: <path or paste list>  — judge ONLY against this (plus the repo's
recorded coding standards where they exist). Every finding must map to an acceptance line.

Repo conventions: <repo AGENTS.md and pointed-to docs>

Do:
- Read the change; run whatever build/test evidence you need, in this snapshot, in log mode
  (no blind sleep polling; kill + report any process silent for <timeout>).
- Verdict PASS: state it and show each acceptance line judged, with evidence you ran.
- Verdict FAIL: list blockers, each tied to the acceptance line it hits; put anything else
  (judgment, optional) in a separate non-blocking notes list.

Read-only discipline:
- You may write exactly one file: .kilo/worktrees/<wt>/SESSION_DONE.md
  (first line "PASS - ..." or "FAIL - blockers: ...", then evidence).
- Any other file write voids this review. No commits, no pushes, no ticket edits.
- Findings live in that marker and in your brief to the dispatcher — nothing else.

Then prompt the dispatcher (session <ses_dispatcher_id>) once: verdict, tip hash,
evidence you ran (suites + counts + run mode), blocker list with acceptance mappings.

Model: <registry review model + variant>
```

## Fix round (reuse of an existing coding session)

```
Continue ticket <nn> <title>. The review of <sha> failed. Address these blockers:

<blocker list, each tied to the acceptance line it hits>

Do not chase anything outside this list; if you judge an item does not belong, say so in
your brief instead of silently expanding scope. Re-run the affected subset after fixing.
Finish with the usual SESSION_DONE.md marker + brief to the dispatcher.
```
