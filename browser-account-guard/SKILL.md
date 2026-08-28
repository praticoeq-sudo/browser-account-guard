---
name: browser-account-guard
description: Use before any browser action inside a client's or another organisation's logged-in panel — Search Console, Google Ads, Analytics, Business Profile/GMB, wp-admin, Cloudflare, domain registrar, Meta Business — whether reading private data or creating, verifying, publishing or deleting. Establishes which account and which property you expect, proves both on the page before acting, and gates the irreversible step. Also use when such a task is already failing — a panel that reports being logged out or shows another client's data, tabs that die between calls, clicks that never register, or deciding to drop the UI for an API. Not for public pages, page-speed runs or screenshots of public sites, and not for Cloudflare, Analytics, Gmail or any panel driven by API token, CLI or MCP with no browser involved — those need no account check.
---

# Right browser ≠ right account ≠ right target

Three failures this exists to prevent, all observed:

- **Wrong account.** Connected to the correct browser — client A's. Search Console loaded
  under client B's account, because that was the browser's default. One click from creating
  a client's property inside another client's account.
- **Right account, wrong target.** Correctly signed in, published to the wrong client's
  site. Identity was never the problem; the *destination* was.
- **False alarm.** Concluding *"the panel is logged out"* when it was the wrong browser, or
  the right browser on the wrong account. Hours debugging a problem that did not exist.

**One rule: never infer. Read it back — the account *and* the target — before you act.**

> **Reading is not safe either.** Steps 0–6 apply to reads. Pulling last month's clicks from
> the wrong property produces a confident, wrong report — and drops one client's data into
> another client's transcript. Prove the account before you *look*, not only before you
> *write*. (Step 7 is the exception: a read needs no consent gate.)

> **No human available?** (scheduled run, headless, non-interactive.) **An unattended run is
> read-only.** The registry from step 8 can answer step 1 — which account, which target — and
> can settle step 3 when it maps this client to one browser. It cannot answer step 7:
> consent for an irreversible action does not come out of a JSON file. If the job needs to
> create, publish, delete, send or pay, **stop and report**; leave a draft or a staged change
> and hand it to a human. Do not treat "the schedule was approved once" as approval for
> tonight's action.

> **Terms of service.** Driving a logged-in UI with synthetic events may breach a platform's
> terms, and the suspension lands on *the client's* account, not yours. This is a further
> reason the API path in the last section is the preferred one, not merely the sturdier one.

## The structural cure, if you can afford it

Everything below is mitigation. The actual fix is **one browser profile per organisation**,
each signed into exactly one account. Then "which browser" and "which account" collapse into
one question. Concretely: Chrome/Edge `--user-data-dir=<path-per-org>` (or a named profile),
Firefox Multi-Account Containers, or one browser binary per client.

Do that first if you control the environment. Use the rest when you cannot.

---

## Step 0 — Does *this* step need the logged-in session?

You are here because the task touches a client's panel. That does not mean every part of it
does. If a sub-task works on a **public page** — reading a site, measuring load time,
screenshots, checking markup — **do not use the user's logged-in browser.** Use a separate
profile or a headless browser: no account to confuse, no client data in scope, no session to
disturb.

This is least privilege, and it is also correctness: a timing measured in a logged-in
profile is contaminated (extensions, warm cache, service worker, personalisation), and a
screenshot from one leaks PII into the deliverable (bookmarks bar, avatar, autofill,
toasts) while showing a consent banner already dismissed — which was the whole point.

**Steps 1–8 then do not apply.** The mechanics sections at the end still do: tabs die and
frameworks swallow clicks on public pages too.

> Caveat: some embedded/preview browsers report `clientWidth: 0` and give useless layout
> measurements. If you need real geometry, drive a real headless browser.

## Step 1 — Establish the EXPECTED account and target, before touching anything

Every check below is an assertion. **An assertion without an expected value is theatre.**

1. **Read your registry first** (step 8). If it maps this client, you have the expected
   account — do not re-ask.
2. **Not in the registry? Ask the user now**, before opening anything:
   *"Which account owns this, and what exactly is the target — domain, site URL, ad
   account id, property?"*
3. **Never derive the account from the browser.** "It's the client's browser" is the
   assumption that starts the whole failure.

**When the registry and the page disagree, the page wins.** The registry is cached
inference; step 4 is measurement. Stop, tell the user what you found, and correct the
registry — never edit the page to match the file.

You may enumerate what is available before asking. Each needs credentials you already hold:

```
https://accounts.google.com/AccountChooser          accounts signed into this browser
GET https://<site>/wp-json/wp/v2/users/me?context=edit   WordPress identity + capabilities
     → application password or cookie+nonce; 401 here means "not authenticated",
       403 means "authenticated but not allowed" — do not conflate them
GET https://api.cloudflare.com/client/v4/user/tokens/verify   which token you hold
GET https://api.cloudflare.com/client/v4/accounts             which tenants it reaches
     → header: Authorization: Bearer <token>
     → /client/v4/user needs "User Details Read" and fails for an account-scoped
       token; for those use /client/v4/accounts/{id}/tokens/verify
```

## Steps 2–3 — Which browser

> These two steps assume a harness that **attaches to the user's running browsers** and can
> ask them to identify themselves. If you drive your own browser (Playwright, Puppeteer,
> raw CDP), there is nothing to choose — skip to step 4, which still applies in full.

**Step 2 — list the connected browsers.** Use whatever enumeration your harness exposes.
Labels are usually generic (`Browser 1`, `Browser 2`): they do **not** say whose browser it
is, do **not** reflect the nickname given at connect time, and do **not** mark which is
active. Do not guess from them.

**Step 3 — ask, when two or more are connected.** The platform generally *requires* human
confirmation; fighting it burns turns. Offer one option per browser (label = display name +
identifier), descriptions saying what you **know** about each — not what you assume — and a
final option that opens a confirmation inside every browser, so the user clicks *Connect* in
the right one and names it. That in-browser confirmation is the one that works: it returns
the real name. It may fail once with *"no browser responded"* — transient, retry once.

> With **one** browser connected you may select it without asking. That skips *this* step
> only. **Steps 4 and 5 still apply** — one browser can hold five accounts.

## Step 4 — Prove the account

🚨 **Do not scan the page for "any label containing an e-mail".** On exactly the pages this
skill targets — Search Console *Users and permissions*, Google Ads *Access and security*,
GA4 *Access management*, the WordPress *Users* list — the DOM is full of other people's
addresses. The expected address can appear there **as a collaborator while you are signed in
as somebody else**, and the check passes while you are in the wrong account. That is the
assertion-theatre step 1 warns about, inside the step that is supposed to be the proof.

**Primary source — a page whose only subject is the signed-in identity:**

```
https://myaccount.google.com/                → the signed-in address, rendered
https://accounts.google.com/AccountChooser   → every account in this browser
```

⚠️ `https://accounts.google.com/` alone does **not** list accounts — with one account signed
in it redirects to `myaccount`.

**Secondary — the account chip inside the panel, anchored, never scanned.** Read it on the
panel's own page, not on a chooser:

```js
const chip = document.querySelector('a[href*="SignOutOptions"], a[href*="accounts.google.com/SignOutOptions"]');
chip ? chip.getAttribute('aria-label') : null   // the label normally carries the address,
                                                // whatever the interface language
```

Match on the `@` *within that element* — never enumerate languages, that is a race you lose.

It must name the account you established in step 1. **If it does not, stop.**

**`null` is not "logged out".** It means *nothing matched* — the chip sits inside an iframe
(this selector does not cross frames), the page is a chooser, or the markup changed. Fall
back to the primary source above.

**If the expected account is not signed in anywhere:** stop and hand back to the user.
Do not type credentials. "Fix it" means re-navigating with the right `authuser`, switching
browsers, or asking the user to sign in — never authenticating on their behalf.

**Do not fall back to a screenshot for this check.** The avatar usually shows only an
initial, not an address; you would be reading a letter and calling it proof.

🔒 This read returns a real person's name and e-mail into your transcript. Compare it and
move on — do not echo it into logs, commits, issues or anything you publish.

## Step 5 — Prove the TARGET, not just the identity

Identity correct + destination wrong is the expensive failure, and it looks like success.
Before acting, read back **what you are pointed at**:

| Panel | Read the target this way |
|---|---|
| WordPress | `GET /wp-json` → `name`, `url`; and `users/me?context=edit` → `capabilities` |
| Cloudflare | `GET /client/v4/zones/{id}` → `name`, plus which account owns it |
| Google Ads | the **10-digit customer id** (`123-456-7890`), read from the account selector — not from the URL. ⚠️ `ocid` there is an internal obfuscated id, **not** the customer id and not exposed by any API: comparing it to the CID the user gave you mismatches every time. Under an MCC, login account ≠ operated account |
| Search Console | the `resource_id` in the URL — **URL-decode it first** (`sc-domain%3Aexample.com` → `sc-domain:example.com`), and note whether it is a `sc-domain:` (Domain) or `https://` (URL-prefix) property; they are different properties with different verification |
| Business Profile | the location id, not the business name (names repeat) |
| Meta Business | three different ids that are easy to swap: the **business id**, the ad account `act_<id>`, and the **Page id**. The one in the URL is often not the one you are about to act on |

Also confirm **permission**, not just presence:

- WordPress: a `subscriber` is correctly signed in and still cannot publish.
- Search Console: a **restricted** user cannot verify a property — which is the showcase
  flow of this document.
- Google Ads: read-only access looks identical until the mutate fails.
- GA4: account-level access ≠ property-level access.

## Step 6 — Force the account through the URL (Google products)

The session index `/u/N/` is **not stable**. In the same browser, on the same day, it can
jump between `/u/0/` and `/u/3/` on consecutive navigations. **Never hard-code `/u/1/`** and
never trust yesterday's index.

Prefer, in this order: **(1)** a profile that holds only that account (the structural cure);
**(2)** the account chooser, then verify; **(3)** `authuser` in the URL, as a last resort:

```
https://search.google.com/search-console?authuser=person%40example.com
```

`authuser` taking an address rather than a session index is **observed behaviour, not a
documented contract** — it works on the main Google panels, but do not build on it as a
guarantee.

⚠️ **`authuser` does not survive every redirect.** Panels that bounce you to a
`?resource_id=` or `?ocid=` URL frequently drop it. **Re-run step 4 after each navigation
that changes the URL.**

⚠️ **Putting an e-mail in a URL is itself a hazard** — it lands in browser history and in any
screenshot of the address bar, and some harnesses refuse to read the DOM of a URL carrying
personal data, which breaks step 4. Navigate once with `authuser`, let it redirect to a
clean URL, and run the check there. **If two such attempts both come back on the wrong
account, stop and switch to option (1) or (2)** — do not loop.

⚠️ `authuser` proves the **Google login**. Under an MCC or multi-tenant panel, the *account
being operated* is a separate thing — that is step 5.

## Step 7 — Gate the irreversible action

Creating, publishing, deleting, sending or paying **in someone else's account** deserves an
explicit confirmation, even when steps 4 and 5 passed. Identity is not authorisation.

- Show what you read back: **account, target, and the exact action**, in that order.
- **Consent is per action.** An OK for one property does not carry to the next one, to a
  second panel, or to a retry after a failure.
- **Re-prove immediately before acting, not only at the start.** It is the user's browser:
  they can switch account in another tab and change the session for the whole profile,
  retroactively invalidating your step 4. Long runs also hit session expiry and
  "verify it's you" interstitials.
- **Know the undo before you act** — Ads change history, WordPress revisions, and note where
  there is none: a deleted Search Console property must be verified again from scratch.
- Prefer a reversible shape when one exists: save as draft, stage it, dry-run it.

## Step 8 — Record what you learned

Keep a registry and **read it at step 1**. A write-only registry is why every session starts
from zero. Store it **outside any repository**, and add its filename to `.gitignore`
wherever it could be picked up:

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

Record the date. An entry nobody has re-proved in months is a hint, not evidence — the page
still wins (step 1).

> ⚠️ It holds personal data and a map of who owns what. **Never commit it to a public
> repository.** Treat it like a credential file.

## Working across several clients in one run

A loop over fifteen properties is the common shape, and every check above resets at each
iteration:

- **Re-prove account and target per iteration.** A proof carries to one target, not to the
  next. Crossing to a *different panel* is even further: run steps 1 and 4–7 again there.
- **Never carry a proven session forward as an assumption** because the previous item in the
  loop worked.
- **Keep client data separated in your own output.** The transcript is one place where two
  clients' data can end up side by side even when the panels never touched.

---

## Surviving tabs that die

Symptom: *tab no longer exists* **between one call and the next**, repeatedly.

1. **Batching is the default, not an optimization.** A tab survives *within* a batch and
   dies *between* batches. Chain navigate → wait → act → capture in one batch.
2. 🔑 **Inside a batch, aim by DOM selector — never by a stored element handle, never by
   coordinate.** A handle (`ref`, or whatever your tool calls the id returned by an earlier
   page read) comes from a call that is exactly what dies. A coordinate refers to the
   screenshot taken *before* the batch, and page scale shifts between loads. Only a selector
   resolved *at execution time* survives.
3. **Do not sleep blindly — poll.** There is usually no wait-for-selector. Resolve a
   **string or boolean**, never the element: a bridge that serialises the return value turns
   a DOM node into `{}`, and you lose the difference between found and timed out.
   ```js
   await new Promise(r => { const t0 = Date.now(); (function look(){
     const el = document.querySelector(SEL);
     if (el) return r(el.textContent.trim().slice(0, 80) || 'found');
     if (Date.now() - t0 > 8000) return r('TIMEOUT');
     setTimeout(look, 200); })(); });
   ```
4. **Recover** by requesting a tab context that creates one if empty (or creating a tab
   outright), then use the new id **in the very next batch**.
5. After switching browsers the previous tab group dies. Always recreate before acting.
6. A batch **stops at the first error** — put the screenshot last.
7. Waits are capped per action in some harnesses (10s is a common ceiling). To wait longer,
   chain two.
8. 🚨 **At most one irreversible action per batch, and it goes last.** The longer the batch,
   the more unreconstructable state when it dies. This is the same rule as "screenshot last",
   for the same reason.
9. 🚨 **Never blindly re-run a batch that died mid-way.** If a step submitted a form,
   re-running duplicates it. To make "did it happen?" answerable at all:
   - **Stamp a unique key before you submit** — a slug, a title, a marker you choose. Without
     one, "does this record already exist?" is undecidable whenever the name is plausible.
   - **Wait for the write to settle before reading.** A submit can still be in flight; read
     too early and you get "not created", re-submit, and duplicate — the very trap this rule
     exists to prevent. Re-read after a pause, and prefer the API or a list view over the UI
     you were driving.
   - **If it is still ambiguous, stop and ask.** There is no safe default here. Duplicating a
     charge is worse than waiting for a human.

## Panels that resist programmatic clicks

Search Console, Meta Business, Gutenberg and similar are reactive UIs.

- **`el.value = x` does not register.** React keeps the last value in an internal tracker on
  the node, sees no change, and drops the event. This is the real failure.
- **`element.click()` usually *does* work** — do not assume otherwise. React 17+ delegates
  listeners at the container root, and a real click event bubbles there and invokes
  `onClick`. What a synthetic event cannot do is anything gated on *transient user
  activation*, because it carries `isTrusted: false`: opening a native `<select>` popup,
  clipboard access, the file picker, `window.open`, fullscreen. When a control ignores
  `.click()`, suspect user activation or the wrong event interface — not "the framework
  ignores clicks".
- Use the **native setter plus events** for value changes:
  ```js
  const set = Object.getOwnPropertyDescriptor(Object.getPrototypeOf(el), 'value').set;
  el.focus();
  set.call(el, value);
  el.dispatchEvent(new Event('input',  { bubbles: true }));
  el.dispatchEvent(new Event('change', { bubbles: true }));
  ```
  ⚠️ This is for `<input>` and `<textarea>`. It does **not** work on `contenteditable` rich
  text (Gutenberg's post body, Notion-style editors) — there you need `document.execCommand`,
  a paste event, or the app's own API.
- **Clicking by coordinate is fragile**: page scale changes between loads and the click
  misses. Find the element and dispatch the full sequence — several menus listen only for
  `mousedown`. Use `PointerEvent` for `pointer*` (a `MouseEvent` has no `pointerId`,
  `isPrimary` or `pointerType`, and a real pointer handler ignores it):
  ```js
  const r = el.getBoundingClientRect();
  const base = { bubbles: true, cancelable: true, composed: true, view: window,
                 button: 0, buttons: 1,
                 clientX: r.left + r.width / 2, clientY: r.top + r.height / 2 };
  const ptr = { ...base, pointerId: 1, isPrimary: true, pointerType: 'mouse' };
  el.focus();
  el.dispatchEvent(new PointerEvent('pointerover', ptr));
  el.dispatchEvent(new MouseEvent('mouseover', base));
  el.dispatchEvent(new PointerEvent('pointerdown', ptr));
  el.dispatchEvent(new MouseEvent('mousedown', base));
  el.dispatchEvent(new PointerEvent('pointerup', ptr));
  el.dispatchEvent(new MouseEvent('mouseup', base));
  el.dispatchEvent(new MouseEvent('click', base));
  ```
  Getting the interface wrong is the most common cause of a `role=combobox` that "resists
  everything": libraries branch on `e.pointerType` or `e.isPrimary` and take the wrong path
  when those are absent. Note that `isTrusted: false` does *not* stop handlers from running —
  it only withholds user activation, which is a separate failure mode.
- Some `role=combobox` menus resist **everything** — coordinates, `.click()`, full event
  sequences and keyboard. **Two attempts per element, then change approach.**
- Repeated hammering on an account chooser can trip a platform's "unusual activity"
  interstitial — **on the client's account**. One more reason the budget is small.
- On some domains screenshots hang — read the DOM. On others the DOM read is blocked because
  the URL carries sensitive data — take a screenshot. But for *reading an account*, the
  screenshot is not a valid substitute (step 4): resolve that case by navigating to a clean
  URL instead.

## The better exit: stop depending on the UI

Every "robust path" below has a precondition: **do you actually hold that token/access?**
Check first — otherwise you are trading a slow path for a dead end.

| Goal | Fragile path (UI) | Robust path | Precondition |
|---|---|---|---|
| Verify a **URL-prefix** property in Search Console | DNS-provider menu | **HTML tag** in `<head>` | you control the site's HTML |
| Verify a **Domain** property | — | **TXT record via the DNS API** — DNS is the only method this property type accepts (sites on Blogger or Google Sites verify automatically). Note DNS is *not exclusive* to Domain properties: URL-prefix accepts it too | API token for that zone |
| Tell **Bing, Yandex, Naver, Seznam, Yep** about new pages | request indexing one by one | **IndexNow** API — participants are obliged to share submissions, so one endpoint reaches all; up to 10k URLs per request | a key file at the site root, *or* elsewhere on the same host declared via `keyLocation` — which then limits that key to URLs under that path |
| Read public social content | scroll the logged-in feed | the platform's **public API** | app credentials |
| Change DNS | registrar panel | the provider's **API** | zone token |
| Publish content | paste into the editor | the CMS **REST API** | app password / token |

⚠️ **Google is not an IndexNow participant.** It said in 2021 that it would test the protocol
and has never announced adoption; IndexNow appears nowhere in Search Central. If the goal was
Google, IndexNow is a different outcome wearing the same label — you still need the sitemap
and Search Console.

⚠️ **Reaching for the HTML tag on a Domain property wastes exactly the minutes the shortcut
promised to save.** Check the property type first (step 5); to use the tag you must add a
URL-prefix property instead.

⚠️ **Crossing systems does not inherit your checks.** That tag lives in the *client's* site —
a second panel, a second account, possibly a second organisation. **Run steps 1 and 4–7
again there.**

**Attempt budget: two per element, not two per page.** If three separate controls each fail
twice, that is a signal to abandon the whole UI, not a quota to spend down.

## Freshness

Roughly half of this describes **platform behaviour**, which rots: the account chip's markup
and its sign-out link, the wait ceiling, whether the harness forces browser confirmation, how
a panel redirects. Steps 2–3 additionally assume a harness that attaches to the user's
browsers.

**Behaviour last verified: August 2026** (Chrome + Claude Code on Windows, Google Search
Console, Google Ads, WordPress, Cloudflare).

The durable part is the doctrine — prove the account, prove the target, gate the irreversible
action, two attempts then change paths. Re-test the mechanics before trusting them in a much
later version.

## License

MIT.
