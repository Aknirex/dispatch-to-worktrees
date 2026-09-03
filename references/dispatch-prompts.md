# Dispatch prompts

Templates for spawning sessions with `agent_manager`. Fill in every `<angle-bracket>` field from the repo rules. Send the prompt whole, not summarized.

## Worker session

```
Role: implement ticket <nn> <title> in this worktree.

Ticket (source of truth): <path>
Acceptance checklist: <path or list>
Repo rules: <repo AGENTS.md>
Runtime policy: <repo pointer>

Worktree: <path>
Branch: <ticket-<nn>-<slug>> (base: main @ <sha>)
Scope: <what this round touches>

Do:
- Implement the acceptance items. Commit on this branch only.
- Run only the affected tests for this round: <command or filter>.
- Do not touch main. Do not merge. Do not delete this worktree.
- Never commit: <secret file names>.

Close-out:
- When done, or blocked:
  1. Write .kilo/worktrees/<wt>/SESSION_DONE.md.
     First line: "PASS - <summary>" or "FAIL - blockers: <list>".
     Below: tip hash, tests + counts + mode.
  2. Send the dispatcher (session <ses_dispatcher_id>) one message with the same info.
  3. Only then end. The message is part of the work.

Model: <registry model + variant>
```

## Reviewer session

```
Role: read-only review of ticket <nn> <title>.

Snapshot: <path> @ <sha>
Change: <ticket branch> vs main @ <sha>
Ticket: <path>
Acceptance checklist: <path or list> — judge only against this.

Do:
- Read the change. Run builds/tests you need, in this snapshot.
- Wait on log output. No logs for <timeout>: kill and report. No sleep polling.
- PASS: show each acceptance line judged, with evidence.
- FAIL: list blockers, each tied to an acceptance line. Notes separate.

Read-only:
- Write exactly one file: .kilo/worktrees/<wt>/SESSION_DONE.md.
- No commits, no pushes, no ticket edits. Any other write voids this review.

Close-out:
- When done:
  1. Write the SESSION_DONE.md marker (first line PASS/FAIL, then evidence).
  2. Send the dispatcher (session <ses_dispatcher_id>) one message:
     verdict, tip hash, blockers with acceptance lines.
  3. Only then end.

Model: <registry review model + variant>
```

## Fix round (same coding session)

```
Continue ticket <nn> <title>. Review of <sha> failed. Fix these blockers:

<blockers, each tied to an acceptance line>

Do only this list. If one item does not fit, say so in your close-out.
Re-run the affected tests after fixing.
Finish with the SESSION_DONE.md marker + close-out message to the dispatcher.
```
