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
