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

- The confirmation protocol when several browsers are connected — and why arguing with it
  wastes turns.
- **Proving** the account with a DOM read before acting, instead of inferring it.
- Forcing the right account through `?authuser=` instead of trusting `/u/N/`.
- Surviving tabs that die: batching as the default, and how to recover.
- Reactive panels that ignore programmatic clicks, and the native-setter recipe.
- A table of **fragile UI path → robust path**, which is the part that saves the most time.

## Install

Copy the folder into your skills directory:

```
~/.claude/skills/browser-account-guard/SKILL.md
```

Claude loads it automatically and invokes it when the task matches the description.

## Adapt it

Step 5 asks you to keep a registry mapping browser identifier → account → organization.
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
