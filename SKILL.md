---
name: dispatch-to-worktrees
description: "Run work across git worktree sessions from one resident dispatcher session in kilocode Agent Manager (调度员把票据分派到 worktree 会话并监管到合并). Use when asked to dispatch/分派/派发 tickets or tasks into worktree sessions and supervise them to merge, gate a finished branch behind read-only review, or clean up completed worktree sessions."
---

# Dispatch to Worktree

One dispatcher session manages many worktree sessions. Each worktree session works on one ticket and tells the dispatcher when it is done. The dispatcher checks the work, gets a review, then merges.

Rules in the repo's `AGENTS.md` override the defaults below.

## Main rule: close-out before end

Every session you dispatch must follow this rule:

> When done, or blocked: send the dispatcher a close-out message first. Then end.

A close-out message is one `agent_manager` prompt to the dispatcher session. It contains:

- session id
- branch tip hash
- verdict: PASS or FAIL
- tests run, with counts
- blockers, if any

The session also writes a `SESSION_DONE.md` marker. See [`references/session-protocol.md`](references/session-protocol.md).

Put this rule in every dispatch prompt. Ask the session to confirm it before starting.

A session that ends without a close-out did not finish. Treat it as lost. Keep its worktree state for the next session.

## Before you start: session check

Act as dispatcher only when this session can really do it. Before anything else:

1. Run `agent_manager` action=list.
2. If the tool is missing, the call fails, or this is not a kilocode Agent Manager session: stop. Tell the user the dispatcher must run as a resident session inside kilocode Agent Manager, then end.

No dispatch, no worktree creation, until the check passes.

Note your own session id from that list. Every dispatch prompt tells the worker to send its close-out to this id (`<ses_dispatcher_id>` in the templates). Use your real id each time; never a fixed value.

## Steps

1. Find ready tickets
   - Use the source named in the repo's `AGENTS.md`, if one exists.
   - If not, find the ready list yourself: local ticket files, or the remote tracker through its CLI (`gh issue list` for GitHub, a Jira CLI or API), or infer from the user's request which issues to take.
   - A ticket is ready when: it is agent-ready, not blocked, and has a definition of done / acceptance list. Skip the rest.
   - If the source is not configured and you cannot tell which tickets to take: ask the user. Do not guess.
   - Done when: you have a concrete list of ready tickets, or you asked.

2. Dispatch a ticket
   - Create one worktree session per ticket. Branch and worktree names follow repo rules (default: `ticket <nn> <title>`).
   - The prompt must include: ticket reference + acceptance checklist, test scope, runtime policy, model + variant from the repo registry, and the close-out rule. Template: [`references/dispatch-prompts.md`](references/dispatch-prompts.md).
   - After creation: reset the worktree to the baseline, run the repo bootstrap, and check the session with `agent_manager` list.
   - Done when: every session is live, on the baseline, and its prompt has the close-out rule.

3. Wait for close-outs
   - Do not poll. A ticket moves forward only when its close-out arrives.
   - Keep a ledger per ticket: stage, tip hash, test counts, verdict.
   - If a session is silent too long: check it once, keep its state, stop it, start a new session on the same branch.
   - Done when: every ticket has a close-out or a successor session.

4. Check the work
   - A close-out is a claim, not proof. Check on the repo:
     - branch tip hash exists
     - worktree is clean
     - test counts are real
     - no secret files in the change
   - If something is missing, send the ticket back with the exact list.
   - Done when: the ticket is ready for review, or returned with gaps.

5. Gate with a review snapshot
   - Create a snapshot at the branch tip (default: `review <nn>`). Reset it to the tip.
   - Dispatch reviewer session(s). Reviewers judge only against the acceptance checklist.
   - A reviewer writes only its `SESSION_DONE.md`. It follows the same close-out rule.
   - Done when: the review runs at the right tip with the registered reviewer model.

6. Decide
   - PASS: go to merge.
   - FAIL: check each blocker against the acceptance checklist.
     - Blocker matches an acceptance item: send a fix round to the same session.
     - Blocker matches nothing: note it on the ticket. Do not block.
   - After a fix round, reset the snapshot and review again.
   - If no result after 3 rounds (repo may change this): stop and decide with the user.
   - Done when: each review ends in PASS, a note, or a decision.

7. Merge and clean up
   - Merge with no-ff into main, from the repo main checkout.
   - The full test suite runs once at this gate.
   - Write the merge record (default: `docs/temp/merge-<date>-ticket-<nn>.md`).
   - Update the ticket status in its source, or comment/transition the remote issue, per that source's conventions.
   - Stop finished sessions. Remove their worktrees and branches. Check with `git branch --merged main` before deleting.
   - Done when: only active work remains.

## Other rules

- Get session ids from `agent_manager` list. Never from memory.
- A reviewer that writes any other file voids its review. Reset the snapshot and review again.
- Check evidence on the repo before merge.
- New worktrees: reset to baseline, run bootstrap.
- For long builds and tests: wait on log output. No logs for 120s (repo may change this): kill and report. No sleep polling.
- Worktree sessions do not talk to the user. Questions go through the dispatcher.
- Never put secret files in commits, logs, or reports.

## Reference files

- `references/session-protocol.md` — close-out contract: marker, message, discipline.
- `references/dispatch-prompts.md` — prompt templates with the close-out rule.
