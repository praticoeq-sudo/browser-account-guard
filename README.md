# browser-account-guard

A Claude skill that stops one specific, expensive mistake:

> **The browser being right does not mean the account is right.**

If you automate logged-in panels — Search Console, Google Ads, Analytics, Business
Profile, wp-admin, a domain registrar, Cloudflare, Meta Business — and you work across
more than one client, more than one browser, or more than one Google account, you will
eventually create something in the wrong account. This skill makes that hard to do.

## Where it came from

Real work, not theory. Every rule here is something that failed first:

- A browser correctly identified as client A's opened Search Console under client B's
  account, because that was the browser's default. One click away from creating a client's
  property inside another client's account.
- The Google session index `/u/N/` jumped between `/u/0/` and `/u/3/` on consecutive
  navigations in the same browser, on the same day.
- Browser tabs died between almost every tool call, turning a five-step flow into twenty.
- A `role=combobox` in Search Console resisted coordinates, `.click()`, full mouse-event
  sequences and keyboard input — six attempts, all failing. An HTML meta tag solved the
  same goal in two minutes without touching the UI.

## What it covers

Nine steps, in order:

0. **Does this even need a logged-in browser?** Public-page work belongs in a clean
   profile — least privilege, and also correctness (a timing measured in a logged-in
   profile is contaminated; a screenshot from one leaks PII into the deliverable).
1. **Establish the expected account and target first.** An assertion without an expected
   value is theatre.
2–3. **Which browser** — the confirmation protocol, and why arguing with it wastes turns.
4. **Prove the account** with a DOM read, including what `null` actually means and why a
   screenshot is not valid evidence here.
5. **Prove the target.** Right account, wrong property is the expensive failure, and it
   looks like success.
6. **Force the account via `?authuser=`** instead of trusting `/u/N/`.
7. **Gate the irreversible action.** Identity is not authorisation.
8. **Record it** — and read the registry back at step 1.

Plus: surviving tabs that die (batching as the default, aim by DOM selector, never re-run a
batch that died mid-write), reactive panels that ignore programmatic clicks, and a table of
**fragile UI path → robust path** with the precondition each one needs.

## How it was tested

Four independent agents were given the skill cold, with realistic scenarios: a high-stakes
two-account setup, a single-browser case, a panel that appeared logged out, a dropdown that
would not respond, and an adversarial review.

They scored the first version **6/10** and found twenty problems — among them a factual
error in its own showcase example (the HTML-tag shortcut does not exist for Search Console
*Domain* properties, which only accept DNS verification), a claim that IndexNow tells
Google about new pages (it does not), a React recipe that fails on `contenteditable`, and a
dead locale selector. All twenty are fixed in what you are reading.

## Install

Copy the folder into your skills directory:

```
~/.claude/skills/browser-account-guard/SKILL.md
```

Claude loads it automatically and invokes it when the task matches the description.

## Adapt it

Step 8 asks you to keep a registry mapping browser identifier → account → organization.
Keep that file **outside** any repository you publish — it holds personal data and a map
of who owns what. A starting shape:

```json
{
  "<browser-id>": {
    "nickname": "Studio",
    "account": "you@example.com",
    "organization": "Your company"
  }
}
```

## License

MIT — see `LICENSE`.
