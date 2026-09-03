# Dispatch to Worktree

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

An agent skill for running a review-gated, ticket-to-merge pipeline over isolated git worktree sessions. A resident **dispatcher** session in kilocode Agent Manager is the single hub: it turns ready tickets into named worktree coding sessions, supervises them via completion briefs instead of polling, freezes every finished branch behind a read-only **review snapshot**, adjudicates review findings against the ticket's acceptance checklist, and merges to `main` only on verified evidence.

The process is distilled from a production workflow used on a UE5 multiplayer repo (single dispatcher supervising ~40 sequential worktree sessions through implement → verify → review → merge cycles).

## Installation

Install the folder into the host's skills directory, e.g.:

```bash
cp -r dispatch-to-worktrees ~/.agents/skills/
```

or, once the project is hosted on GitHub:

```bash
npx skills add Aknirex/dispatch-to-worktrees -y
```

## What It Does

- Turns ready tickets (from the repo's issue tracker) into one worktree session per ticket, with repo-conventional naming
- Replaces human/agent polling with a **completion-brief protocol**: each session writes a `SESSION_DONE.md` marker, then prompts the dispatcher with commit hash, test counts, and verdict
- Verifies evidence (commit on branch, clean worktree, real test output) before anything reaches a gate
- Dispatches **read-only review snapshots** at the coding branch tip; reviewers may write exactly one file
- Adjudicates every review blocker line-by-line against the ticket's acceptance checklist, downgrading off-checklist demands to notes instead of blocking merges
- Merges no-ff into `main`, writes a merge record, and cleans up finished worktrees and branches
- Enforces test layering (affected subset on fix rounds, full suite once at the gate) and log-mode process supervision with no blind sleep polling

## Dependencies and Boundaries

- **Session + worktree capability is required.** The host agent must (1) manage git worktrees and (2) spawn and supervise **independent agent sessions** — not just inline subagents. The whole protocol (dispatch, close-out, stop, re-dispatch) only makes sense for sessions with their own context and lifecycle.
- **Developed primarily for kilocode Agent Manager.** The skill assumes an `agent_manager`-style session tool and a resident AM dispatcher session. If the runtime lacks that capability, the skill stops and tells the user to run it inside Agent Manager.
- **No fixed default ticket source.** The agent finds ready tickets from whatever the repo actually uses — local ticket files, a remote tracker through its CLI (GitHub issues via `gh`, Jira CLI/API), or the user's request in context. When the source is not configured and unclear, it asks the user instead of guessing. Repo `AGENTS.md` conventions always win.
- **A process skill, not an implementer.** It does not split tickets itself and it does not rewrite a repo's conventions — ticket source, model registry, and test commands all come from the target repo's `AGENTS.md`.
- **Not for a single isolated task.** It exists to run concurrent ticket pipelines; a lone coding task does not need it.

## Good Input To Give The Agent

```text
Use dispatch-to-worktrees. The playtest follow-up phase has these ready tickets:
<list or path to the tracker>. Dispatch the unblocked ones, supervise to merge,
and clean up finished worktree sessions.
```

The dispatcher reads the repo's conventions itself; you mainly hand it the ready tickets and any boundary conditions (which branches must not be touched, which tickets are out of scope).

## Repository Structure

```text
.
|-- SKILL.md                       # the skill: pipeline steps + dispatcher iron rules
|-- README.md
|-- LICENSE
|-- .gitignore
|-- references/
|   |-- session-protocol.md        # SESSION_DONE marker + brief + read-only discipline
|   `-- dispatch-prompts.md        # fill-in prompt templates for worker/reviewer sessions
`-- skill/agents/
    `-- openai.yaml                # agent interface metadata
```

## Design Notes

- **Defaults, not dogma**: naming, marker paths, convergence caps and timeouts are defaults; the repo's `AGENTS.md` wins.
- **Evidence over narrative**: completion briefs carry hashes and counts, and the dispatcher re-checks them on the repo before any gate.
- **Read-only is a hard gate**: review sessions that touch anything but their marker void their own review.

## License

[MIT](LICENSE) © 2026 Aknirex
