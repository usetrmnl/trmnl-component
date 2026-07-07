# Multi-Device Frames Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a TRMNL X device frame to `<trmnl-frame>`, selectable via a `model` attribute, with the OG frame remaining pixel-identical by default.

**Architecture:** A `MODELS` registry in `trmnl-component.js` where each model owns its frame SVG, screen geometry (native iframe resolution + scale), and optional palette overrides. A `frameTemplate(modelName)` function stamps the selected model into the shadow DOM; `_renderFrame()` re-stamps on model change and re-applies color/dark/content state.

**Tech Stack:** Vanilla JS web component (single file, no build, no runtime deps). Verification via Playwright (chromium/webkit/firefox) + pixelmatch, installed in the session scratchpad — never in the repo.

**Spec:** `docs/superpowers/specs/2026-07-07-multi-device-frames-design.md`

**Branch:** `multi-device-frames` (already created; work directly on it, no worktree needed — single-file repo, user requested work here).

## Global Constraints

- The OG SVG markup stays **byte-identical**; OG shadow-DOM output must be visually pixel-identical (verified by screenshot compare).
- `trmnl-component.js` stays a single dependency-free file; no package.json/node_modules in the repo.
- Default model is `og`; unknown model → `console.warn` + fall back to `og`.
- Color `mint` renamed to `sage` (same hex); `mint` stays accepted as deprecated alias with a `console.warn`.
- Landscape only; no portrait work.
- X iframe renders at physical 1872×1404 CSS px, scaled by `800/1872` into an 800×600-unit drawn screen. For slotted content the component adds the model's `screenClass` (`screen--v2`) to injected `.screen` elements — the framework then lays content out at 1040×780 logical and self-scales ×1.8 to fill the physical viewport. Stored raw HTML and external `src` pages are never mutated. **[Amended 2026-07-07: bare `.screen` defaults to OG 800×480, so class injection is required; an intermediate attempt at a logical 1040×780 iframe was reverted because `.screen--v2`'s internal scale(1.8) clips in a logical-size viewport. User approved the screen--v2 approach.]**
- All verification tooling lives in `TOOLING=/private/tmp/claude-501/-Users-ryanckulp-Documents-code-trmnl-trmnl-component/21c79a4b-3e0d-4319-813a-084b2d4bf06e/scratchpad/tooling`.
- Commit at the end of every task with the trailer `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>`.

---

### Task 1: Verification tooling + OG baseline screenshots

Set up Playwright tooling in the scratchpad and capture the "before" OG screenshots that later tasks compare against. No repo changes except nothing — this task produces no commit (tooling is outside the repo).

**Files:**
- Create: `$TOOLING/package.json` (via npm init), `$TOOLING/harness-og-core.html`, `$TOOLING/shoot.js`, `$TOOLING/compare.js`
- Create: `$TOOLING/baseline/og-core.png`

**Interfaces:**
- Produces: `node shoot.js <fileUrl> <outPng> [chromium|webkit|firefox]` — full-page screenshot after a 4s settle.
- Produces: `node compare.js <a.png> <b.png> <diff.png>` — prints `N pixels differ`, exit 0 iff N=0.
- Produces: `$TOOLING/baseline/og-core.png` — baseline for Tasks 2–8.

- [ ] **Step 1: Install tooling**

```bash
TOOLING=/private/tmp/claude-501/-Users-ryanckulp-Documents-code-trmnl-trmnl-component/21c79a4b-3e0d-4319-813a-084b2d4bf06e/scratchpad/tooling
mkdir -p "$TOOLING/baseline" "$TOOLING/shots"
cd "$TOOLING" && npm init -y && npm i playwright pngjs pixelmatch@5 && npx playwright install chromium webkit firefox
```

Expected: installs succeed. (If `webkit`/`firefox` downloads fail, proceed with chromium and note it; they're only needed in Task 8.)

- [ ] **Step 2: Write the OG regression harness page**

Create `$TOOLING/harness-og-core.html`. It renders four empty OG frames (white, black, mint, gray). `wood` is deliberately excluded — its hex legitimately changes in Task 3. `mint` is deliberately included — after the sage rename it must still render identical pixels via the alias.

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <script src="/Users/ryanckulp/Documents/code/trmnl/trmnl-component/trmnl-component.js" defer></script>
  </head>
  <body style="margin: 0; background: #ffffff;">
    <div style="width: 950px;">
      <trmnl-frame color="white"></trmnl-frame>
      <trmnl-frame color="black"></trmnl-frame>
      <trmnl-frame color="mint"></trmnl-frame>
      <trmnl-frame color="gray"></trmnl-frame>
    </div>
  </body>
</html>
```

- [ ] **Step 3: Write the screenshot script**

Create `$TOOLING/shoot.js`:

```js
const { chromium, webkit, firefox } = require("playwright");

(async () => {
  const [, , pageUrl, outPath, engineName = "chromium"] = process.argv;
  const browser = await { chromium, webkit, firefox }[engineName].launch();
  const page = await browser.newPage({
    viewport: { width: 1000, height: 800 },
    deviceScaleFactor: 2,
  });
  await page.goto(pageUrl);
  await page.waitForTimeout(4000);
  await page.screenshot({ path: outPath, fullPage: true });
  await browser.close();
  console.log(`wrote ${outPath}`);
})();
```

- [ ] **Step 4: Write the pixel-compare script**

Create `$TOOLING/compare.js`:

```js
const fs = require("fs");
const { PNG } = require("pngjs");
const pixelmatch = require("pixelmatch");

const a = PNG.sync.read(fs.readFileSync(process.argv[2]));
const b = PNG.sync.read(fs.readFileSync(process.argv[3]));
if (a.width !== b.width || a.height !== b.height) {
  console.log(`SIZE MISMATCH: ${a.width}x${a.height} vs ${b.width}x${b.height}`);
  process.exit(1);
}
const diff = new PNG({ width: a.width, height: a.height });
const n = pixelmatch(a.data, b.data, diff.data, a.width, a.height, { threshold: 0 });
fs.writeFileSync(process.argv[4] || `${process.argv[3]}.diff.png`, PNG.sync.write(diff));
console.log(`${n} pixels differ`);
process.exit(n === 0 ? 0 : 1);
```

- [ ] **Step 5: Capture the baseline**

```bash
cd "$TOOLING" && node shoot.js "file://$TOOLING/harness-og-core.html" baseline/og-core.png chromium
```

Expected: `wrote baseline/og-core.png`.

- [ ] **Step 6: Sanity-check determinism** — shoot a second time to a different path and compare:

```bash
cd "$TOOLING" && node shoot.js "file://$TOOLING/harness-og-core.html" shots/og-core-repeat.png chromium && node compare.js baseline/og-core.png shots/og-core-repeat.png
```

Expected: `0 pixels differ`, exit 0. If nonzero, the harness is nondeterministic — investigate (likely a remote-font race; empty frames should not load remote assets) and fix before proceeding. Also `Read` the baseline PNG to visually confirm 4 correctly-colored OG frames rendered.

---

### Task 2: Refactor to model registry (OG only, zero behavior change)

Restructure `trmnl-component.js` around `MODELS` + `frameTemplate()` + `_renderFrame()` without changing what renders. Also fix the broken `#container` selector (spec §6).

**Files:**
- Modify: `trmnl-component.js`
- Test: Task 1 harness + baseline compare

**Interfaces:**
- Produces: `const MODELS = { og: { screen: {width, height, scale}, colors: {}, svg: "..." } }` (module scope).
- Produces: `function frameTemplate(modelName)` → full shadow-DOM HTML string (style block + `#${containerId}` div wrapping the model's SVG).
- Produces: `this._model` (string, default `"og"`) and `this._renderFrame()` — stamps `frameTemplate(this._model)` into the shadow root, rewires `this._iframe` + its `onload`, re-applies forwarded class/style, color, dark mode.
- Consumes: existing ids (`containerId`, `trmnlComponentIframeId`, etc.), existing `colors` object (renamed in Task 3).

- [ ] **Step 1: Move the OG SVG into the registry.** In `trmnl-component.js`, the SVG block currently spans from the line `<svg width="950px" height="639px" ...>` (line 85) through `</svg>` (line 384) inside `templateHTML`. Above the `templateHTML` definition, add the registry and move the SVG string **verbatim** (it is a template literal already — the `${contentWrapperId}`/`${trmnlComponentIframeId}` interpolations keep working because those constants are module-scoped):

```js
  // Device model registry. Each model owns its frame SVG, the native
  // resolution its screen content renders at, and any palette overrides.
  const MODELS = {
    og: {
      screen: { width: 800, height: 480, scale: 1 },
      colors: {},
      svg: `<svg width="950px" height="639px" viewBox="0 0 950 639" version="1.1" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink">
        ... lines 86-384 of the current file, moved verbatim, unmodified ...
      </svg>`,
    },
  };
```

- [ ] **Step 2: Replace `templateHTML` with `frameTemplate()`.** Delete the `const templateHTML = \`...\`` block entirely and add in its place:

```js
  function frameTemplate(modelName) {
    const model = MODELS[modelName];
    const scaleCSS =
      model.screen.scale !== 1
        ? `
        transform: scale(${model.screen.scale});
        transform-origin: 0 0;`
        : "";
    return `
    <style>
      :host {
        display: inline-block;
        width: auto;
      }
      #${containerId} {
        width: 100%;
        height: auto;
      }
      svg {
        width: 100%;
        height: auto;
        display: block;
      }
      #${mainContentId} {
        width: 100%;
        height: auto;
        display: block;
        background-opacity: 0;
      }
      #${trmnlComponentIframeId} {
        border: none !important;
        width: ${model.screen.width}px;
        height: ${model.screen.height}px;${scaleCSS}
      }
      #${trmnlComponentIframeId}.dark-mode {
        filter: invert(1) brightness(0.9) sepia(25%) contrast(0.75);
        opacity:90%;
      }
    </style>
    <div id="${containerId}">
    ${model.svg}
    </div>
  `;
  }
```

Note: for `og` this emits `width: 800px; height: 480px;` and no transform — the same computed styles as today. (Whitespace inside the container div is layout-inert; the pixel compare in Step 6 is the arbiter.)

- [ ] **Step 3: Rewire the constructor through `_renderFrame()`.** Replace the constructor's template-stamping section — everything from `// Attach a shadow root...` through the `if (this._iframe) {...}` block — with:

```js
      this._model = "og"; // Default device model

      // Attach a shadow root and stamp the default frame
      this.attachShadow({ mode: "open" });
      this._renderFrame();
```

(Keep the surrounding lines: the `console.log`, the `_src/_color/_contentMode/_htmlContent/_darkMode` initializers before it, and `_processSlottedContent()` + the dark-mode media query wiring after it.)

- [ ] **Step 4: Add `_renderFrame()`** as a method on the class (place it right before `_handleIframeLoaded()`):

```js
    _renderFrame() {
      const template = document.createElement("template");
      template.innerHTML = frameTemplate(this._model);
      this.shadowRoot.replaceChildren(template.content.cloneNode(true));

      // Get reference to the iframe for later use
      this._iframe = this.shadowRoot.querySelector(
        `#${trmnlComponentIframeId}`,
      );

      // Setup iframe load handler
      if (this._iframe) {
        this._iframe.onload = () => {
          this._handleIframeLoaded();
        };
      }

      // Re-apply state that lives in the frame's DOM
      this.updateContainerAttribute("class", this.getAttribute("class"));
      this.updateContainerAttribute("style", this.getAttribute("style"));
      this.updateColors();
      this._updateDarkMode();
    }
```

- [ ] **Step 5: Fix the broken container selector and make `updateColors()` model-aware.** In `updateContainerAttribute()`, change `this.shadowRoot.querySelector("#container")` to:

```js
      const container = this.shadowRoot.querySelector(`#${containerId}`);
```

In `updateColors()`, change `const colorSet = colors[this._color];` lookups to merge model overrides — replace the validation + lookup at the top of the method with:

```js
      const palette = { ...colors, ...MODELS[this._model].colors };

      // Validate that the color exists in our palette
      if (!palette[this._color]) {
        console.warn(`Color "${this._color}" not found, defaulting to white`);
        this._color = "white";
      }

      const colorSet = palette[this._color];
```

Note: the container fix means host `class`/`style` now actually forward to the inner div (they were silently dropped before). This is the spec §6 intentional behavior fix; the harness frames carry no class/style, so the pixel compare is unaffected.

- [ ] **Step 6: Verify zero visual change**

```bash
grep -n "templateHTML" /Users/ryanckulp/Documents/code/trmnl/trmnl-component/trmnl-component.js
```

Expected: no output (old template fully removed).

```bash
cd "$TOOLING" && node shoot.js "file://$TOOLING/harness-og-core.html" shots/og-core-task2.png chromium && node compare.js baseline/og-core.png shots/og-core-task2.png
```

Expected: `0 pixels differ`. If it fails, `Read` the diff PNG — the only acceptable fix is making the refactor more faithful, never updating the baseline.

- [ ] **Step 7: Commit**

```bash
cd /Users/ryanckulp/Documents/code/trmnl/trmnl-component && git add trmnl-component.js && git commit -m "refactor frame into model registry; fix container attr forwarding

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 3: Rename mint → sage, deprecation alias, re-sample wood hex

**Files:**
- Modify: `trmnl-component.js` (colors object, `updateColors()`)
- Create: `$TOOLING/sample-wood.js`
- Test: harness compare (mint alias must be pixel-identical) + console-warning check

**Interfaces:**
- Produces: `const baseColors = { white, black, sage, gray, wood }` (renamed from `colors`; each `{start, end, logo}`).
- Produces: `const colorAliases = { mint: "sage" }`.
- Consumes: `MODELS[this._model].colors` merge from Task 2 (update its `...colors` spread to `...baseColors`).

- [ ] **Step 1: Sample the light-wood finish from a product photo.** Find the TRMNL X light-wood product image:

```bash
curl -s https://shop.trmnl.com/products/trmnl-x | grep -oiE 'https://shop\.trmnl\.com/cdn/shop/[a-z]+/[^"?]*' | sort -u | grep -iE 'wood|birch|maple' | head
```

If that greps nothing, run it without the wood filter, download the top candidates, and `Read` them to find the light-wood front view. Download the chosen image to `$TOOLING/wood.jpg`, then `Read` it and note the pixel coordinates of two clean bezel regions (top bezel and bottom bezel, avoiding shadows/reflections/screen).

- [ ] **Step 2: Write and run the sampling script.** Create `$TOOLING/sample-wood.js` (edit the four region rectangles to the coordinates you noted; regions are `[x, y, w, h]` in image pixels):

```js
const { chromium } = require("playwright");
const path = require("path");

const TOP_BEZEL = [400, 150, 300, 30]; // EDIT to actual top-bezel region
const BOTTOM_BEZEL = [400, 900, 300, 30]; // EDIT to actual bottom-bezel region

(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  const result = await page.evaluate(
    async ({ url, top, bottom }) => {
      const img = new Image();
      await new Promise((res, rej) => {
        img.onload = res;
        img.onerror = rej;
        img.src = url;
      });
      const c = document.createElement("canvas");
      c.width = img.naturalWidth;
      c.height = img.naturalHeight;
      const ctx = c.getContext("2d");
      ctx.drawImage(img, 0, 0);
      const avg = ([x, y, w, h]) => {
        const d = ctx.getImageData(x, y, w, h).data;
        let r = 0, g = 0, b = 0, n = 0;
        for (let i = 0; i < d.length; i += 4) {
          r += d[i]; g += d[i + 1]; b += d[i + 2]; n++;
        }
        const hx = (v) => Math.round(v / n).toString(16).padStart(2, "0");
        return `#${hx(r)}${hx(g)}${hx(b)}`;
      };
      const mix = (hex, target, t) => {
        const p = (h) => [1, 3, 5].map((i) => parseInt(h.slice(i, i + 2), 16));
        const [r1, g1, b1] = p(hex);
        const [r2, g2, b2] = p(target);
        const m = (a, b2_) => Math.round(a + (b2_ - a) * t).toString(16).padStart(2, "0");
        return `#${m(r1, r2)}${m(g1, g2)}${m(b1, b2)}`;
      };
      const start = avg(top);
      const end = avg(bottom);
      return {
        dims: [img.naturalWidth, img.naturalHeight],
        wood: { start, end, logo: mix(start, "#ffffff", 0.25) },
      };
    },
    { url: `file://${path.resolve(process.argv[2])}`, top: TOP_BEZEL, bottom: BOTTOM_BEZEL },
  );
  console.log(JSON.stringify(result, null, 2));
  await browser.close();
})();
```

```bash
cd "$TOOLING" && node sample-wood.js wood.jpg
```

Expected: JSON with a `wood: {start, end, logo}` object of plausible warm-wood hexes (in the family of `#e8d5b5`, not pink-tinted like the current `#f9e3cb`). If `start` is darker than `end` (photo lit from below), swap them so the gradient stays light-on-top like the other palettes.

- [ ] **Step 3: Apply the palette changes.** In `trmnl-component.js`, rename `const colors = {` to `const baseColors = {`, rename the `mint:` key to `sage:` (hex values unchanged), replace the three `wood` hex values with the sampled ones, and add above the palette:

```js
  // Deprecated color names, mapped to their canonical replacements.
  const colorAliases = {
    mint: "sage",
  };
```

Update the Task 2 merge in `updateColors()` from `{ ...colors, ... }` to `{ ...baseColors, ... }`, and add alias resolution as the first lines of `updateColors()`:

```js
      if (colorAliases[this._color]) {
        console.warn(
          `Color "${this._color}" is deprecated; use "${colorAliases[this._color]}" instead`,
        );
        this._color = colorAliases[this._color];
      }
```

(Normalizing `this._color` makes the warning fire once per assignment, not on every repaint.)

```bash
grep -cE "colors\b" /Users/ryanckulp/Documents/code/trmnl/trmnl-component/trmnl-component.js || true
grep -n "baseColors\|colorAliases" /Users/ryanckulp/Documents/code/trmnl/trmnl-component/trmnl-component.js
```

Expected: no remaining references to the old bare `colors` identifier; `baseColors`/`colorAliases` referenced in palette + updateColors.

- [ ] **Step 4: Verify — alias renders identically and warns.** The Task 1 harness uses `color="mint"`, so:

```bash
cd "$TOOLING" && node shoot.js "file://$TOOLING/harness-og-core.html" shots/og-core-task3.png chromium && node compare.js baseline/og-core.png shots/og-core-task3.png
```

Expected: `0 pixels differ` (mint alias → same sage hexes; wood not in this harness). Then check the warning fires — create `$TOOLING/check-warn.js`:

```js
const { chromium } = require("playwright");

(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  const warnings = [];
  page.on("console", (m) => m.type() === "warning" && warnings.push(m.text()));
  await page.goto(process.argv[2]);
  await page.waitForTimeout(2000);
  console.log(JSON.stringify(warnings, null, 2));
  await browser.close();
})();
```

```bash
cd "$TOOLING" && node check-warn.js "file://$TOOLING/harness-og-core.html"
```

Expected: exactly one warning containing `Color "mint" is deprecated; use "sage" instead` (one mint frame on the page), and no "not found" warnings.

- [ ] **Step 5: Visually verify wood.** Add a temporary fifth frame `<trmnl-frame color="wood"></trmnl-frame>` to a copy of the harness (`$TOOLING/harness-wood.html`), shoot it, and `Read` the screenshot next to `$TOOLING/wood.jpg`. Acceptance (spec §5): the frame swatch reads as the same wood tone as the photo — warm/tan, not pink. Iterate on the sampled regions if off.

- [ ] **Step 6: Commit**

```bash
cd /Users/ryanckulp/Documents/code/trmnl/trmnl-component && git add trmnl-component.js && git commit -m "rename mint to sage (mint deprecated alias); accurate wood palette

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 4: Model selection plumbing (attribute, method, fallback)

Wire up `model` end-to-end while `og` is still the only registered model — testable via the unknown-model fallback path.

**Files:**
- Modify: `trmnl-component.js` (`observedAttributes`, `attributeChangedCallback`, `connectedCallback`, new `_setModel`/`_reloadContent`/`setModel`)
- Test: `$TOOLING/check-model-plumbing.js`

**Interfaces:**
- Produces: `setModel(model)` public chainable method; `model` observed attribute.
- Produces: `_setModel(model)` — validates against `MODELS`, warns + falls back to `"og"`, no-ops if unchanged, else sets `this._model`, calls `this._renderFrame()` then `this._reloadContent()`.
- Produces: `_reloadContent()` — re-triggers `updateIframe()` (external mode w/ `_src`) or `_prepareThenInjectHTML()` (internal mode w/ `_htmlContent`).
- Consumes: `_renderFrame()` from Task 2.

- [ ] **Step 1: Write the failing check.** Create `$TOOLING/harness-model.html`:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <script src="/Users/ryanckulp/Documents/code/trmnl/trmnl-component/trmnl-component.js" defer></script>
  </head>
  <body style="margin: 0; background: #ffffff;">
    <div style="width: 950px;">
      <trmnl-frame id="t" model="nope" color="white">
        <div class="screen"><div class="view"><h1>MODEL-TEST-CONTENT</h1></div></div>
      </trmnl-frame>
    </div>
  </body>
</html>
```

Create `$TOOLING/check-model-plumbing.js`:

```js
const { chromium } = require("playwright");

(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  const warnings = [];
  page.on("console", (m) => m.type() === "warning" && warnings.push(m.text()));
  await page.goto(process.argv[2]);
  await page.waitForTimeout(2500);

  const result = await page.evaluate(() => {
    const el = document.getElementById("t");
    const iframe = el.shadowRoot.querySelector("iframe");
    return {
      hasSetModel: typeof el.setModel === "function",
      observesModel: el.constructor.observedAttributes.includes("model"),
      contentInjected: iframe.contentDocument.body.innerHTML.includes(
        "MODEL-TEST-CONTENT",
      ),
      chainable: el.setModel("og") === el,
    };
  });
  console.log(JSON.stringify({ warnings, ...result }, null, 2));
  const ok =
    result.hasSetModel &&
    result.observesModel &&
    result.contentInjected &&
    result.chainable &&
    warnings.some((w) => w.includes('Model "nope" not found'));
  console.log(ok ? "PASS" : "FAIL");
  process.exit(ok ? 0 : 1);
})();
```

- [ ] **Step 2: Run it to confirm it fails**

```bash
cd "$TOOLING" && node check-model-plumbing.js "file://$TOOLING/harness-model.html"
```

Expected: `FAIL` (`hasSetModel: false`, `observesModel: false`).

- [ ] **Step 3: Implement the plumbing.** In `trmnl-component.js`:

`observedAttributes`:

```js
    static get observedAttributes() {
      return ["src", "class", "style", "color", "dark", "model"];
    }
```

In `attributeChangedCallback`, add before the final `}`:

```js
      } else if (name === "model") {
        this._setModel(newValue);
```

In `connectedCallback`, add right after the opening `console.log` line (before content/src processing, so content targets the final frame):

```js
      // Initialize device model
      if (this.hasAttribute("model")) {
        this._setModel(this.getAttribute("model"));
      }
```

Add methods after `_renderFrame()`:

```js
    _setModel(model) {
      const requested = model || "og";
      let resolved = requested;
      if (!MODELS[resolved]) {
        console.warn(`Model "${requested}" not found, defaulting to og`);
        resolved = "og";
      }
      if (resolved === this._model) return;

      this._model = resolved;
      this._renderFrame();
      this._reloadContent();
    }

    _reloadContent() {
      if (this._contentMode === "external" && this._src) {
        this.updateIframe();
      } else if (this._contentMode === "internal" && this._htmlContent) {
        this._prepareThenInjectHTML();
      }
    }
```

Add the public method next to `setColor()`:

```js
    /**
     * Set the device model: 'og' or 'x'
     * @param {string} model - The device frame to render
     */
    setModel(model) {
      this._setModel(model);
      return this; // For chaining
    }
```

- [ ] **Step 4: Run the check again**

```bash
cd "$TOOLING" && node check-model-plumbing.js "file://$TOOLING/harness-model.html"
```

Expected: `PASS` (fallback warning fired, content still injected, method chainable).

- [ ] **Step 5: OG regression check** (plumbing must not disturb the default path):

```bash
cd "$TOOLING" && node shoot.js "file://$TOOLING/harness-og-core.html" shots/og-core-task4.png chromium && node compare.js baseline/og-core.png shots/og-core-task4.png
```

Expected: `0 pixels differ`.

- [ ] **Step 6: Commit**

```bash
cd /Users/ryanckulp/Documents/code/trmnl/trmnl-component && git add trmnl-component.js && git commit -m "add model attribute + setModel() with og fallback

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 5: TRMNL X frame artwork + native-resolution screen

**Files:**
- Modify: `trmnl-component.js` (`MODELS.x`)
- Create: `$TOOLING/harness-x.html`
- Test: screenshot review vs product photo

**Interfaces:**
- Produces: `MODELS.x = { screen: { width: 1872, height: 1404, scale: 800 / 1872 }, colors: {}, svg: ... }`.
- Consumes: `frameTemplate()` emits the scale transform automatically (Task 2); `_setModel("x")` now resolves (Task 4). Gradient stop ids `start`/`end` are required by `updateColors()`; the X has no `.logo__path` elements (clean front) and `updateColors()` already tolerates that.

- [ ] **Step 1: Add the X registry entry.** In `MODELS`, after the `og` entry:

```js
    x: {
      screen: { width: 1872, height: 1404, scale: 800 / 1872 },
      colors: {},
      svg: `<svg width="956px" height="724px" viewBox="0 0 956 724" version="1.1" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink">
                <title>TRMNL X</title>
                <defs>
                    <linearGradient x1="50%" y1="0.295017483%" x2="50%" y2="98.7079327%" id="x-gradient">
                        <stop id="start" stop-color="#F7F6F6" offset="0%"></stop>
                        <stop id="end" stop-color="#E9E9E9" offset="100%"></stop>
                    </linearGradient>
                    <rect id="x-body" x="0" y="0" width="860" height="660" rx="34"></rect>
                    <filter x="-1.3%" y="-1.7%" width="102.7%" height="104.0%" filterUnits="objectBoundingBox" id="x-drop-shadow">
                        <feOffset dx="0" dy="2" in="SourceAlpha" result="shadowOffsetOuter1"></feOffset>
                        <feGaussianBlur stdDeviation="3.5" in="shadowOffsetOuter1" result="shadowBlurOuter1"></feGaussianBlur>
                        <feColorMatrix values="0 0 0 0 0   0 0 0 0 0   0 0 0 0 0  0 0 0 0.342083698 0" type="matrix" in="shadowBlurOuter1"></feColorMatrix>
                    </filter>
                    <rect id="x-screen" x="30" y="30" width="800" height="600" rx="8"></rect>
                    <filter x="-0.5%" y="-0.8%" width="101.0%" height="101.7%" filterUnits="objectBoundingBox" id="x-inner-shadow">
                        <feGaussianBlur stdDeviation="3" in="SourceAlpha" result="shadowBlurInner1"></feGaussianBlur>
                        <feOffset dx="0" dy="2" in="shadowBlurInner1" result="shadowOffsetInner1"></feOffset>
                        <feComposite in="shadowOffsetInner1" in2="SourceAlpha" operator="arithmetic" k2="-1" k3="1" result="shadowInnerInner1"></feComposite>
                        <feColorMatrix values="0 0 0 0 0   0 0 0 0 0   0 0 0 0 0  0 0 0 0.5 0" type="matrix" in="shadowInnerInner1"></feColorMatrix>
                    </filter>
                </defs>
                <g id="Artboard" stroke="none" stroke-width="1" fill="none" fill-rule="evenodd">
                    <g id="Group" transform="translate(48, 32)">
                        <g id="Body">
                            <use fill="black" fill-opacity="1" filter="url(#x-drop-shadow)" xlink:href="#x-body"></use>
                            <use fill="url(#x-gradient)" fill-rule="evenodd" xlink:href="#x-body"></use>
                        </g>
                        <g id="Screen">
                            <use fill="#E4E2E1" fill-rule="evenodd" xlink:href="#x-screen"></use>
                            <use fill="black" fill-opacity="1" filter="url(#x-inner-shadow)" xlink:href="#x-screen"></use>
                        </g>

                       <foreignObject class="node" x="30" y="30" width="800" height="600"
                                      style="border-radius: 8px; opacity: 0.9; mix-blend-mode: darken;">
                         <div id="${contentWrapperId}"
                             style="position: static; width: 100%; height: 100%; max-width: 100%; max-height: 100%; overflow: hidden; border-radius: 8px;">
                           <iframe id="${trmnlComponentIframeId}" src="about:blank"
                                   style="display:block; position: static; border: none;"></iframe>
                         </div>
                       </foreignObject>
                   </g>
               </g>
               </svg>`,
    },
```

Design notes baked into this markup (spec §3–4): 4:3 screen at 800×600 units; even 30-unit bezel, no bottom chin, no logo; corner radius 34 vs OG's 28; same gradient/shadow language as OG; `start`/`end` stop ids present for `updateColors()`. Unlike the OG's iframe, the X iframe has **no** inline `width/height: 100%` — the stylesheet's `1872px/1404px` + `scale(800/1872)` must win (1872 × 800/1872 = 800, 1404 × 800/1872 = 600 exactly).

- [ ] **Step 2: Write the X harness.** Create `$TOOLING/harness-x.html`:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <script src="/Users/ryanckulp/Documents/code/trmnl/trmnl-component/trmnl-component.js" defer></script>
  </head>
  <body style="margin: 0; background: #ffffff;">
    <div style="width: 950px;">
      <trmnl-frame id="x1" model="x" color="white">
        <div class="screen">
          <div class="view view--full">
            <div class="layout">
              <span class="value value--xxlarge">Hello TRMNL X!</span>
            </div>
            <div class="title_bar">
              <span class="title">TRMNL X Frame</span>
            </div>
          </div>
        </div>
      </trmnl-frame>
      <trmnl-frame model="x" color="black"></trmnl-frame>
      <trmnl-frame model="x" color="wood"></trmnl-frame>
    </div>
  </body>
</html>
```

- [ ] **Step 3: Shoot and inspect**

```bash
cd "$TOOLING" && node shoot.js "file://$TOOLING/harness-x.html" shots/x-task5.png chromium
```

`Read` the screenshot. Acceptance checklist:
- Frame is 4:3-ish with even bezels, rounded corners, no logo, gradient + shadows matching the OG's style.
- "Hello TRMNL X!" content fills the whole screen (no letterboxing, no overflow, not clipped at 800×480 size — if content occupies only the top-left ~43%, the scale transform is being ignored; if text looks tiny relative to the frame, plugins.css didn't get the 1872×1404 viewport).
- Color variants tint the body correctly (black + wood frames).

- [ ] **Step 4: Compare proportions to the real device.** Download a front-view X product photo (same discovery command as Task 3 Step 1, without the wood filter), `Read` it next to the screenshot, and tune `rx`, bezel inset, and viewBox numbers if the drawn frame reads noticeably off (spec allows tuning; acceptance is the side-by-side match).

- [ ] **Step 5: OG regression check**

```bash
cd "$TOOLING" && node shoot.js "file://$TOOLING/harness-og-core.html" shots/og-core-task5.png chromium && node compare.js baseline/og-core.png shots/og-core-task5.png
```

Expected: `0 pixels differ`.

- [ ] **Step 6: Commit**

```bash
cd /Users/ryanckulp/Documents/code/trmnl/trmnl-component && git add trmnl-component.js && git commit -m "add TRMNL X frame with native 1872x1404 screen rendering

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 6: Model-switch integration check (content persistence)

**Files:**
- Create: `$TOOLING/check-model-switch.js`, `$TOOLING/harness-switch.html`
- Test: this task IS the test; no component changes expected unless it finds a bug

**Interfaces:**
- Consumes: `setModel()` (Task 4), `MODELS.x` (Task 5), inline-content injection (existing).

- [ ] **Step 1: Write the harness.** Create `$TOOLING/harness-switch.html`:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <script src="/Users/ryanckulp/Documents/code/trmnl/trmnl-component/trmnl-component.js" defer></script>
  </head>
  <body style="margin: 0; background: #ffffff;">
    <div style="width: 950px;">
      <trmnl-frame id="t" color="sage" dark="false">
        <div class="screen"><div class="view"><h1>SWITCH-TEST-CONTENT</h1></div></div>
      </trmnl-frame>
    </div>
  </body>
</html>
```

- [ ] **Step 2: Write the check.** Create `$TOOLING/check-model-switch.js`:

```js
const { chromium } = require("playwright");

const read = (page) =>
  page.evaluate(() => {
    const el = document.getElementById("t");
    const iframe = el.shadowRoot.querySelector("iframe");
    const svg = el.shadowRoot.querySelector("svg");
    return {
      viewBox: svg.getAttribute("viewBox"),
      iframeWidth: getComputedStyle(iframe).width,
      hasContent: iframe.contentDocument.body.innerHTML.includes(
        "SWITCH-TEST-CONTENT",
      ),
      gradientStart: el.shadowRoot
        .querySelector("#start")
        .getAttribute("stop-color"),
      darkClass: iframe.classList.contains("dark-mode"),
    };
  });

(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  await page.goto(process.argv[2]);
  await page.waitForTimeout(2500);

  const og = await read(page);
  await page.evaluate(() => document.getElementById("t").setModel("x"));
  await page.waitForTimeout(2500);
  const x = await read(page);
  await page.evaluate(() => document.getElementById("t").setDark("true"));
  await page.waitForTimeout(500);
  const xDark = await read(page);
  await page.evaluate(() => document.getElementById("t").setModel("og"));
  await page.waitForTimeout(2500);
  const back = await read(page);

  console.log(JSON.stringify({ og, x, xDark, back }, null, 2));
  // NOTE: no exact-width assert for og — its iframe has inline `width: 100%`
  // (resolves to the 840px foreignObject), which is legacy behavior we keep.
  const ok =
    og.viewBox === "0 0 950 639" && og.iframeWidth !== "1872px" && og.hasContent &&
    x.viewBox === "0 0 956 724" && x.iframeWidth === "1872px" && x.hasContent &&
    x.screenClassApplied === true && // injected .screen carries screen--v2
    back.screenClassApplied === false && // og re-injection must NOT carry it
    og.gradientStart === x.gradientStart && // sage survived the switch
    xDark.darkClass === true &&
    back.viewBox === "0 0 950 639" && back.hasContent && back.darkClass === true;
  console.log(ok ? "PASS" : "FAIL");
  await browser.close();
  process.exit(ok ? 0 : 1);
})();
```

- [ ] **Step 3: Run it**

```bash
cd "$TOOLING" && node check-model-switch.js "file://$TOOLING/harness-switch.html"
```

Expected: `PASS` — og→x→og switching preserves inline content, sage color, and dark mode across re-stamps, with the iframe resizing 800px↔1872px. The `read()` helper must also return `screenClassApplied: iframe.contentDocument.querySelector(".screen")?.classList.contains("screen--v2") === true`. If it fails, debug with superpowers:systematic-debugging; likely culprits are `_reloadContent()` ordering or the fresh iframe's natural `about:blank` onload racing `_prepareThenInjectHTML()`.

- [ ] **Step 4: Also verify external-src persistence.** Copy the harness to `$TOOLING/harness-switch-src.html`, replacing the `<trmnl-frame ...>...</trmnl-frame>` element with:

```html
      <trmnl-frame id="t" color="sage" dark="false"
        src="https://usetrmnl.com/component-default.html"></trmnl-frame>
```

Run a variant check by editing a copy of `check-model-switch.js` (`check-model-switch-src.js`) that replaces the `hasContent` line with:

```js
      hasContent: iframe.getAttribute("src") === "https://usetrmnl.com/component-default.html",
```

(Cross-origin `contentDocument` is inaccessible; asserting the src survived the model switch is the meaningful invariant.)

```bash
cd "$TOOLING" && node check-model-switch-src.js "file://$TOOLING/harness-switch-src.html"
```

Expected: `PASS`.

- [ ] **Step 5: Commit** (only if component fixes were needed; otherwise nothing to commit — tooling lives outside the repo):

```bash
cd /Users/ryanckulp/Documents/code/trmnl/trmnl-component && git status --short
```

If `trmnl-component.js` changed: `git add trmnl-component.js && git commit -m "fix content persistence across model switches` + co-author trailer.

---

### Task 7: example.html + README

**Files:**
- Modify: `example.html`, `README.md`

**Interfaces:**
- Consumes: `model` attribute, `setModel()`, `sage` color (Tasks 3–5). No code produced.

- [ ] **Step 1: example.html — showcase both models.**
  - Tailwind config: rename `"trmnl-mint": "#D8DCD1"` → `"trmnl-sage": "#D8DCD1"`, and update `"trmnl-wood"` to the Task 3 sampled `start` hex.
  - Both color `<select>`s: `<option value="mint">Mint</option>` → `<option value="sage">Sage</option>`.
  - Swatch row: `bg-trmnl-mint` → `bg-trmnl-sage`, `title="Mint"` → `title="Sage"`.
  - ~~Give the second demo the X frame~~ **[Amended 2026-07-07: the second demo's external src (component-default.html) uses bare OG-sized `.screen` markup, which would render top-left-clustered inside an X frame — external pages style themselves and cannot be class-injected. trmnl-2 stays `og`; the X showcase is the interactive demo's model selector, where slotted content renders correctly via `screen--v2` injection.]**
  - Add a model selector to the first demo, directly after the color dropdown's closing `</div>` of the `mt-6` block (duplicate that block's structure):

```html
                            <!-- Model dropdown selector -->
                            <div class="mt-6">
                                <label
                                    for="model-select"
                                    class="block text-sm/6 font-medium text-gray-900"
                                >
                                    Device Model
                                </label>
                                <div class="mt-2 grid grid-cols-1">
                                    <select
                                        id="model-select"
                                        class="col-start-1 row-start-1 w-full appearance-none rounded-md bg-white py-1.5 pl-3 pr-8 text-base text-gray-900 outline outline-1 -outline-offset-1 outline-gray-300 focus:outline focus:outline-2 focus:-outline-offset-2 focus:outline-indigo-600 sm:text-sm/6"
                                        onchange="document.getElementById('trmnl-1').setAttribute('model', this.value);"
                                    >
                                        <option value="og" selected>TRMNL (OG)</option>
                                        <option value="x">TRMNL X</option>
                                    </select>
                                    <svg
                                        class="pointer-events-none col-start-1 row-start-1 mr-2 size-5 self-center justify-self-end text-gray-500 sm:size-4"
                                        viewBox="0 0 16 16"
                                        fill="currentColor"
                                        aria-hidden="true"
                                    >
                                        <path
                                            fill-rule="evenodd"
                                            d="M4.22 6.22a.75.75 0 0 1 1.06 0L8 8.94l2.72-2.72a.75.75 0 1 1 1.06 1.06l-3.25 3.25a.75.75 0 0 1-1.06 0L4.22 7.28a.75.75 0 0 1 0-1.06Z"
                                            clip-rule="evenodd"
                                        />
                                    </svg>
                                </div>
                            </div>
```

  - API docs section: in the Attributes list, change the `color` entry's values to `white`, `black`, `sage`, `gray`, `wood` and append a new `<li>` for `model` — "Device frame to render (`og`, `x`). Default `og`."; in the Methods list add `<li>` for `setModel(model)` — "Sets the device frame (`og` or `x`)"; in the JS example block replace `setColor('mint')` with `setColor('sage')` and add `terminal.setModel('x');` lines.

- [ ] **Step 2: README.md.**
  - Features list: change the themes bullet to `- **Custom Color Themes**: Choose from white, black, sage, gray, and wood themes` and add `- **Multiple Device Frames**: Render the original TRMNL or the TRMNL X via the \`model\` attribute`.
  - Basic Usage: after the external-content example add:

````markdown
### TRMNL X Frame

```html
<trmnl-frame model="x" src="https://example.com/content.html"></trmnl-frame>
```

Content inside the X frame is laid out at the device's framework-logical
1040×780 (a 1872×1404 physical panel at pixel-ratio 1.8)
resolution and scaled to fit the frame.
````

  - Attributes table: add `| \`model\` | Device frame to render (\`og\`, \`x\`) | \`"og"\` |` and update the color row to `Color theme (white, black, sage, gray, wood; \`mint\` is a deprecated alias for \`sage\`)`.
  - Methods table: add `| \`setModel(model)\` | Sets the device frame (\`og\` or \`x\`) |`.
  - Update the "With Direct HTML Content" example's `color="mint"` to `color="sage"`.

- [ ] **Step 3: Verify example.html renders.** Shoot it and `Read` the screenshot — first demo shows OG, second shows X with external content; controls present:

```bash
cd "$TOOLING" && node shoot.js "file:///Users/ryanckulp/Documents/code/trmnl/trmnl-component/example.html" shots/example-task7.png chromium
```

- [ ] **Step 4: Commit**

```bash
cd /Users/ryanckulp/Documents/code/trmnl/trmnl-component && git add example.html README.md && git commit -m "document model attribute; demo X frame; mint -> sage

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 8: Cross-browser verification + final regression

**Files:**
- Test only; component fixes only if bugs surface (Safari `foreignObject` transform is the known risk, spec §3).

**Interfaces:**
- Consumes: all harnesses and scripts from Tasks 1–6.

- [ ] **Step 1: WebKit (Safari engine) and Firefox screenshots**

```bash
cd "$TOOLING" \
  && node shoot.js "file://$TOOLING/harness-x.html" shots/x-webkit.png webkit \
  && node shoot.js "file://$TOOLING/harness-x.html" shots/x-firefox.png firefox \
  && node shoot.js "file://$TOOLING/harness-og-core.html" shots/og-webkit.png webkit
```

`Read` all three. Acceptance: X frame renders with content correctly scaled in both engines (compare against the Task 5 chromium shot). Known failure mode: WebKit ignoring the scale transform inside `foreignObject` → content renders huge/clipped. If that happens, apply the spec's fallback — move the transform off the iframe onto the wrapper div: in `frameTemplate()`, target `#${contentWrapperId}` with the scale (add `width/height: ${screen.width}px/${screen.height}px` there too) and set the iframe to `width: 100%; height: 100%` for the X — then re-run Tasks 5–6 checks in all three engines.

- [ ] **Step 2: Behavior checks in all engines.** Re-run the switch check under webkit and firefox by launching those engines — edit copies of `check-model-switch.js` replacing `chromium` with `webkit`/`firefox`, or parameterize like `shoot.js`. Expected: `PASS` on all three.

- [ ] **Step 3: Final OG regression gate**

```bash
cd "$TOOLING" && node shoot.js "file://$TOOLING/harness-og-core.html" shots/og-core-final.png chromium && node compare.js baseline/og-core.png shots/og-core-final.png
```

Expected: `0 pixels differ`.

- [ ] **Step 4: Commit** any fixes made (same pattern as Task 6 Step 5). Then run the superpowers:verification-before-completion skill before declaring the plan done, and note for the human: real-device Safari should get a quick manual look (Playwright WebKit ≈ Safari but not identical), and publishing the CDN copy at usetrmnl.com/js/latest/trmnl-component.js is a separate, out-of-repo step (spec Rollout section).
