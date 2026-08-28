---
name: browser-account-guard
description: Use before acting in a signed-in web panel where more than one account, tenant or property could be selected — Search Console, Google Ads, Analytics, Business Profile/GMB, wp-admin, Cloudflare, domain registrar, Meta Business — including read-only work, including scheduled or unattended runs, and including the user's own accounts on a machine that holds several. Establishes the expected account and target, proves both on the page, then gates any create, publish, delete or send. Also use when the panel claims you are logged out or shows the wrong organisation's data. Skip only when no signed-in session is opened at all — public-page work, or a task carried out end to end through an API token, CLI or MCP.
---

# Right browser ≠ right account ≠ right target

Three real failures, all near-misses: a browser correctly identified as client A's opened
Search Console under client B's account (the browser's default) — one click from creating a
property in the wrong account. A correct login published to the wrong client's site. And
hours spent debugging a panel that "was logged out" and was simply the wrong account.

**One rule: never infer. Read it back — the account *and* the target — before you act.**

> **Reading a panel is not safe either.** Steps 0–6 and 8 apply to reads. The wrong property
> produces a confident, wrong report and drops one client's data into another client's
> transcript. Only step 7, the consent gate, is exempt for reads.

> **Unattended run (scheduled, headless, no human)? It is read-only.** The registry (step 8)
> answers step 1 — which account, which target. It does **not** answer step 3: knowing which
> browser you want is not the same as being placed in it. And it cannot answer step 7 —
> consent does not come out of a JSON file. If the job must create, publish, delete, send or
> pay, **stop, write the reason to the job's own log or output, and leave a draft**. Never
> treat "the schedule was approved once" as approval for tonight's action.

> **Terms of service.** Driving a logged-in UI with synthetic events may breach a platform's
> terms, and the suspension lands on *the client's* account. Prefer the API. When both work,
> the API is not merely sturdier — it is the compatible one.

**The structural cure:** one browser profile per organisation, each signed into exactly one
account (`--user-data-dir=<path-per-org>` on Chrome/Edge, Firefox Multi-Account Containers,
or one browser binary per client). Then "which browser" and "which account" become one
question. Do this if you control the environment; everything below is mitigation.

For tabs that die, clicks that do not register and comboboxes that resist every event, see
the companion skill **`flaky-browser-ui`**. This skill is about *which account*; that one is
about *making the UI obey*.

---

## Step 0 — Does *this* step need the signed-in session?

A task that touches a client's panel usually also contains sub-tasks that do not: reading a
public page, measuring load time, screenshots, checking markup. Those go in a **separate
profile or a headless browser** — least privilege, and also correctness, because a timing
measured in a signed-in profile is contaminated and a screenshot from one leaks PII and
shows a consent banner already dismissed.

Steps 1–9 apply only to the signed-in part.

## Step 1 — Establish the EXPECTED account and target, before touching anything

Every check below is an assertion, and **an assertion without an expected value is theatre.**

1. **Read the registry** — default `~/.config/browser-account-guard/registry.json`. If the
   file does not exist, treat it as empty and go to 2. If it maps this client, you have the
   expected account; do not re-ask.
2. **Not there? Ask the user now**, before opening anything: *"Which account owns this, and
   what exactly is the target — domain, site URL, ad account id, property?"*
3. **Never derive the account from the browser.** "It's the client's browser" is the
   assumption that starts the whole failure.

**When the registry and the page disagree, the page wins.** The registry is cached
inference; step 4 is measurement. Correct the file, never the page.

🔒 Enumerating what is signed in (`accounts.google.com/AccountChooser`,
`GET /client/v4/accounts`) dumps **every** identity and tenant in that browser — personal
accounts and other clients included — into your transcript. Prefer asking. If you must
enumerate, compare and move on; do not echo the list into logs, commits or anything you
publish.

## Steps 2–3 — Which browser *(only if your harness attaches to the user's browsers)*

Skip both if you drive your own browser (Playwright, Puppeteer, CDP) — there is nothing to
choose. Otherwise: list them, and **do not trust the labels** (`Browser 1`, `Browser 2` do
not say whose they are, do not reflect the nickname given at connect time, and do not mark
which is active).

With two or more connected, the platform generally **requires** human confirmation — offer
one option per browser plus a final option that asks every browser to confirm, which is the
one that returns a real name. With **one** connected you may select it without asking; that
skips this step only, because one browser can hold five accounts.

## Step 4 — Prove the account

🚨 **Never scan the page for "any label containing an e-mail".** On exactly the pages this
skill targets — Search Console *Users and permissions*, Google Ads *Access and security*,
GA4 *Access management*, the WordPress *Users* list — the DOM is full of other people's
addresses, and the expected one can appear **as a collaborator while you are signed in as
somebody else**. The check passes and you act in the wrong account.

**Primary — a page whose only subject is the signed-in identity:**

```
https://myaccount.google.com/                → the signed-in address, rendered
https://accounts.google.com/AccountChooser   → every account in this browser
```

`https://accounts.google.com/` alone does **not** list accounts; with one account signed in
it redirects to `myaccount`.

**Secondary — the account chip, anchored, never scanned.** Read it on the panel's own page:

```js
const chip = document.querySelector('a[href*="SignOutOptions"]');
chip ? chip.getAttribute('aria-label') : null   // carries the address in most locales
```

Match the `@` *inside that element* — never enumerate languages. **`null` means nothing
matched**, not "logged out": the chip may sit in an iframe (this does not cross frames), the
page may be a chooser, or the markup may have changed. Fall back to the primary source.

**Outside Google**, identity has no shared shape. Known reads:

| Panel | Prove identity by |
|---|---|
| WordPress | `GET /wp-json/wp/v2/users/me?context=edit` → `slug`, `capabilities`. 401 = not authenticated; 403 = authenticated but not allowed — do not conflate |
| Cloudflare | `GET /client/v4/user/tokens/verify` for the token; the dashboard carries the account id in the URL path |
| Meta Business | the `business_id` in the URL — and see step 5, it is not the ad account |
| Registrar / other | read the account name in the page header, and treat it as weak evidence — prefer an API call that echoes the authenticated identity |

**If the expected account is signed in nowhere:** stop and hand back to the user. Do not type
credentials. Re-navigating, switching browsers or asking the user to sign in are fixes;
authenticating on their behalf is not.

**A screenshot is not proof here** — the avatar usually shows an initial, not an address.

## Step 5 — Prove the TARGET, not just the identity

Right identity + wrong destination is the expensive failure, and it looks like success.

| Panel | Read the target this way |
|---|---|
| Search Console | the `resource_id` in the URL, **URL-decoded** (`sc-domain%3Aexample.com` → `sc-domain:example.com`), and note whether it is a Domain (`sc-domain:`) or URL-prefix (`https://`) property — different properties, different verification |
| Google Ads | the **10-digit customer id** (`123-456-7890`) from the account selector, not the URL. `ocid` there is an internal obfuscated id, not the customer id, and never matches |
| Analytics / GA4 | the numeric **property id** (`properties/123456789`), not the account name — account access ≠ property access |
| WordPress | `GET /wp-json` → `name`, `url` |
| Cloudflare | `GET /client/v4/zones/{id}` → `name`, plus which account owns it |
| Business Profile | the location id, not the business name (names repeat) |
| Meta Business | business id, ad account `act_<id>` and Page id are three different things; the one in the URL is often not the one you are about to act on |
| Registrar | the domain exactly as registered, including the TLD — sibling domains differ by one character |

For a **metrics read**, the target includes the **date range**. A wrong period produces a
wrong report just as surely as a wrong property.

Confirm **permission**, not just presence: a WordPress `subscriber` is signed in and cannot
publish; a *restricted* Search Console user cannot verify a property; read-only Ads access
looks identical until the mutate fails.

## Step 6 — Force the account (Google products)

The session index `/u/N/` is **not stable** — it can move between navigations in the same
browser on the same day. Never hard-code it. In order of preference:

1. a profile holding only that account (the structural cure);
2. the account chooser, then verify;
3. `?authuser=person%40example.com`, last resort.

`authuser` taking an address rather than an index is **observed behaviour, not a documented
contract**. It does not survive every redirect — panels that bounce to `?resource_id=` or
`?ocid=` frequently drop it, so **re-run step 4 after any navigation that changes the URL**.
The address also lands in browser history, in screenshots of the address bar, and in your
harness's own request logs, so navigate once with it and let it redirect to a clean URL.
**If two attempts still land on the wrong account, stop and hand back to the user** — do not
loop, and do not fall back to options already unavailable.

`authuser` proves the **Google login**. Under an MCC or a multi-tenant panel, the account
being *operated* is a separate thing — that is step 5.

## Step 7 — Gate the irreversible action

Creating, publishing, deleting, sending or paying **in an account that is not yours** needs
an explicit OK even when steps 4 and 5 passed. Identity is not authorisation.

Show exactly this, then wait:

```
Account:  person@example.com        (proved via myaccount, 14:02)
Target:   sc-domain:client.com      (read from resource_id)
Action:   verify property via DNS TXT record
Undo:     none — a removed property must be verified again from scratch
Proceed?
```

- **Consent is per action.** An OK for one property does not carry to the next one, to a
  second panel, or to a retry after a failure.
- **Re-prove immediately before acting**, not only at the start. It is the user's browser:
  they can switch account in another tab and change the session for the whole profile,
  invalidating your step 4 retroactively. Long reads drift the same way — re-prove on
  elapsed time, not only per iteration.
- **Make the proof and the action atomic.** Put both in one batch, proof first: the
  batch-compatible check is the *anchored chip read*, because navigating away to
  `myaccount` breaks the sequence. Treat `null` there as **abort**, not as "carry on" — this
  is the one place where the secondary source outranks the primary, and where ambiguity must
  fail closed.
- **Know the undo before you act** — Ads change history, WordPress revisions, and note where
  there is none.
- Prefer a reversible shape: save as draft, stage it, dry-run it.

## Step 8 — Record what you learned

Write after step 4 or 5 succeeds — not only on task success, because the mapping you proved
is true regardless of what happened afterwards. Read it back at step 1.

Default path `~/.config/browser-account-guard/registry.json`:

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

An entry nobody has re-proved in months is a hint, not evidence — the page still wins.

> ⚠️ It holds personal data and a map of who owns what. **Never commit it.** Keep it outside
> any repository and add its name to `.gitignore` wherever it could be picked up.

## Step 9 — If you did act in the wrong account

Prevention fails eventually. When it does:

1. **Stop immediately.** Do not start a second action to undo the first.
2. **Do not clean up silently.** Tell the user which account was touched, what was written,
   and when — before you touch anything else.
3. **Establish the blast radius** from the change history (Ads change history, WP revisions,
   the panel's audit log), not from memory of what you intended.
4. **Note that the evidence may live inside the client's account**, which you may no longer
   be authorised to read. Say so rather than guessing.
5. Only then propose a remedy, and gate it through step 7 like any other action.

## Working across several clients in one run

A loop over fifteen properties is the common shape. **Re-prove account and target every
iteration** — a proof carries to one target, not the next — and never carry a session
forward because the previous item worked. Keep the clients' data separated in your own
output too. Crossing to a *different panel* is further still: run steps 1 and 4–7 again.

⚠️ Cost is real: fifteen properties with a full re-prove each is dozens of navigations, and
hammering an account chooser can trip an "unusual activity" interstitial **on the client's
account**. Use the cheap re-prove (the anchored chip read, no navigation) inside the loop,
and the expensive one (navigate to `myaccount`) only at the start and before writes.

---

## Worked example — verify a client's Domain property, start to finish

1. Registry has `client.com → ops@clientco.com`. Expected account and target established.
2. One browser connected → select it, no question. (Steps 2–3 done.)
3. Navigate `myaccount.google.com` → reads `ops@clientco.com`. Matches. (Step 4.)
4. Open Search Console; `resource_id` decodes to `sc-domain:client.com`. It is a **Domain**
   property, so the HTML tag is not an option — DNS only. (Step 5.)
5. Chip read confirms the account did not drift on the redirect. (Step 6.)
6. Show the block from step 7 — account, target, action, undo — and wait for the OK.
7. On OK: one batch, chip read first as an aborting guard, then write the TXT record through
   the DNS API. (Step 7, atomic.)
8. Append `verified` and the target to the registry. (Step 8.)

## Freshness

Half of this is **platform behaviour**, which rots — the chip's markup and its sign-out link,
how each panel redirects, whether your harness forces browser confirmation. **Last verified
August 2026** (Chrome, Search Console, Google Ads, WordPress, Cloudflare).

The doctrine is the durable part: prove the account, prove the target, gate the irreversible
action, and re-prove immediately before acting.

## License

MIT.
