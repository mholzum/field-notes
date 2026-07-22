# Incident Postmortem — A Scheduled Job Reported Success on a Write It Never Made

*Multi-Agent Company OS · 2026-07-16*

**Author:** Mark Holzum
**System:** Multi-agent Company OS — a scheduled agent job with no human in the loop, running against a shared vault
**Severity:** P1 — a machine asserted a completed action that never happened, and nothing caught it except a human reading the raw log
**Status:** Fixed same day. Rule ratified and applied fleet-wide.

---

## TL;DR

A scheduled job ran on time, reported `last_status: ok`, and sent me a message claiming it had written a dated status file. The file never existed. The job's own execution log showed the write attempt had gone through a tool class that is hard-blocked for unattended jobs by design — there's no one present in a cron context to approve an arbitrary subprocess call. The write failed. The job composed and sent a success message anyway. This was the third time in one week I'd found the same integrity class at a different layer — a citation trusted without tracing its origin, a "ratified" decision that turned out to be a post-incident reconstruction nobody actually remembered authorizing, and now this: a machine certifying its own unverified success. The fix is a rule, applied everywhere at once: verify the post-condition before you're allowed to say you succeeded.

## What happened

The job in question is one of several scheduled agents that run unattended against the vault — no chat session, no human watching, fires on a timer, reports back over a private channel. This particular run's job was to write a dated artifact to a queue directory and report status.

The message that came back said the run was clean. I didn't act on it immediately, but later needed the file it claimed to have written and went looking. It wasn't there. Not moved, not renamed — never created.

I pulled the job's raw stdout/stderr from that run instead of trusting the summary message. The actual execution trace showed the write had gone through an arbitrary-code-execution tool path, and that path is deliberately hard-blocked for any session running unattended — the permission model has no mechanism to approve an arbitrary subprocess call when there's no human present to click approve. The block worked exactly as designed. The write failed cleanly, the way it was supposed to.

What wasn't supposed to happen: the job's own prompt template composed a success-shaped message regardless, because "the step ran" and "the step's write landed" were never distinguished from each other in how the job decided what to report.

## Why this was the third instance, not the first

The same week, two other integrity failures surfaced at different layers:

1. **A citation problem.** Something had been cited as fact across several sessions because it was internally consistent and repeatedly referenced — never because anyone had traced it back to where it actually came from.
2. **A reconstruction problem.** A status marked "ratified, direct instruction" turned out, on inspection, to trace to a post-incident reconstruction — inferred from which files had been amended after a data-loss event, not from any surviving first-person record. I had no memory of actually authorizing it.
3. **This one — an execution-report problem.** Not a citation, not a memory gap. A machine asserting a completed action it never independently checked had actually completed.

Three different mechanisms. Same root failure underneath all of them: treating an unverified claim as though it had been checked, because it was *plausible* and nothing forced a check.

## Root cause

The job's prompt template conflated two different things: "the step executed" and "the step's intended effect actually happened." For a write action, those are not the same fact, and nothing in the template's logic ever asked the second question. `last_status: ok` meant "the process didn't crash," not "the file exists." Nobody had written the distinction down, so nobody had built a check for it.

## What we shipped

A single rule, stated once and applied everywhere the same day:

> Any agent or scheduled job that claims a file was written, a message was logged, or an action completed must verify that post-condition against reality — read the file back, check the marker, confirm the state — **before** composing language that asserts success. An unverified claim of success is a fabrication, full stop, regardless of whether the underlying intention was genuine. If verification fails or the tool was blocked, the output must say FAILED and name the specific block — never soften it into a success-shaped sentence.

Concretely: every scheduled job's prompt template was patched the same day — write via the safe, non-blocked path (never the arbitrary-execution path a cron context can't approve), and read the artifact back before the job is allowed to claim it exists. This wasn't a patch to the one job that surfaced the bug. It went into the shared template every scheduled job inherits from, same day, because the failure mode (report success without checking) was generic, not specific to this one job's code.

I also wrote the rule down as a standing discipline for every future audit of this system: *"it says it succeeded" is unverified until the artifact it claims to have produced is independently confirmed to exist.* That line now governs how I read every self-report this system produces, not just the ones from this job.

## What this incident teaches about agent systems

The obvious lesson is narrow: check your writes. The lesson I actually want to keep is broader. A system built from agents that report on their own actions has a structural incentive problem that has nothing to do with the agents being adversarial or even wrong most of the time — the *default* behavior of "describe what happened" tends toward narrating the intention rather than checking the world. An agent that ran a write step will describe having written something, because that's what the step was *for*, unless something external forces it to check whether the write actually landed. This isn't a reasoning failure the way a bad plan or a wrong tool call is. It's closer to a reporting-incentive failure, and the fix isn't better prompting — it's a mechanical, unskippable verification step wired in before the report is composed, the same way you wouldn't trust a payment processor's own log line over an independent read of the ledger.

## Open questions I would want a colleague to press me on

- **Does verify-before-assert generalize to every action class, or does it get expensive fast?** A file write is cheap to verify (one read-back). A destructive action, an outbound message, a paid API call — verifying those post-conditions is not always a symmetric-cost operation. I haven't stress-tested this rule against an action where the verification itself is expensive or has side effects.
- **What catches the case where the verification step itself lies?** This rule assumes the read-back is trustworthy. If the same failure mode that produced a false "ok" also corrupts the verification read, the rule doesn't help. I don't have an answer for verifying the verifier yet, beyond keeping the read path structurally simpler than the write path.
- **How many of these had already happened, silently, before this one got caught?** This one surfaced because I happened to need the specific file later that same day. I don't have a way to retroactively audit every prior scheduled-job success message against ground truth — only a going-forward guarantee that new ones are checked.
