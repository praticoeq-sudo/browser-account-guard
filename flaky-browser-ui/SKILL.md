---
name: flaky-browser-ui
description: Use when driving a web UI programmatically and it fights back — tabs that stop existing between calls, clicks or typed values that never register, React or Gutenberg inputs that ignore assignment, comboboxes that resist every event, or a batch that died mid-way after submitting something. Applies on any page, public or signed-in, in your own account or a client's, in an attached browser or in Playwright. Covers batching, aiming by DOM selector, polling instead of sleeping, native-setter input, full pointer sequences, the attempt budget, and when to abandon the UI for an API.
---

# When the UI will not obey

Automating someone else's web UI fails in three recurring ways: the **tab dies** between
calls, the **framework swallows** your input, or the **control is unreachable** by any
synthetic event. Each has a specific answer, and each has a point where the right move is to
stop and use an API.

> If the page is a signed-in panel belonging to a client or another organisation, run the
> companion skill **`browser-account-guard`** first. Making the UI obey in the wrong account
> is worse than not making it obey at all.

---

## Tabs that die between calls

Symptom: *tab no longer exists*, between one call and the next, repeatedly.

1. **Batching is the default, not an optimization.** A tab survives *within* a batch and dies
   *between* batches. Chain navigate → wait → act → capture in one batch.
2. 🔑 **Inside a batch, aim by DOM selector — never by a stored element handle, never by
   coordinate.** A handle comes from an earlier page read, and that earlier call is exactly
   what dies. A coordinate refers to the screenshot taken *before* the batch, and page scale
   shifts between loads. Only a selector resolved *at execution time* survives.
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
4. **Recover** by requesting a tab context that creates one if empty, then use the new id in
   the very next batch.
5. After switching browsers the previous tab group dies. Recreate before acting.
6. A batch **stops at the first error** — put the screenshot last.
7. Waits are capped per action in some harnesses (10s is a common ceiling). Chain two.

## A batch that died mid-way

🚨 **Never blindly re-run it.** If a step submitted a form, re-running duplicates it. A retry
loop around a creation call is how you end up with two properties, two posts, two charges.

- **At most one irreversible action per batch, and it goes last.** The longer the batch, the
  more unreconstructable state when it dies. Same rule as "screenshot last", same reason.
- **Stamp a unique key before you submit** — a slug, a title, a marker you choose. Without
  one, "does this record already exist?" is undecidable whenever the name is plausible.
- **Wait for the write to settle before reading.** A submit can still be in flight; read too
  early, get "not created", re-submit, duplicate — the exact trap this rule exists to
  prevent. Re-read after a pause, and prefer an API or a list view over the UI you were
  driving.
- **If it is still ambiguous, stop and ask.** There is no safe default. Duplicating a charge
  is worse than waiting for a human.

## Input the framework ignores

- **`el.value = x` does not register.** React keeps the last value in an internal tracker on
  the node, sees no change, and drops the event. Use the **native setter plus events**:
  ```js
  const set = Object.getOwnPropertyDescriptor(Object.getPrototypeOf(el), 'value').set;
  el.focus();
  set.call(el, value);
  el.dispatchEvent(new Event('input',  { bubbles: true }));
  el.dispatchEvent(new Event('change', { bubbles: true }));
  ```
  This is for `<input>` and `<textarea>`. It does **not** work on `contenteditable` rich text
  (Gutenberg's post body, Notion-style editors) — there you need `document.execCommand`, a
  paste event, or the app's own API.
- **`element.click()` usually *does* work** — do not assume otherwise. React 17+ delegates
  listeners at the container root, and a real click bubbles there and invokes `onClick`.
  What a synthetic event cannot do is anything gated on **transient user activation**,
  because it carries `isTrusted: false`: opening a native `<select>` popup, clipboard access,
  the file picker, `window.open`, fullscreen. If one of those is your goal, no event sequence
  will ever work — change approach now.

## Controls that resist every event

**Clicking by coordinate is fragile** — page scale changes between loads and the click
misses. Find the element and dispatch the full sequence, with the **right interfaces**: a
`MouseEvent` has no `pointerId`, `isPrimary` or `pointerType`, and libraries that branch on
those take the wrong path. This is the most common cause of a `role=combobox` that "resists
everything".

`button` and `buttons` are **not constant across the sequence** — a real browser reports
nothing pressed before the press and after the release, and `button: -1` on a move where no
button changed. Sending `buttons: 1` on every event is a tell that some libraries check.

```js
const r  = el.getBoundingClientRect();
const at = { bubbles: true, cancelable: true, composed: true, view: window,
             clientX: r.left + r.width / 2, clientY: r.top + r.height / 2 };
const hover = { ...at, button: -1, buttons: 0 };                // nothing pressed yet
const down  = { ...at, button:  0, buttons: 1, pressure: 0.5 }; // primary held
const up    = { ...at, button:  0, buttons: 0 };                // released
const ptr   = { pointerId: 1, isPrimary: true, pointerType: 'mouse' };
el.focus();
el.dispatchEvent(new PointerEvent('pointerover', { ...hover, ...ptr }));
el.dispatchEvent(new MouseEvent  ('mouseover',   hover));
el.dispatchEvent(new PointerEvent('pointermove', { ...hover, ...ptr }));
el.dispatchEvent(new MouseEvent  ('mousemove',   hover));
el.dispatchEvent(new PointerEvent('pointerdown', { ...down,  ...ptr }));
el.dispatchEvent(new MouseEvent  ('mousedown',   down));
el.dispatchEvent(new PointerEvent('pointerup',   { ...up,    ...ptr }));
el.dispatchEvent(new MouseEvent  ('mouseup',     up));
el.dispatchEvent(new MouseEvent  ('click',       { ...up, detail: 1 }));
```

The `pointermove` is not decoration — a real pointer always emits at least one before the
press, and `detail: 1` on the click matters because the constructor defaults it to 0.

`isTrusted: false` does **not** stop handlers from running — it only withholds user
activation, which is the separate failure mode above.

**Try the keyboard before giving up.** An ARIA `combobox` is required to respond to it, and
it often works where pointer events do not:

```js
el.focus();
const key = (k) => el.dispatchEvent(new KeyboardEvent('keydown',
  { key: k, code: k, bubbles: true, cancelable: true }));
key('ArrowDown');            // opens the listbox and moves to the first option
key('ArrowDown');            // navigate
key('Enter');                // commit
```

## The attempt budget

**Two attempts per element means two tries of the *same* technique.** Escalating through
coordinate → `.click()` → native setter → full pointer sequence → keyboard is **one pass**,
not five attempts — you have not spent the budget by climbing the ladder.

Repeating any one of them a third time is the waste. And if three separate controls each
exhaust their ladder, stop driving that UI at all: the page-level ceiling exists, and it is
reached by breadth of failure, not by counting clicks.

⚠️ Hammering a signed-in control — an account chooser above all — can trip a platform's
"unusual activity" interstitial, and on a client's panel that lands on **their** account.

## The better exit: stop depending on the UI

Every robust path has a precondition: **do you actually hold that token or access?** Check
first, or you trade a slow path for a dead end.

| Goal | Fragile path (UI) | Robust path | Precondition |
|---|---|---|---|
| Publish content | paste into the editor | the CMS **REST API** | application password / token |
| Change DNS | registrar panel | the provider's **API** | zone token |
| Read public social content | scroll the signed-in feed | the platform's **public API** | app credentials |
| Verify a site in Search Console | DNS-provider menu | **HTML tag** for a URL-prefix property; a **TXT or CNAME record via the DNS API** for a Domain property, which accepts no other method. Using DNS on a URL-prefix property auto-verifies the Domain one too | control of the HTML, or a zone token |
| Tell Bing, Yandex, Naver, Seznam or Yep about new pages | request indexing one by one | **IndexNow** — participants share submissions, so one endpoint reaches all; up to 10k URLs per request | a key file at the site root, or elsewhere on the same host declared via `keyLocation`, which then limits that key to URLs under that path |

⚠️ **Google is not an IndexNow participant.** It said in 2021 it would test the protocol and
has never announced adoption. If the goal was Google, IndexNow is a different outcome wearing
the same label — you still need the sitemap and Search Console.

⚠️ **Crossing into another system does not inherit your checks.** A verification tag lives in
the *client's* site — a second panel, possibly a second organisation. Re-run
`browser-account-guard` there.

⚠️ **Driving a signed-in UI with synthetic events may breach a platform's terms**, and the
suspension lands on the account you are driving. When both paths work, the API is the
compatible one, not merely the sturdier one.

## Freshness

Most of this is **platform and harness behaviour**, which rots — the wait ceiling, how tabs
are recovered, which internal a framework uses to track input. **Last verified August 2026**
(Chrome, React-based panels, Gutenberg, Search Console).

The durable part: batch, aim by selector, poll instead of sleeping, one irreversible action
per batch and it goes last, never re-run a dead batch, and two tries of the same technique
before changing approach.

## License

MIT.
