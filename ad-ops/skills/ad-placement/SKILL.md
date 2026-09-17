---
name: ad-placement
description: Diagnose and fix ad unit placement/injection issues on Flytedesk SSP-managed publisher websites (masthead, sticky bottom, interstitial, in-content/ICV). Use when an ad unit renders in the wrong spot, doesn't appear where expected, or when configuring Placement Settings (Include/Exclude XPath, Injection Algorithm) for an ad unit in the SSP (platform.flytedesk.com).
---

# SSP Ad Unit Placement

## When to use this

- A publisher or account manager reports an ad unit rendering in the wrong location (e.g. "the in-content ad is showing up under the masthead").
- You're configuring placement for a new ad unit, or copying config from one property to a sibling one.
- An ad unit isn't visibly rendering and you need to tell whether that's a placement/injection failure or an ad-fill (Kevel) problem — these are independent failure modes and the fix is completely different for each.

## Step 0 — Open the in-app browser and authenticate

Open the Browser pane and navigate to `platform.flytedesk.com`. **The user authenticates, not you** — never enter a password or other credential yourself. Wait for them to sign in, then confirm you're logged in (screenshot or `read_page`) before doing anything else.

## Step 1 — Reproduce on the live site with the debug console

Every Flytedesk-instrumented publisher site supports a debug console via URL params:

- `?fddebug` — opens the Flytedesk Digital debug console. It renders as a floating button in the bottom-right corner of the page. **It takes 5-10 seconds to appear after the rest of the page loads** — it's the last thing to initialize (async, by design). Don't conclude the console is broken just because it isn't there immediately.
- `&fdtest` (or `?fdtest` if it's the first param) — shortcut that force-enables **Test Mode** straight from the URL, equivalent to flipping the "Test Mode" toggle inside the debug console. It forces Kevel to serve a guaranteed test creative into every ad unit regardless of real inventory/fill.

Combine them: `https://<site>/<path>/?fddebug&fdtest`

**Use Test Mode whenever you need to visually confirm placement.** Production ad fill is frequently zero for a given unit/moment (targeting, flight dates, budget) and tells you nothing about whether placement is correct — an empty, correctly-placed container and a misplaced container both look like "nothing there" until you force a creative into them.

Click the floating button to open the console. Two tabs matter:

- **Overview** — environment, ad unit count, active ads, a Kevel Decision Explainer, and a **Restart** button (re-runs injection with current settings). After toggling Test Mode or after an SSP config change, prefer a full page reload over Restart — reload also catches script-load-order issues that Restart can mask.
- **Ad Units (N)** — one card per ad unit. Click a card to expand it. The magnifying-glass icon next to "LOCATIONS" scrolls the actual rendered ad into view. Each expanded card has a **Logs** button — this is the primary diagnostic tool, see Step 2.

## Step 2 — Read the injector log; don't diagnose from the rendered page alone

Click **Logs** on the ad unit card. For an auto-injected unit you'll see a trace shaped like:

```
Auto injecting ad unit
Injecting ad unit at xPath <Include XPath>  <unit name>
[algorithm-specific reasoning steps]
Ad Unit has N active ads. Requesting decision from kevel
Requesting decision from Kevel (direct fetch): {...}
Decision response from Kevel (direct fetch): {...}
Ads returned for <element id>: <fill or null>
```

Read it for three things:

1. **What XPath it actually evaluated** — confirm it matches the SSP's stored "Include (XPath)" for that unit. If two *different* ad units share the same Include (XPath), that's a red flag — see the Common Bug below.
2. **What the algorithm did with it.** For "In-Content", you'll see it walk spacing offsets between text nodes (`Injector: current offset is X/Y`, `Injector: Rejecting Nodes`). If it logs `Rejecting Nodes: Array[0]`, then jumps to a large/nonsensical offset, then `target offset exceeded... attempting correction`, and finally `Inject element into id(...)` pointing at a **different ad unit's own container** — that is the exact signature of the Common Bug below.
3. **Whether Kevel returned fill.** `Ads returned for <id>: null` / `No ads were assigned to the Ad Unit` means fill, not placement, is the open question — a correctly-placed unit with zero fill renders `display:none` / 0×0 and is invisible, which is expected, not a bug. Don't chase a "missing ad" as a placement issue until you've ruled fill out with Test Mode.

Cross-check the DOM directly when useful:

```js
// Every flytead ad unit currently in the DOM, with its class (which encodes the au_ id) and computed size:
Array.from(document.querySelectorAll('[class*="flytead-au_"]')).map(el => ({
  cls: el.className.match(/au_[A-Za-z0-9]+/)[0],
  display: getComputedStyle(el).display,
  rect: el.getBoundingClientRect(),
}))
```

```js
// Is a given unit nested inside the article body, or inside another ad unit's container?
document.getElementById('<fd-unit id from the log>').parentElement
```

## Step 3 — Know the two placement families

| Family | Units | Algorithm | Include (XPath) target |
|---|---|---|---|
| **Body-level** | Masthead, Sticky Bottom, Interstitial | Prepend / fixed | Almost always `/html/body` — **this is correct** for these units. If one of these looks visually broken, it is rarely the XPath; look at Custom CSS / z-index / stacking-context conflicts instead. These units float or overlay above page content, so they're the ones that collide with a site's own sticky header, cookie banner, or z-index rules. |
| **In-Content (ICV)** | In-Content \| Top, In-Content \| Middle, and similarly named units | "In-Content" or "Append" | Must be scoped to the site's **actual article/content-body container** — never `/html/body`. This is the family that gets misconfigured in practice; see below. |

Don't assume `/html/body` is "the bug" reflexively — check which family you're looking at first. It's correct for one family and wrong for the other.

## Common bug: an In-Content unit scoped to `/html/body`

**Symptom:** an In-Content (ICV) ad renders under or inside the Masthead unit (or another body-level unit) instead of inside the article text — sometimes reported as happening "on the home page," which is a clue, not a contradiction (see why, below).

**Root cause:** the In-Content unit's Include (XPath) is `/html/body` instead of the article body wrapper. The in-content spacing algorithm walks the descendants of whatever Include (XPath) resolves to, looking for real paragraph/text nodes to space ads between. Scoped to the whole `<body>`, it walks nav/header/hero/sidebar along with real content, the spacing math breaks, and it falls back to injecting itself into the nearest already-injected ad container it can find — typically the Masthead's, since that one usually injects first and sits at the very top of `<body>`.

This also explains reports of it happening "only on the home page": `/html/body` matches literally every page, including listing/home pages that have no article body at all — the failure was never article-specific, `/html/body` just happens to produce a visible collision at the top of any page.

**Fix:**

1. Find the site's actual content-wrapper element — do not guess, every CMS template differs. On a real article page:
   ```js
   document.querySelector('main')?.id
   // or inspect visually for the div that wraps just the article text
   ```
   Patterns seen so far on the SNO/SNworks platform (student newspaper sites — e.g. technicianonline.com, alligator.org): an id like `#sno-story-body-content`, or a class like `.article-content`. **Two sites on the same platform can still use different markup** — verify on the actual target site, don't copy an XPath from another property without checking.
2. **Compare against a known-working sibling property on the same ad platform/CMS** rather than relying on theory alone. The same SKU pattern (e.g. `Website | Premium | In-Content`) on a different supplier running the same CMS is strong evidence for what "correct" looks like. Check more than one comparison property if the first result looks unusual — config can drift or be copy-paste-wrong on more than one property at a time.
3. Set **Include (XPath)** to the real container, e.g.:
   ```
   html/body//div[@id="<content-wrapper-id>"]
   ```
   or, matching an existing working convention:
   ```
   html/body//div[contains(concat(' ',normalize-space(@class),' '),' <content-class> ')]
   ```
4. Set **Exclude (XPath)** to keep ads off images/captions, matching the working reference pattern seen across properties:
   ```
   .//figure|.//img|.//figcaption
   ```
5. Leave Minimum Spacing, Close Button, and Custom CSS as they were unless the user asks you to change them too. Don't silently copy a reference property's spacing values — those are a per-property editorial choice, not part of the placement fix.

## Pattern A — Injection Algorithm "Append" vs "In-Content"

**Symptom:** the Include (XPath) is scoped correctly to the article/content-body container, but the ad still lands in the wrong spot within it — most often reported as "the ad is still getting placed at the end of this article" even after the XPath itself was fixed. With two In-Content units on the same page (e.g. Top and Middle), it can also show up as "top in content is showing up below the middle."

**Root cause:** the **Injection Algorithm** field is a separate setting from Include (XPath), and "Append" behaves very differently from "In-Content":

- "Append" is a literal DOM `appendChild` into whatever Include (XPath) resolves to — no spacing/offset logic at all.
- If the XPath targets a specific paragraph (e.g. `.../p[3]`), Append nests the ad **inside** that `<p>`, which is invalid HTML and breaks text flow.
- If the XPath targets the whole container (e.g. `.../entry-content`), Append puts the ad as the **last child** of the container — i.e. at the very end of the article, not distributed through it.
- When two ad units (Top and Middle) both use Append into the *same* container, whichever one injects second in script execution order lands physically below the other — regardless of which one is named "Top" and which is named "Middle." Naming is not placement.

**Diagnostic tell in the injector Logs** (Step 2): the trace still walks `Injector: current offset is X/Y` and a handful of `Injector: Rejecting Nodes` entries, then `Inject element into id(...)` resolves to a location that doesn't match where you'd expect — for Append-into-whole-container, that location is the container's last child.

**Fix:** change the **Injection Algorithm** dropdown from "Append" to "In-Content."

**Critical UI gotcha:** the dropdown is a Tom-Select widget (`select#auto_inject_algorithm`, `data-controller="tom-select"`), not a plain `<select>`. Setting the value via JS on the hidden native `<select>` — e.g. `Object.getOwnPropertyDescriptor(...).set` plus dispatched `change`/`input` events — *looks* like it worked in a screenshot (the visible control shows the new value) but silently fails to persist: the SSP's real saved state doesn't change. The only reliable method:

1. Click the visible Tom-Select control to open it.
2. Scroll to see the options (Append / Prepend / In-Content).
3. Click "In-Content" directly.
4. Click Submit.
5. Verify via the read-only Placement Settings summary view showing "Injection Algorithm: In-Content" after reload — don't trust the open dropdown's own display, reload and re-check the summary.

**Important:** a placement bug can have *both* an XPath problem and an Algorithm problem at the same time. Fixing only the XPath (per the Common bug above) is not sufficient if Algorithm is also wrong — always check both independently.

## Pattern B — Theme lazy-load CSS hides the injected `<img>` (opacity: 0, never fades in)

**Symptom:** the ad **container** renders completely fine — correct size, correct position, close button present, mute icon on video units, click-through link works — but the `<img>` (or video poster) inside it is invisible. Reported by users as "ICVs are blank," "in-content ads are showing all white," or "this one is in the right place, showing up as blank," usually with a screenshot of a real ad container (black bar, mute icon, close button) that's visually empty.

**Root cause** (confirmed via live DevTools and reproduced on 4 separate properties this session, all running the tagDiv "Newspaper" WordPress theme): the theme ships a broad CSS rule along these lines:

```css
body.td-animation-stack-type0 .post img:not(.woocommerce-product-gallery img):not(.rs-pzimg) { opacity: 0; }
```

intended as a scroll-triggered fade-in effect for the theme's *own* lazy-loaded images. The theme's JS flips matching images to `opacity: 1` once its own lazy-load observer sees them — but it never sees flytedesk-injected images, since they aren't part of the theme's lazy-load pipeline, so they stay invisible forever.

**This is easy to misdiagnose as a Kevel fill/inventory problem** ("no ads were assigned," "zero fill") because a quick DOM check might not immediately reveal that the image is present-but-invisible rather than absent. This session repeatedly got this wrong at first. Always check computed opacity/visibility before concluding "no fill" — DOM presence/absence and a quick glance are not enough.

**Diagnostic JS** (run after loading with `?fddebug&fdtest`, wait 8-10s for real render):

```js
document.querySelectorAll('[class*="flytead-au_"] img').forEach(img =>
  console.log(img.closest('[class*="flytead-au_"]').className.match(/au_[A-Za-z0-9]+/)[0], img.src, getComputedStyle(img).opacity));
```

Confirm the exact CSS rule responsible:

```js
Array.from(document.styleSheets).forEach(s => { try { Array.from(s.cssRules).forEach(r => {
  if (r.selectorText && img.matches(r.selectorText.split(',')[0].trim()) && r.style.opacity) console.log(r.selectorText, r.href);
})} catch(e){} });
```

**Fix:** append (never replace or remove existing rules) to the ad unit's existing **Custom CSS** field in the SSP:

```css
div.flytead-au_<UNIT_ID> img {
  opacity: 1 !important;
}
```

`!important` is required to beat the theme's specificity. Unlike the Injection Algorithm dropdown (Pattern A), the Custom CSS field is a plain `<textarea id="auto_inject_css">`, so setting it via native setter + dispatched events works reliably:

```js
const ta = document.getElementById('auto_inject_css');
const newValue = ta.value + '\n\ndiv.flytead-au_XXXX img {\n  opacity: 1 !important;\n}';
const nativeSetter = Object.getOwnPropertyDescriptor(window.HTMLTextAreaElement.prototype, 'value').set;
nativeSetter.call(ta, newValue);
ta.dispatchEvent(new Event('input', {bubbles: true}));
ta.dispatchEvent(new Event('change', {bubbles: true}));
```

Then click the real Submit button.

**Verification note:** after an SSP save, live propagation on this platform commonly takes 6-10+ minutes — not instant, and not a simple browser-cache issue (the debug console's own live config fetch shows the old value for several minutes post-save). Don't conclude a fix failed just because it looks unchanged 30 seconds after saving — wait and re-check.

## Pattern C — Known tooling limitation: some masked/formatted numeric fields resist browser automation

Found on **Minimum Spacing (Above)** (`auto_inject_min_spacing`), displayed as e.g. "1,000 px" — a masked/formatted number input (Cleave.js-style).

**Symptom:** the field reverts to its original value on blur regardless of input method tried — triple-click + type, native-setter + dispatched `input`/`change`/`blur` events, End + repeated Backspace all fail identically. This isn't a business-rule minimum: both a lower and a higher target value were tested, and both reverted the same way. It's a tooling/automation limitation specific to this masked-input widget type, not a validation rule.

**Downstream effect:** if Minimum Spacing is set higher than the actual article's content-container height, the in-content spacing algorithm never finds a valid injection point and the unit simply never renders — not blank, *absent*.

**Guidance:** don't burn more than ~2-3 attempts on this field type once the revert-on-blur signature is confirmed. Hand off to a human to edit it manually in the SSP UI, and note the exact field and target value needed.

## Step 4 — Apply the fix in the SSP

1. `platform.flytedesk.com` → **Suppliers** → find/switch to the supplier → open the property (or use **Inventory** with a Medium = Website filter to browse sibling properties for comparison).
2. On the Ad Units table, use the row's **vertical-ellipsis (⋮) menu → Edit Ad Unit Details**. Clicking the ad unit's name link instead opens its inventory/availability calendar, which is not what you want.
3. Scroll to **Placement Settings**, click the pencil/edit icon, update the field(s), **Submit**.
4. There is no bulk edit — each ad unit (Top, Middle, etc.) has its own Include/Exclude XPath and must be edited individually, even when the fix is identical across all of them.

## Step 5 — Verify, don't just assert

1. Reload the target page with `?fddebug&fdtest`.
2. Confirm **visually** — a screenshot of the ad actually rendering inline in the right spot. A DOM/JS check that a unit is "nested in the right container" is good supporting evidence, but it is not the same claim as "the ad renders visibly in the right spot." If you've only done the DOM check, say so explicitly rather than implying you've seen it render.
3. Confirm the *old* wrong behavior is gone too, not just that the new behavior appears — e.g. re-check that the previously-colliding unit's container is now empty, and that the page where the bug originally showed (home page, etc.) no longer injects the fixed unit there at all.
4. Turn Test Mode back off when you're done, unless the user wants it left on. It's a per-browser-session toggle (not a production setting), but leaving it on is confusing for the next person debugging the same page.

## Anti-patterns

- Calling a fix "confirmed" from DOM structure alone, without a visual render check.
- Assuming `/html/body` is always wrong (it's correct for Masthead/Sticky/Interstitial) or always right (it's the classic bug for In-Content).
- Copying a reference property's Include (XPath) verbatim without checking the target site's own DOM — CMS templates vary even within the same platform family.
- Treating "no ads returned" in the Kevel logs as a placement bug — it's a separate fill/inventory investigation with a different fix.
- Editing Placement Settings before reading the injector Logs. The log tells you exactly what XPath ran and what it found; skipping it means debugging blind.
- Concluding "zero fill / Kevel issue" from DOM presence/absence or a quick glance alone — always verify with computed style (`getComputedStyle(el).opacity`, `.display`) plus a visual screenshot first (see Pattern B).
- Fixing an XPath and stopping there — always also check Injection Algorithm. An XPath-only fix can leave the ad correctly scoped but still landing in the wrong spot (Pattern A).
- Driving a Tom-Select (or similar JS-enhanced) dropdown by setting the value on the underlying hidden `<select>` via JS instead of clicking the real widget — it can appear to work in a screenshot while silently failing to persist (Pattern A).
- Concluding a fix "didn't take" seconds after an SSP save without budgeting for real propagation delay — it commonly takes 6-10+ minutes on this platform, not seconds (Pattern B).
