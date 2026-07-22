# field-notes
## Field notes from operating a multi-agent company OS

Writeups from designing and running a multi-agent orchestration system
across four production lines. Postmortems get their own file.

### Incidents

- [2026-07-14 — Concurrent session write collision on shared coordination
  files](./2026-07-14-concurrency-incident.md)
- [2026-07-16 — A scheduled job reported success on a write it never
  made](./2026-07-16-fabricated-success-postmortem.md)

### Patterns

- Constitution files: how per-agent written constitutions actually get
  enforced (and where they don't). *coming soon*
- [Publish gates: the console-to-publish rail and what belongs
  in it](./2026-07-18-publish-gate-design.md)
- ~~Overseer review: catching fabricated outputs before ship.~~ see the
  2026-07-16 incident above.
