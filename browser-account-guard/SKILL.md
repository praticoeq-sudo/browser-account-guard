---
name: browser-account-guard
description: Prove the right account AND the right target before acting inside someone's logged-in panel — Search Console, Google Ads, Analytics, Business Profile, wp-admin, domain registrar, Cloudflare, Meta Business. Use when a task requires a signed-in session and you are about to read private data or create, verify, publish or delete something, especially with several browsers or several accounts in play. Also says when NOT to use a logged-in browser at all. Not needed for public pages, page-speed measurement or screenshots of public sites — those belong in a clean profile. Covers establishing the expected account, the browser confirmation protocol, forcing the account via URL, proving the target property, the consent gate before irreversible actions, surviving tabs that die between calls, and when to abandon the UI for an API.
---

# Right browser ≠ right account ≠ right target

## The mistakes this exists to prevent

> **Wrong account.** I connected to the correct browser — client A's. I opened Search
> Console. It loaded under client B's account, because that was the browser's default.
> One click from creating a client's property inside another client's account.

> **Right account, wrong target.** Correctly signed in as myself, I published to the wrong
> client's site. Identity was never the problem; the *destination* was.

> **False alarm.** Concluding *"the panel is logged out"* when it was the wrong browser, or
> the right browser on the wrong account. Hours of debugging a problem that did not exist.

**One rule: never infer. Read it back — the account *and* the target — before you act.**

---

## Step 0 — Does this need a logged-in browser at all?

If the task works on a **public page** — reading a site, measuring load time, screenshots,
checking markup — **do not use the user's logged-in browser.** Use an isolated or headless
one. That removes this entire class of risk: no account to confuse, no client data in
scope, no session to disturb. **Stop here; the rest of this skill does not apply.**

This is least privilege, and it is also correctness:

- **A measurement taken in a logged-in profile is invalid.** Extensions, a warm cache, a
  service worker, an ad blocker and personalisation all contaminate the timing.
- **A screenshot from a logged-in profile leaks PII into the deliverable** — bookmarks bar,
  account avatar, autofill, notification toasts.
- **A consent banner already dismissed in that profile** means the screenshot does not show
  what a new visitor sees, which was the whole point.

> Caveat: some embedded/preview browsers report `clientWidth: 0` and give useless layout
> measurements. If you need real geometry, drive a real headless browser.

## Step 1 — Establish the EXPECTED account and target, before touching anything

Every check below is an assertion. **An assertion without an expected value is theatre.**

1. **Read your registry first** (see step 7). If it already maps this client, you have the
   expected account — do not re-ask.
2. **Not in the registry? Ask the user now**, before opening anything:
   *"Which account owns this, and what exactly is the target — domain, site URL, ad
   account id, property?"*
3. **Never derive the account from the browser.** "It's the client's browser" is the
   assumption that starts the whole failure.

You may also enumerate what is available before asking:

```
https://accounts.google.com/            → accounts signed into this browser
GET /wp-json/wp/v2/users/me?context=edit → WordPress identity + capabilities
GET /client/v4/user  and  /client/v4/accounts → Cloudflare identity + tenants
```

## Step 2 — List the connected browsers

Labels are usually generic (`Browser 1`, `Browser 2`). They do **not** say whose browser it
is, do **not** reflect the nickname given at connect time, and do **not** mark which is
active. Do not guess from them.

## Step 3 — Ask which browser (only when two or more are connected)

With two or more, the platform generally **requires** human confirmation. Fighting it burns
turns. Build the question with one option per browser (label = display name + identifier),
descriptions saying what you **know** about each — not what you assume — and a final option
that opens a confirmation inside every browser so the user clicks *Connect* in the right one
and names it.

In practice the **in-browser confirmation is the one that works**: it returns the real name.
It may fail once with *"no browser responded"* — transient. Retry once before concluding.

> With **one** browser connected you may select it without asking. That skips *this* step
> only. **Steps 4 and 5 still apply** — one browser can hold five accounts.

## Step 4 — Prove the account

```js
document.querySelector('[aria-label*="Google Account"],[aria-label*="Conta del"],'
  + '[aria-label*="Conta do Google"],[aria-label*="Compte Google"]')
  ?.getAttribute('aria-label')
```

It must name the account you established in step 1. **If it does not, stop.**

**`null` is not "logged out".** It means *the selector did not match*, which happens when:
the UI is in a language you did not cover, the account chip sits inside an iframe, or you
are on an account-chooser screen. Disambiguate with a source that does not depend on the
panel's markup:

```
https://myaccount.google.com/   → read the e-mail directly
https://accounts.google.com/    → list every account in this browser
```

**If the expected account is not signed in anywhere:** stop and hand back to the user.
Do not type credentials. "Fix it" means re-navigating with the right `authuser`, switching
browsers, or asking the user to sign in — never authenticating on their behalf.

**Do not fall back to a screenshot for this particular check.** The avatar usually shows
only an initial, not an address; you would be reading a letter and calling it proof.

🔒 This read returns a real person's name and e-mail into your transcript. Compare it and
move on — do not echo it into logs, commits, issues or anything you publish.

## Step 5 — Prove the TARGET, not just the identity

Identity correct + destination wrong is the expensive failure, and it looks like success.
Before acting, read back **what you are pointed at**:

| Panel | Read the target this way |
|---|---|
| WordPress | `GET /wp-json` → `name`, `url`; and `users/me?context=edit` → `capabilities` |
| Cloudflare | `GET /client/v4/zones/{id}` → `name`, plus which account owns it |
| Google Ads | the customer id in the URL (`ocid`/`customerId`) — under an MCC the login and the account are different things |
| Search Console | the `resource_id` in the URL matches the intended property |
| Business Profile | the location id, not the business name (names repeat) |

Also confirm **permission**, not just presence: a WordPress `subscriber` can be correctly
signed in and still fail to publish, after you already reported "account confirmed".

## Step 6 — Force the account through the URL (Google products)

The session index `/u/N/` is **not stable**. In the same browser, on the same day, it can
jump between `/u/0/` and `/u/3/` on consecutive navigations. **Never hard-code `/u/1/`** and
never trust yesterday's index.

Use the **`authuser`** parameter with the e-mail:

```
https://search.google.com/search-console?authuser=person%40example.com
https://analytics.google.com/analytics/web/?authuser=person%40example.com
```

⚠️ **`authuser` does not survive every redirect.** Panels that bounce you to a
`?resource_id=` or `?ocid=` URL frequently drop it. **Re-run step 4 after each navigation
that changes the URL**, not only at the start.

⚠️ `authuser` proves the **Google login**. Under an MCC or a multi-tenant panel, the
*account being operated* is a separate thing — that is step 5.

## Step 7 — Gate the irreversible action

Creating, publishing, deleting, sending or paying **in someone else's account** deserves an
explicit confirmation, even when steps 4 and 5 passed. Show the user what you read back —
account, target, and the exact action — and get an OK. Identity is not authorisation.

Prefer a reversible shape when one exists: save as draft, stage it, dry-run it.

## Step 8 — Record what you learned

Keep a registry mapping `browser identifier → nickname → account → organization → targets`,
and **read it at step 1**. A write-only registry is why every session starts from zero.

> ⚠️ It holds personal data and a map of who owns what. **Never commit it to a public
> repository.** Treat it like a credential file.

---

## Surviving tabs that die

Symptom: *tab no longer exists* **between one call and the next**, repeatedly.

1. **Batching is the default, not an optimization.** A tab survives *within* a batch and
   dies *between* batches. Chain navigate → wait → act → capture in one batch.
2. 🔑 **Inside a batch, aim by DOM selector — never by `ref`, never by coordinate.**
   This is the sentence that makes batching actually work. A `ref` comes from an earlier
   `read_page`, and that earlier call is exactly what dies. A coordinate refers to the
   screenshot taken *before* the batch, and page scale shifts between loads. Only a
   selector resolved *at execution time* survives.
3. **Do not sleep blindly — poll.** There is usually no wait-for-selector. Run a small JS
   loop that returns as soon as the element exists, and report what it found:
   ```js
   await new Promise(r => { const t0 = Date.now(); (function look(){
     const el = document.querySelector(SEL);
     if (el || Date.now() - t0 > 8000) return r(el);
     setTimeout(look, 200); })(); });
   ```
4. **Recover** by requesting a tab context that creates one if empty (or creating a tab
   outright — not every tool exposes a "create if empty" flag), then use the new id
   **in the very next batch**.
5. After switching browsers the previous tab group dies. Always recreate before acting.
6. A batch **stops at the first error** — put the screenshot last.
7. Waits are capped per action in some harnesses (10s is a common ceiling). To wait longer,
   chain two.
8. 🚨 **Never blindly re-run a batch that died mid-way.** If step 3 of 7 submitted a form,
   re-running duplicates it. **Read the state first** — does the record already exist? —
   and resume from there. A retry loop around a creation call is how you end up with two
   properties, two posts, two charges.

## Panels that resist programmatic clicks

Search Console, Meta Business, Gutenberg and similar are reactive UIs.

- `element.click()` and `el.value = x` **do not register** — the framework ignores them.
- Use the **native setter plus events**:
  ```js
  const set = Object.getOwnPropertyDescriptor(Object.getPrototypeOf(el), 'value').set;
  el.focus();
  set.call(el, value);
  el.dispatchEvent(new Event('input',  { bubbles: true }));
  el.dispatchEvent(new Event('change', { bubbles: true }));
  ```
  ⚠️ `_valueTracker` is React's internal for `<input>` and `<textarea>` **only**. It does
  not exist on `contenteditable` rich text (Gutenberg's post body, Notion-style editors) —
  there you need `document.execCommand`, a paste event, or the app's own API.
- **Clicking by coordinate is fragile**: page scale changes between loads and the click
  misses. Find the element in the DOM and dispatch the full sequence — several menus listen
  only for `mousedown` with the button fields populated:
  ```js
  const r = el.getBoundingClientRect();
  const base = { bubbles: true, cancelable: true, composed: true, button: 0, buttons: 1,
                 clientX: r.left + r.width / 2, clientY: r.top + r.height / 2 };
  ['pointerover','mouseover','pointerdown','mousedown','pointerup','mouseup','click']
    .forEach(t => el.dispatchEvent(new MouseEvent(t, base)));
  ```
- Some `role=combobox` menus resist **everything** — coordinates, `.click()`, full event
  sequences and keyboard. **Two attempts per element, then change approach.**
- On some domains screenshots hang — read the DOM. On others the DOM read is blocked
  because the URL carries sensitive data — take a screenshot. Two different paths, but see
  the warning in step 4: for *reading an account*, the screenshot is not a valid substitute.

## The better exit: stop depending on the UI

Every "robust path" below has a precondition: **do you actually hold that token/access?**
Check first — otherwise you are trading a slow path for a dead end.

| Goal | Fragile path (UI) | Robust path | Precondition |
|---|---|---|---|
| Verify a site in Search Console | DNS-provider menu | **HTML tag** in `<head>` — but see the trap below | you control the site's HTML |
| Same, for a Domain property | — | **TXT record via the DNS API** | API token for that zone |
| Tell **Bing/Yandex/Seznam** about new pages | request indexing one by one | **IndexNow** API — no quota | a key file at the site root |
| Read public social content | scroll the logged-in feed | the platform's **public API** | app credentials |
| Change DNS | registrar panel | the provider's **API** | zone token |
| Publish content | paste into the editor | the CMS **REST API** | app password / token |

⚠️ **IndexNow does not reach Google.** Google ignores it. If the goal was Google, IndexNow
is a different outcome wearing the same label — you still need the sitemap and Search Console.

⚠️ **The HTML-tag shortcut does not exist for every property type.** In Search Console the
DNS-provider dropdown only appears in the **Domain property** flow, and a Domain property
accepts **DNS verification only** — no HTML tag. To use the tag you must add a
**URL-prefix** property instead. Reaching for the tag without switching property type wastes
exactly the minutes the shortcut promised to save.

⚠️ **Crossing systems does not inherit your checks.** That tag lives in the *client's* site —
a second panel, a second account, possibly a second organisation. **Run steps 1 and 4–7
again there.**

**Attempt budget: two per element, not two per page.** If three separate controls each fail
twice, that is six attempts and a signal to abandon the whole UI, not to keep going.

---

## Anti-patterns

- Opening a logged-in browser for a task that only needs a public page.
- Running the account check without having established what account you expect.
- Picking a browser yourself when more than one is connected.
- Trusting the generic `Browser 1` label to identify whose browser it is.
- Reading `null` from the account selector and calling it "logged out".
- Hard-coding `/u/1/` or any session index in a URL.
- Checking the account once and assuming it survived every later redirect.
- Proving the identity and never proving the target.
- Firing an irreversible action because identity checked out.
- One tool call per browser action when tabs are dying.
- Repeating the same failing click a third time instead of changing strategy.
- Committing the browser/account registry to a public repository.

## Freshness

Roughly half of this describes **platform behaviour**, which rots: the account `aria-label`,
the wait ceiling, whether the harness forces browser confirmation, how a panel redirects.

**Behaviour last verified: August 2026** (Chrome + Claude Code on Windows, Google Search
Console, Google Ads, WordPress, Cloudflare).

The durable part is the doctrine — prove the account, prove the target, gate the
irreversible action, two attempts then change paths. Re-test the mechanics before trusting
them in a much later version.

## License

MIT.
