# Design Note — The Publish Gate: Why a Write Call's Exit Code Is Never the Proof

*Multi-Agent Company OS · 2026-07-18/19*

**Author:** Mark Holzum
**System:** Multi-agent Company OS — the console-to-publish rail that takes an approved piece of content from private/draft to public on a real platform account
**Status:** Live. One standing watcher service, one independently-verified live publish as proof it actually works.

---

## The problem this design answers

This system's content pipeline ends in exactly one human-approval step: I reply `APPROVE <id>` on a private channel, and a standing service is supposed to take that piece from private to public on the named account, unattended, from that point on. That's the easy part to build. The hard part is a question I didn't take seriously enough the first time I built something like this: **when the publish-write call returns success, is that success?**

It isn't, necessarily. A publish-write call can return exit 0 for reasons that have nothing to do with "the content is now live" — a wrong-channel credential that the API silently no-ops on instead of erroring, a race with another process touching the same account, a partial state change that looks committed but isn't. An exit code tells you the process didn't crash. It does not tell you the world changed the way you wanted.

## What the watcher actually does

The service (`approve-watcher`) polls my message history on a fixed interval for the approval pattern, matched against a specific chat ID so it can't be triggered by anyone else. On a match, it doesn't just fire the write and report the write's own return value. It does this instead:

1. **Tries each of several channel credentials in turn**, against a write function that's built to refuse cleanly on a wrong-channel guess *before* writing anything — safe by construction, so a mismatched attempt fails loud instead of silently doing nothing.
2. **After the write call returns success, makes a separate, independent read call back against the platform's own API** — not trusting its own write, asking the platform from scratch what the real current state of that item is.
3. **Only reports success to me if that independent read confirms the expected public state.** A write that returned "success" but whose follow-up read doesn't confirm it is logged as its own distinct failure class — separate from "the API call itself errored" — and alerts loudly instead of retrying silently.

The design also does something I think is easy to skip: it knows where its own authority ends, and stops there even when it technically could go further. The same service detects the approval for a second, closely related publish path — an image post staged in an already-open browser tab — but it does not click the final submit button, even though clicking a specific DOM element is mechanically no harder than the API write it does perform for the first path. That action was explicitly ruled human-only elsewhere in this system's standing policy, because browser-driven posting to a live consumer account carries a different risk profile than an idempotent, independently-verifiable API call, regardless of how similar the two look from the outside. The watcher acknowledges the approval and stops — a later, human-driven session finishes that one click.

## Why the independent read matters more than it sounds like it should

This is the same lesson as [the fabricated-success postmortem](./2026-07-16-fabricated-success-postmortem.md), applied one layer earlier — as a design constraint on the service itself, not a rule enforced after the fact by someone auditing logs. That postmortem's rule says "verify the post-condition before you assert success." This is what "verify" concretely means when the action is a live, third-party API state change: don't trust the call's own return value, ask an independent source of truth — the same one a human would check by hand — whether the state you wanted actually landed.

It's a cheap thing to build. One extra read call. And it closes an entire class of silent-failure bugs that no amount of careful prompting on the write call itself would catch, because the write call's own description of what happened is exactly the thing you can't trust.

## Proof it's real, not a demo

I don't think a design note like this means anything without a receipt, and I don't think the receipt should be the system's own claim that it worked. So: here's an independent check, run fresh, against a real published piece from this pipeline.

```
$ curl -s -o /dev/null -w "%{http_code}\n" \
  "https://www.youtube.com/oembed?url=https://youtu.be/Vqs6TH6ptQ0&format=json"
200
```

HTTP 200 on YouTube's own oEmbed endpoint, checked live, confirms that video is real, public, and live right now. It's one episode of a faceless explainer-format channel this pipeline produces for — scripted, produced, run through the pipeline's quality gates, published only after my own review-console approval. It's not a claim about scale or virality; it's a narrow, checkable claim: the full chain from approved piece to live public asset actually reaches a real platform account and stays there.

The asymmetry is the point, not the video itself. This same pipeline has, on at least one other occasion, self-reported a successful public publish for a different piece that — checked the same way, independently, later — returned a 404 instead of a 200. That claim did not hold up. I left that anomaly on record rather than quietly re-checking it away, because a verification discipline that only gets applied to the claims that turn out true isn't a verification discipline.

## What I'd want a colleague to press me on

- **Does "never trust your own write" scale to actions where the independent read is expensive or rate-limited?** This works cleanly for a YouTube API call. I haven't had to design this for a write where the confirming read costs real money or hits a tight rate limit.
- **What's the actual boundary between "technically capable" and "authorized"?** The watcher stops short of the browser click because a policy said so, not because it's technically incapable. That boundary lived in a document, not in code that made the click physically impossible. I don't have a strong answer yet for how to make an authority boundary as hard to cross as a permission boundary, short of literally not building the capability at all.
