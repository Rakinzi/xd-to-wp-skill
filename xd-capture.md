# XD Capture Reference

Detail reference for Step 1 (Capture the Design) in SKILL.md — scrolling XD prototypes, verifying fade-in content, and clicking hotspots to find linked screens.

## Capturing a Scrolling XD Prototype

XD prototypes render on a `<canvas>`-adjacent scrollable `<div>`, not the page body — `document.body.scrollHeight` will lie (often near-zero or one viewport tall).

1. Open the XD URL with chrome-devtools-mcp `new_page`, timeout 45000ms (cold loads can exceed the default 10s and falsely report failure while actually still rendering)
2. Find the real scroll container:
   ```js
   Array.from(document.querySelectorAll('*'))
     .filter(el => el.scrollHeight > el.clientHeight + 50 && el.clientHeight > 200)
   ```
   Look for an element like `#scroll-1` with `scrollHeight` much larger than `clientHeight`.
3. Set `scrollTop` on that element (not `window.scrollTo`) in increments ~ viewport height, `sleep 1` after each, screenshot each step. `take_screenshot fullPage:true` does NOT work here since the real content isn't in document flow.
4. From the screenshots, note: section order, colors (hex if visible), copy text verbatim, image subjects (you'll substitute stock/placeholder images — XD-hosted images aren't fetchable), any empty/placeholder states in the original design (e.g. a card grid with only one card populated — reproduce that faithfully, don't invent fake content for empty slots).

## Fade-In Animations Can Make a Section Look Wrong or Empty

**Scroll-triggered fade-in animations can make a section look wrong or empty on the first screenshot after landing on it.** XD prototypes commonly animate cards/images in as they enter the viewport (opacity 0 → 1, sometimes with a color/position shift mid-transition). A screenshot taken immediately after `scrollTop` can catch elements mid-fade — pale/washed-out colors, missing icons, empty-looking cards — that look like real design content (e.g. a card row appeared to be light lavender before a coincidental second capture, ~1-2s later, revealed the true solid navy). Always `sleep` 1-2s after scrolling to a new section before screenshotting, and if anything still looks unfinished/faded, re-screenshot again before concluding it's the actual design — don't trust the first frame.

**A single quick screenshot after scrolling to a card-grid section is not enough to conclude cards are "empty placeholders."** A real session scrolled to the same section twice earlier in a broader effort and both times concluded (from a screenshot taken after only a 1-4 second dwell) that two of three cards were unpopulated ghost/placeholder outlines — a real, embarrassing misread corrected only when the user pointed at the same screen a third time and a longer dwell (3+ seconds settled, not just post-scroll) revealed all three cards fully populated with solid navy backgrounds and real icon/title/text content all along. The "wait 1-2s for fade-in" guidance above is a *minimum*, not a guarantee — if a section still looks suspiciously sparse (only one of several visually-identical cards populated) after the standard wait, treat that as a signal to wait longer and re-screenshot again, or explicitly flag the uncertainty to the user, rather than concluding "these are unpopulated by design" from limited evidence. Getting this wrong doesn't just cost a wasted screenshot — it can send an entire fix in the wrong direction (several exchanges spent debating card-height sizing against a phantom "cards 2/3 are intentionally sparse" assumption that was never true).

## Clicking Hotspots to Find Linked Screens

**Interactive elements (nav dropdowns, mega-menus, tabs) may or may not be modeled in the XD file as real click-through hotspots.** Don't assume a `:hover` CSS state on your build is equivalent to — or exempt from checking against — the XD. First check whether XD models the interaction at all: click the relevant hotspot for real (see the click-mechanics note below) and see if it navigates to a linked "open" state screen. If it does, that linked screen is the actual design reference — measure it properly (exact panel width/height/position via the screenshot-to-viewport scale factor: screenshot pixels × (real viewport width ÷ screenshot width) = real CSS pixels) rather than guessing proportions from general site styling. If no linked screen exists (clicking does nothing), then no reference exists and it's fair to design the interaction consistent with the rest of the site's visual language — but say so explicitly rather than silently presenting an invented style as if it were XD-verified.

**XD hotspots need real pointer event sequences, not a bare synthetic `click`.** A single `element.dispatchEvent(new MouseEvent('click', {...}))` on the element under the target coordinates (found via `document.elementFromPoint(x, y)`) is usually not sufficient to trigger XD's own interaction handling — it did not fire a linked-screen navigation in one real case even though a real navigation did exist. Dispatch the fuller sequence instead: `pointerdown` → `mousedown` → `pointerup` → `mouseup` → `click`, all as bubbling/cancelable events with matching `clientX`/`clientY` (and `pointerId`/`isPrimary` on the Pointer events) at the same coordinates. Compute those coordinates by taking a screenshot, visually locating the target in screenshot-pixel space, then converting to real CSS pixels with the scale factor above (get real viewport dimensions from the scroll container's `getBoundingClientRect()`, e.g. `document.getElementById('scroll-1').getBoundingClientRect()`) — do not click at raw screenshot-pixel coordinates directly, since the screenshot is rendered at a different (often ~1.4-2x) scale than the live viewport.

XD's own DOM/accessibility tree is opaque (a single unlabeled `generic` node covering the whole canvas) — `take_snapshot` will not give you clickable element references the way it does on a normal webpage. Coordinate-based clicking via `elementFromPoint` is the only reliable approach for XD prototypes specifically.

## Confirm Which XD Screen Is Actually Loaded Before Trusting What You See

XD prototype flows can link between screens, and a `navigate_page`/click during exploration can land on a completely different screen than the one intended (in one project, a scroll or stray interaction silently swapped from the homepage screen to a "Results & Reports" screen mid-session — same site, same visual language, entirely different content, and nothing in the page chrome made this obvious at a glance). Before trusting a screenshot as "the XD reference for this section," confirm the current URL matches the exact screen URL the user gave, especially after any navigation, click, or tab-switch — don't assume the tab still shows what it showed several tool calls ago.
