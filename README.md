# browser-account-guard

A Claude skill that stops one specific, expensive mistake:

> **The browser being right does not mean the account is right.**

If you automate logged-in panels — Search Console, Google Ads, Analytics, Business
Profile/GMB, wp-admin, a domain registrar, Cloudflare, Meta Business — and you work across
more than one client, more than one browser, or more than one account, you will eventually
act in the wrong one. This skill makes that hard to do.

## Where it came from

Real work, not theory. Every rule here is something that failed first:

- A browser correctly identified as client A's opened Search Console under client B's
  account, because that was the browser's default. One click away from creating a client's
  property inside another client's account.
- The Google session index `/u/N/` jumped between `/u/0/` and `/u/3/` on consecutive
  navigations in the same browser, on the same day.
- Browser tabs died between almost every tool call, turning a five-step flow into twenty.
- A `role=combobox` in Search Console resisted coordinates, `.click()`, full mouse-event
  sequences and keyboard input — six attempts, all failing.

## What it covers

Nine steps, in order:

0. **Does this step need the logged-in session at all?** Public-page work belongs in a
   separate profile — least privilege, and also correctness (a timing measured in a
   logged-in profile is contaminated; a screenshot from one leaks PII into the deliverable).
1. **Establish the expected account and target first.** An assertion without an expected
   value is theatre. When the registry and the page disagree, the page wins.
2–3. **Which browser** — the confirmation protocol, marked as harness-specific so it can be
   skipped by anyone driving their own browser.
4. **Prove the account** against a source whose only subject is the signed-in identity —
   never by scanning the page for an e-mail, which passes falsely on any permissions screen.
5. **Prove the target.** Right account, wrong property is the expensive failure, and it looks
   like success.
6. **Force the account**, in order of preference: a profile per organisation, the account
   chooser, and `?authuser=` only as a last resort.
7. **Gate the irreversible action.** Identity is not authorisation, consent is per action,
   and the session can change under you — so re-prove immediately before acting.
8. **Record it** — with a schema, a verification date, and the rule that it never outranks
   the page.

Plus: working across several clients in one run, surviving tabs that die (batching, one
irreversible action per batch and it goes last, idempotency keys, never re-running a dead
batch), reactive panels that ignore programmatic input, and a table of **fragile UI path →
robust path** with the precondition each one needs.

An unattended run is **read-only**. The registry can say which account and which target; it
cannot supply consent for an irreversible action.

## How it was built

Two rounds of adversarial review by independent agents reading the document cold, each given
a realistic scenario and told to find holes rather than praise it. Round one scored it 6/10
and found twenty problems; round two scored the revision 8/10 on scenario handling and 6–6.5
on trigger design and adversarial review, finding a further set — including a false-positive
in the account check itself and several factual errors about platform behaviour. Both rounds
are fixed in what you are reading.

The method generalises: a cold reader evaluates what is *written*, while the author
evaluates what they *meant*.

## Install

Copy the folder into your skills directory:

```
~/.claude/skills/browser-account-guard/SKILL.md
```

Claude loads it automatically and invokes it when the task matches the description.

## Adapt it

Step 8 asks you to keep a registry mapping browser identifier → account → organization →
targets. Keep that file **outside** any repository you publish, and add its name to
`.gitignore` wherever it could be picked up — it holds personal data and a map of who owns
what.

```json
{
  "<browser-id>": {
    "nickname": "Studio",
    "account": "you@example.com",
    "organization": "Your company",
    "targets": ["sc-domain:example.com", "act_1234567890"],
    "verified": "2026-08-28"
  }
}
```

## License

MIT — see `LICENSE`.
