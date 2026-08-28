# browser-account-guard + flaky-browser-ui

Two Claude skills for agents that drive web panels. They were one file until it grew past
400 lines and started loading a browser-automation cookbook into every task that only needed
to read a dashboard.

| Skill | Answers | Fires when |
|---|---|---|
| **`browser-account-guard`** | *Am I in the right account, pointed at the right thing?* | you are about to act in a signed-in panel where more than one account or property could be selected |
| **`flaky-browser-ui`** | *Why won't this UI obey?* | tabs die between calls, clicks or typed values do not register, a batch died mid-write |

They are independent. The second applies on public pages, in your own account, and under
Playwright — no account question involved. The first points at it when you need it.

## The mistake this exists to prevent

> **The browser being right does not mean the account is right.**

If you automate Search Console, Google Ads, Analytics, Business Profile, wp-admin, a domain
registrar, Cloudflare or Meta Business, and you work across more than one client, more than
one browser, or more than one account on the same machine — you will eventually act in the
wrong one. It rarely announces itself: the page loads, the data looks plausible, nothing
errors.

## Where it came from

Real work, not theory. Every rule is something that failed first:

- A browser correctly identified as client A's opened Search Console under client B's
  account, because that was the browser's default. One click from creating a client's
  property inside another client's account.
- The Google session index `/u/N/` moved between `/u/0/` and `/u/3/` on consecutive
  navigations in the same browser, on the same day.
- Tabs died between almost every call, turning a five-step flow into twenty.
- A `role=combobox` in Search Console absorbed six attempts across four techniques before
  anyone asked whether the UI was the right path at all. **That failure is where the
  two-attempt budget comes from** — it is the mistake, not a badge.

## How it was built, and where it still falls short

Three rounds of review by independent agents reading the files cold, each given a realistic
scenario and told to find holes rather than praise. Round 1 scored the original 6/10 and
found twenty problems. Round 2 scored the revision 8/10 on scenario handling, 6/10 on
trigger design, 6.5/10 adversarial. Round 3 drove the split you see here.

Known open points, stated rather than hidden:

- **Identity proof is strongest on Google.** WordPress, Cloudflare and Meta have one read
  each; a registrar has none worth trusting. The account-guard skill says so where it
  matters.
- **Re-proving costs navigations.** A fifteen-client loop pays for it. The skill offers a
  cheap in-loop check and an expensive one for writes, but the tension is real.
- **Steps 2–3 assume a harness that attaches to the user's running browsers.** They are
  marked skippable for anyone driving their own.

## Install

Each skill is one folder containing one file. Copy the folders you want into your skills
directory:

```
~/.claude/skills/browser-account-guard/SKILL.md
~/.claude/skills/flaky-browser-ui/SKILL.md
```

Or fetch a single file directly:

```bash
mkdir -p ~/.claude/skills/browser-account-guard && curl -sL https://raw.githubusercontent.com/praticoeq-sudo/browser-account-guard/main/browser-account-guard/SKILL.md -o ~/.claude/skills/browser-account-guard/SKILL.md
```

Claude loads them automatically and invokes each when the task matches its description.

## The registry

`browser-account-guard` step 8 keeps a registry mapping browser identifier → account →
organization → targets, at `~/.config/browser-account-guard/registry.json` by default.

**Keep it outside any repository.** It holds personal data and a map of who owns what. The
`.gitignore` here already covers the usual filenames, because the distance between "I copied
the folder from the repo" and "I saved my registry next to SKILL.md" is one `git add`.

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
