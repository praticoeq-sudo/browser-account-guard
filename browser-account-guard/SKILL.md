---
name: browser-account-guard
description: Pick the right browser AND prove the right account before automating any logged-in panel — Search Console, Google Ads, Analytics, Business Profile, wp-admin, domain registrar, Cloudflare, Meta Business. Use whenever more than one browser is connected, more than one account exists in the same browser, or you are about to create, verify, publish or delete something in someone's account. Covers the confirmation protocol, forcing the account via URL, surviving tabs that die between calls, and when to abandon the UI for an API.
---

# The browser being right does not mean the account is right

## The mistake this exists to prevent

> I connected to the correct browser — client A's. I opened Search Console.
> It loaded under client B's account, because that was the browser's default.
> Had I created the property there, one client's site would have been born
> inside another client's account.

The inverse is more common and wastes more time: concluding *"the panel is logged out"*
when it was only the wrong browser, or the right browser on the wrong account.

**One rule fixes both: never infer the account — prove it before you act.**

---

## Procedure

### 1. List the connected browsers

Labels are usually generic (`Browser 1`, `Browser 2`). They do **not** tell you whose
browser it is, do **not** reflect the nickname given at connect time, and do **not** mark
which one is active. Do not guess from them.

### 2. Ask the user — this is not optional

With two or more browsers connected, the platform **requires** human confirmation before
any action. Fighting that only burns turns. Build the question with:

- one option per browser (label = display name + identifier in parentheses);
- in each description, what you **know** about that identifier — not what you assume;
- and a final option that opens a confirmation inside every browser, so the user clicks
  *Connect* in the one they want and names it.

In practice **the in-browser confirmation is the one that works**: it returns the real
name. It may fail once with *"no browser responded"* — that is transient. Retry once
before concluding anything.

### 3. Prove the account — the step almost everyone skips

Before you create, verify, save or delete anything:

```js
document.querySelector('[aria-label*="Google Account"],[aria-label*="Conta do Google"]')
        ?.getAttribute('aria-label')
```

It must name the account you expect. **If it does not, stop and fix it — do not continue.**

For non-Google panels, find the equivalent: the profile menu, the header e-mail, a
`/api/me` call — whatever exists. What matters is that it is a **read**, not an assumption.

### 4. Force the account through the URL (Google products)

The session index `/u/N/` is **not stable**. In the same browser, on the same day, it can
jump between `/u/0/` and `/u/3/` on every navigation. **Never hard-code `/u/1/`** and never
trust yesterday's index.

What always works is the **`authuser`** parameter with the e-mail:

```
https://search.google.com/search-console?authuser=person%40example.com
https://analytics.google.com/analytics/web/?authuser=person%40example.com
https://business.google.com/dashboard?authuser=person%40example.com
```

Google redirects to the correct index for that browser on its own.

### 5. Record what you learned

Keep a registry — outside your code repository — mapping
`browser identifier → nickname → account → organization`. Without it, every session starts
from zero and asking the user becomes a ritual instead of an informed decision.

> ⚠️ That file holds personal data and an account map. **Never commit it to a public
> repository.** Treat it like a credential file.

---

## Surviving tabs that die

Symptom: *tab no longer exists* **between one call and the next**, repeatedly.

1. **Batching is the default, not an optimization.** A tab usually survives *within* a
   batch and dies *between* batches. Chain navigate → wait → act → capture in one batch.
2. **Recover** by requesting the tab context with "create if empty", then use the new id
   **in the very next batch**.
3. After switching browsers, the previous tab group dies. Always recreate before acting.
4. A batch typically **stops at the first error** — put the screenshot last.
5. Waits are usually capped per action (10s is common). To wait longer, chain two.

---

## Panels that resist programmatic clicks

Search Console, Meta Business, Gutenberg and similar are reactive UIs. In them:

- `element.click()` and `el.value = x` **do not register** — the framework ignores the change.
- Use the **native setter plus events**:
  ```js
  const set = Object.getOwnPropertyDescriptor(Object.getPrototypeOf(el), 'value').set;
  el.focus();
  set.call(el, value);
  el.dispatchEvent(new Event('input',  { bubbles: true }));
  el.dispatchEvent(new Event('change', { bubbles: true }));
  ```
  In editor fields such as Gutenberg, clear `el._valueTracker.setValue('')` first.
- **Clicking by coordinate is fragile**: page scale changes between loads and the click
  misses. Find the element in the DOM and dispatch the full sequence
  (`pointerdown → mousedown → pointerup → mouseup → click`).
- Some `role=combobox` menus resist **everything** — coordinates, `.click()`, event
  sequences and keyboard. **Do not try more than twice.**
- On some domains screenshots hang; read the DOM instead. On others the DOM read comes back
  blocked because the URL carries sensitive data; take a screenshot instead. When one path
  fails, the other usually works.

---

## The better exit: stop depending on the UI

Before fighting a panel, ask **what is already under my control**:

| Goal | Fragile path (UI) | Robust path |
|---|---|---|
| Verify a site in Search Console | DNS-provider menu | **HTML tag** in `<head>` (you own the site) or **TXT record via the DNS API** |
| Tell search engines about new pages | request indexing one by one, daily quota | **IndexNow** API — Bing, Yandex, Seznam, no quota |
| Read public social content | scroll the logged-in feed | the platform's **public API** |
| Change DNS | registrar panel | the provider's **API**, when you hold a token |
| Publish content | paste into the editor | the CMS **REST API** |

**Two attempts in the UI. Still failing? Change paths.**

A domain verification that had failed six times in a dropdown was solved in two minutes by
an HTML tag — which needed no interface at all.

---

## Before saying "it is not logged in"

Prove it. Open the service's account page and read the identifier, or run the check from
step 3. Declaring "logged out" without verifying has cost hours of debugging a problem that
did not exist.

---

## Anti-patterns

- Picking a browser yourself when more than one is connected.
- Trusting the generic `Browser 1` label to identify whose browser it is.
- Hard-coding `/u/1/` or any session index in a URL.
- Acting on an account you have not read back.
- One tool call per browser action when tabs are dying.
- Repeating the same failing click a third time instead of changing strategy.
- Committing the browser/account registry to a public repository.

## License

MIT.
