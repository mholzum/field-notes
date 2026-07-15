# Incident Postmortem — Concurrent Session Write Collision

*Multi-Agent Company OS · 2026-07-12 → 2026-07-14*

**Author:** Mark Holzum
**System:** Multi-agent Company OS — orchestrates content, ops, and sales across four business lines via named CLI agents reading a shared Obsidian vault
**Severity:** P1 — silent data loss to coordination state, twice in one week, still recurring at the time this was written
**Status:** HOLD in place. Fix designed, not yet fully shipped.

---

## TL;DR

Between July 8 and July 12, the vault's shared coordination files (`decisions/log.md`, `blocked.md`, `suggestions.md`, `questions.md`) were silently overwritten or truncated on two separate occasions. Root cause was concurrent CLI sessions writing to the same files without any coordination — the classic last-write-wins failure of using a filesystem as a message bus. The system had no locks, no CAS check, no append-only discipline, and no telemetry to detect a collision after it happened. On the morning of July 14, four separate sessions were running against the vault simultaneously with the incident still unresolved and the HOLD instruction still in the queue. A fifth was about to start. This is the writeup of what actually happened, what I got wrong, what I got right, and the design that replaces it.

## Timeline

**2026-07-08, evening.** First noticed corruption: `decisions/log.md` was missing three entries that had been written earlier that day. Blamed on a mispaste. Restored from git.

**2026-07-11, morning.** Second corruption event: `blocked.md` and `suggestions.md` both truncated to earlier versions. This time git reflog showed the writes had happened, then been overwritten within minutes by a second session that started with a stale in-memory copy of the file. No mispaste. This was concurrent write collision.

**2026-07-12, late night.** CEO-boot session found the pattern, wrote an incident entry into `queue/needs-mark.md` with the instruction: *HOLD — no new sessions in this vault until this entry is cleared*.

**2026-07-14, 09:53–10:49.** Four separate claude CLI processes started against the vault anyway — the HOLD lived only as a note in a file; nothing in the system prevented a new session from booting. A CTO-boot session began working through a seven-item punch list. Git status showed large uncommitted changes in flight across exactly the files that broke before.

**2026-07-14, midday.** Fifth session paused before dispatch. This document was written before continuing.

## What went wrong

Three failure modes stacked.

**First, coordination-as-file.** The vault was designed for a single human writer plus one assistant. It scaled to multiple parallel sessions by convention, not by contract. Each session read a file into memory, appended in memory, and wrote the whole file back. Two sessions that overlapped by even a few seconds would both write their own version last, and one would win by wall clock — silently.

**Second, HOLD-as-note.** The July 12 incident produced a HOLD entry in `queue/needs-mark.md` — but a HOLD written into a file that new sessions do not read on boot is not a HOLD. It is a hope. There was no gate, no boot-time check, no lockfile the CLI would trip over. The instruction was correct; the enforcement did not exist.

**Third, no incident telemetry.** When a file lost content, nothing in the system noticed. Detection relied on me eyeballing files and noticing entries were missing. This is how the second event went a full day before it was caught, and how any future events could pass silently while the system continued dispatching agent work off stale state.

## Root cause

The vault was serving three different jobs with one primitive: it was long-term memory (facts, decisions, history), it was a work queue (blocked, suggestions, needs-mark), and it was a coordination surface between concurrent processes. Long-term memory tolerates last-write-wins for a solo writer. A work queue for concurrent writers does not. I had built one thing where three were needed and hoped the difference would not matter until it did.

## What we shipped

Not a hotfix. The right fix is architectural, and I am writing it up before shipping the next line of code so future sessions inherit the design rather than the fix.

- **Single-writer discipline for shared files.** The four coordination files become append-only journals with one writer process — a lightweight queue daemon. All agent sessions send events through a socket; the daemon serializes writes and stamps a monotonic sequence number. Reads remain concurrent.
- **Advisory lockfiles at the session level.** New sessions cannot boot against the vault while a HOLD lockfile exists. The CLI wrapper checks `.vault/HOLD` before doing anything else and exits with a printed reason if present. HOLDs are files, not notes.
- **Boot-time integrity check.** On session start, the wrapper checksums the four coordination files against the last committed HEAD. A checksum mismatch on files that should only append is a loud, fast-failing warning — the session prompts before continuing.
- **Postmortem-first culture.** When a coordination file loses content, the next session's first job is a postmortem entry in `incidents/`, not a fix. I want the record before I want the patch.

## What we did not fix

Three things are known-broken and staying broken for now, because they cost more than they return this quarter:

- **The vault is still a single point of failure.** A proper multi-writer system would use a real queue (Redis, SQLite with WAL, or a hosted service). At current scale a lockfile + daemon is enough. When agent count crosses ~10 concurrent, this becomes wrong.
- **There is no dead-letter handling.** If the daemon rejects a write, the session that sent it currently retries without exponential backoff and without visibility. Fine for now, expensive at scale.
- **Agent constitutions are enforced by prompt, not by code.** An agent that violates its constitution — writes to files outside its scope, invokes tools it should not — is caught by the Overseer at review, not at emit. This is by design for now (fast to iterate on written rules) and wrong long-term.

## What this incident teaches about agent systems

The industry conversation about multi-agent systems is mostly about reasoning: how to make agents plan better, use tools better, verify outputs better. That is downstream. Upstream is that any system with more than one agent running against shared state is a distributed system, and distributed systems fail in the same way they have failed for forty years — races, silent overwrites, drift between what the process believes and what the substrate holds. The prompt engineering does not save you. The eval framework does not save you. Only a coordination model saves you, and it has to exist before the second process starts.

The reason I am writing this document rather than shipping the fix first is that the same lesson applies to how I run these systems: a repeated failure that is caught only by inspection is a system without alarms. Alarms come first. Fixes second. If the alarm is right and the fix is wrong, the next failure is cheap. If the fix is right and there is no alarm, the next failure is silent.

## Open questions I would want a colleague to press me on

- **Is the daemon a single point of failure worse than the collision it prevents?** I think no at current scale, but I have not proven it — a supervisor process and restart-on-crash policy is the next work if we ship this.
- **Does the HOLD lockfile create its own footgun** — a stale HOLD that blocks all work indefinitely because the writer forgot to clear it? Probably yes. Solution: HOLDs must include an owner and an expiry, and expired HOLDs raise a warning at boot rather than block.
- **Should the four coordination files be one file with typed entries instead?** A single append-only log with entry types (decision, block, suggestion, question) is arguably cleaner and removes cross-file consistency worries. Trade-off is human readability — I read these files with my eyes as often as agents do.

## Appendix — what the failing state looked like

At the point this document was written, `git status` against the vault showed:

```
modified:   company/BOARD.md
modified:   company/decisions/log.md
modified:   company/queue/blocked.md
modified:   company/queue/suggestions.md
modified:   company/queue/questions.md
modified:   company/queue/needs-mark.md
modified:   company/queue/completed.md
```

Four claude CLI processes had started that morning between 09:53 and 10:49, each holding an in-memory copy of some subset of those files, each about to write. This is what the system looks like the moment before the third incident. The HOLD in `needs-mark.md` was still the last entry, unread by any of them.
