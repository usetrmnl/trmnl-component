# Multi-Device Frames for `<trmnl-frame>`

**Date:** 2026-07-07
**Status:** Approved design, pending implementation

## Context

`trmnl-component.js` renders TRMNL plugin content inside a mocked-up device
frame (an inline SVG with an `<iframe>` in a `<foreignObject>`). The frame is
hardcoded to the original ("OG") TRMNL device: 950×639 viewBox, 800×480
screen, TRMNL logo below the screen. trmnl.com uses this component on Recipe
landing pages (e.g. trmnl.com/recipes/11023).

TRMNL now sells a second device, the TRMNL X (shop.trmnl.com/products/trmnl-x),
and the component must be able to render content inside an X frame as well.

Key product facts driving the design:

| | OG | X |
|---|---|---|
| Screen | 7.5", 800×480 (5:3) | 10.3", 1872×1404 (4:3) |
| Front branding | TRMNL logo below screen | None (clean front) |
| Colorways | Black, White, Clear, Sage, Gray, Light/Dark Wood | Black, White, Clear, Sage, Gray, Light Wood |
| Orientation | Landscape | Landscape + portrait |

## Goals

- Render an accurate TRMNL X frame, selectable via a `model` attribute and a
  `setModel()` method.
- Existing embeds (no `model` attribute) render **exactly** as today — the OG
  SVG stays byte-identical.
- X content renders the way trmnl.com production renders X screens: a
  1040×780 logical-pixel viewport (the X's 1872×1404 panel at the framework's
  `--pixel-ratio: 1.8`), scaled down to fit the drawn screen.
  **Amended 2026-07-07 during implementation:** the original assumption that
  plugins.css handles a raw 1872×1404 viewport automatically was wrong — the
  deployed framework sizes screens via `.screen--<device>` modifier classes,
  and the X's is `.screen--v2` (1040×780 @ pixel-ratio 1.8). Bare `.screen`
  defaults to OG's 800×480.
- Adding future devices is a matter of adding one registry entry.

## Non-Goals

- Portrait orientation for the X (follow-up; would be an `orientation`
  attribute later).
- New color names beyond the `mint` → `sage` rename (no `clear` finish).
- Any change to the content-injection, dark-mode, or iframe lifecycle logic
  beyond what model switching requires.

## Design

### 1. Model registry

A `MODELS` object at the top of `trmnl-component.js`, keyed by model name.
Each entry owns everything device-specific:

```js
const MODELS = {
  og: {
    screen: { width: 800, height: 480, scale: 1 },   // iframe CSS px + scale
    svg: `...existing SVG, byte-identical...`,
    colors: {},                                       // per-model palette overrides
  },
  x: {
    screen: { width: 1872, height: 1404, scale: 800 / 1872 },
    screenClass: "screen--v2",  // framework device-size class for slotted content
    svg: `...new X artwork...`,
    colors: {},
  },
};
```

- `screen` drives the iframe's CSS `width`/`height` and the `transform:
  scale(...)` applied to shrink it into the drawn screen box.
- `svg` is the full frame markup, including the `<foreignObject>` that hosts
  the iframe. Each model's SVG must contain gradient stops with ids `start`
  and `end` (queried by `updateColors()`). The logo group (`.logo__path`
  elements) is optional — `updateColors()` already no-ops when the query
  returns nothing.
- `colors` merges over the shared base palette (`{...baseColors,
  ...model.colors}`); starts empty for both models.

Shadow DOM scopes the SVG's internal ids (`path-2`, `filter-3`, `start`,
`end`), and only one model is stamped at a time, so ids cannot collide.

### 2. Selection API

- New `model` attribute: `og` (default) | `x`. Added to
  `observedAttributes`.
- New `setModel(model)` method, chainable, mirroring `setColor()`.
- Unknown model → `console.warn`, fall back to `og` (same pattern as unknown
  colors).
- **Model switching re-stamps the frame.** `_renderFrame()`:
  1. Replace the frame markup (style block + container) in the shadow root
     with the new model's template.
  2. Re-grab `this._iframe`, rewire its `onload` handler.
  3. Re-apply forwarded `class`/`style`, color, and dark mode.
  4. Re-inject content: `updateIframe()` for external `src`, or
     `_prepareThenInjectHTML()` for stored inline HTML — switching is
     lossless from the user's perspective (the iframe itself reloads).
- The constructor stamps the default (OG) frame as today; if a `model`
  attribute is present, `attributeChangedCallback` re-stamps before content
  loads (content injection is already deferred via `requestAnimationFrame` /
  `about:blank` onload, so no double-load).

### 3. X screen rendering

- Iframe laid out at **1872×1404 CSS pixels** (the X's physical resolution).
  The framework's `.screen--v2` class sizes content at 1040×780 logical px
  and applies its own internal `transform: scale(1.8)` up to 1872×1404 — so
  the iframe viewport must be the physical size or the scaled content clips.
  *(Amended 2026-07-07 twice: first to logical 1040×780, then reverted to
  physical after empirically confirming `.screen--v2` self-scales.)*
- Visually shrunk with `transform: scale(800/1872); transform-origin: 0 0;`
  to fill a 4:3 screen box drawn at 800×600 SVG units.
- For slotted (internal) content, the component adds the model's
  `screenClass` (`screen--v2` for X) to injected `.screen` elements so the
  framework sizes them correctly; the stored raw HTML (`getHTML()`) is not
  mutated. External `src` pages are untouched — they style themselves. Bare
  `.screen` without the class would otherwise default to OG's 800×480
  variables. The OG model has no `screenClass` (bare `.screen` already
  defaults to OG dimensions — zero behavior change).
- Dark mode is unchanged: the invert filter applies to the iframe regardless
  of model.
- **Safari/WebKit — outcome (amended 2026-07-07):** the risk materialized.
  WebKit (Playwright and real Safari, user-confirmed) mis-paints scaled
  iframe content inside `foreignObject` — content bleeds past the bezel's
  clip. Moving the transform to the wrapper div did not help; CSS `zoom`
  was worse (mis-scaled AND mis-positioned). The proven fix (prototyped in
  isolation, clean in WebKit/Chromium/Firefox): **the X screen is an HTML
  overlay, not `foreignObject` content.** Design:
  - The X SVG draws the frame only (no `foreignObject`). The container div
    is `position: relative`; a wrapper div is absolutely positioned over
    the drawn screen box using percentages of the viewBox (left 78/956,
    top 62/724, width 800/956, height 600/724), so it tracks the SVG's
    responsive `width: 100%` scaling automatically. It keeps
    `overflow: hidden`, `border-radius`, and the e-ink look
    (`opacity: 0.9; mix-blend-mode: darken`) the foreignObject carried.
  - The iframe inside stays 1872×1404 layout px. Its visual scale must
    track the rendered overlay size, which CSS transforms can't derive, so
    a `ResizeObserver` on the wrapper sets
    `transform: scale(wrapperWidth / 1872)` (initial CSS scale 800/1872 for
    sane first paint; observer set up in `_renderFrame`, reconnected in
    `connectedCallback`, disconnected in `disconnectedCallback`).
  - The OG model is untouched — it keeps its original `foreignObject`
    structure (scale 1 never triggered the bug) under the byte-identical /
    pixel-gate constraint.

### 4. X frame artwork

Drawn from scratch in the OG's visual language, using shop.trmnl.com product
photos as proportion reference:

- Same construction as OG: gradient-filled body (`#start`/`#end` stops),
  outer drop-shadow filter, inner screen rect with inset shadow, screen
  background `#E4E2E1`.
- X-specific: 4:3 screen (800×600 units), corner radius ~34 (slightly larger
  than OG's 28), and **no logo** — the X has a clean, unbranded front.
- **Bezels (amended 2026-07-07 from the 233×192 mm mechanical drawing the
  user provided):** side and top bezels 44 units, bottom bezel 88 units
  (~2× — houses the screen controller and touch bar). Body 888×732
  (aspect 1.2131 ≈ the drawing's 233/192 = 1.2135). The original "slim even
  bezel, no bottom chin" reading came from angled product photos and was
  wrong.
- ViewBox 984×796 (same 48/32 drop-shadow margins as OG), so the X drops
  into existing pages at a similar rendered width. Host pages control final
  size as today (`width: 100%; height: auto`). Overlay screen box sits at
  (92, 76) in viewBox coordinates.
- The bezel/radius/viewBox numbers above are starting points, not contracts:
  final values are tuned during implementation, and the acceptance criterion
  is a side-by-side visual match against the shop.trmnl.com product photos.

### 5. Colors

- One shared base palette for both models.
- **Rename `mint` → `sage`** (canonical, matches shop naming). Keep `mint`
  accepted as a deprecated alias — attribute and `setColor()` both map it to
  `sage` with a one-time `console.warn` — because the CDN build
  (usetrmnl.com/js/latest/trmnl-component.js) is public and existing embeds
  may use it. Hex values are unchanged for the rename.
- **Re-sample the `wood` hex** from the light-wood product photo on
  shop.trmnl.com during implementation (current values `#f9e3cb`/`#ecd4bc`
  read too pink/pale). Acceptance: swatch visually matches the product photo
  side by side.
- Per-model overrides are supported by the registry merge but start empty.

### 6. Targeted fix: broken class/style forwarding

`updateContainerAttribute()` queries `#container`, but the container's id is
randomized (`container-${random}`), so forwarding the host's `class`/`style`
onto the inner container has been silently broken. Fix the selector to use
the generated id. This is in scope because `_renderFrame()` must re-apply
forwarded attributes on model switch, which touches this code path.

### 7. example.html and README

- example.html: add a model selector (OG / X) to the interactive demo;
  replace `mint` with `sage` in both color dropdowns and swatches; document
  `model` / `setModel()` in the API reference section.
- README: document the `model` attribute and `setModel()`, note the X renders
  at the X's framework-logical 1040×780 (physical 1872×1404), document the
  `sage` rename and `mint` deprecation.

## Verification

No test infrastructure exists in this repo; verification is empirical:

1. **OG regression:** headless screenshot (Playwright via npx) of
   example.html before and after the change — the OG frame must be
   pixel-identical.
2. **X rendering:** screenshot review — frame proportions vs product photo,
   content fills the screen, plugin layout is legible at the scaled size.
3. **Behavior matrix, manually in Chrome + Safari + Firefox:** color
   switching on both models (including `mint` alias warning), dark mode on
   both, `setModel()` switching preserves content (both `src` and inline
   HTML modes), unknown model falls back to OG with a warning.

## Rollout

This repo is the source for the public CDN build. Merging to main does not
by itself update usetrmnl.com/js/latest/trmnl-component.js; publishing the
CDN copy and adopting `model="x"` on trmnl.com recipe pages happen outside
this repo.
