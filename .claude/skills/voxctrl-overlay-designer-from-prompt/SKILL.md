---
name: voxctrl-overlay-designer-from-prompt
description: "Builds a custom VoxCtrl voice-overlay (index.html + style.css) from a written description alone, no reference image. Use instead of voxctrl-overlay-designer when the user describes an aesthetic (e.g. 'cyberpunk, dark with neon yellow, circuit board background') rather than attaching a picture."
---

# VoxCtrl overlay designer — from a text prompt

Sibling to `voxctrl-overlay-designer`, which recreates a *reference image*.
This one starts from a *written description* instead — "cyberpunk, dark
with neon yellow, an animated circuit-board background, a reactive
waveform" and nothing to look at. That changes the workflow: instead of
studying an image, you ask a short round of concrete questions to pin
down the ambiguous parts, then build every visual asset (background
pattern, waveform bar weights, etc.) programmatically with a small
Python script rather than hand-typing them, so the result looks organic
rather than repetitive. Use `voxctrl-overlay-designer` instead whenever
the user *does* attach an image, sketch or screenshot.

## The hard constraints (same contract either way)

VoxCtrl's custom-overlay folders have a fixed contract. If a
`voxctrl-overlay-designer` skill or its bundled
`references/voxctrl-overlays-readme.md` is available in this session,
read it now for the full, current version — VoxCtrl may have shipped
contract changes since this was written. Otherwise, the essentials:

- **No `<script>`, ever.** The overlay window's CSP is `script-src
  'self'` with no `unsafe-inline`. It is silently blocked — no visible
  error, the overlay just doesn't do whatever the script was for. Every
  behavior must come from plain CSS: `var()`, `calc()`, `clamp()`,
  `@keyframes`, `transition`, and (for smoothly-animatable custom
  properties) `@property`.
- **One substituted placeholder**, `{{target}}` (alias `{{trigger}}`) —
  the active routing target's label, swapped in once when the HTML
  renders. Keep it visible somewhere; it's the one piece of real content
  VoxCtrl gives the overlay. If the user asks for the current command or
  target to be shown ("Target >> Dictate", a HUD readout, etc.), this
  placeholder is what supplies the dynamic half of it.
- **Six live CSS custom properties**, written continuously onto the page
  root: `--voxctrl-recording`, `--voxctrl-processing`,
  `--voxctrl-speaking`, `--voxctrl-mcp-recording` (each 0 or 1),
  `--voxctrl-audio-ready` (0 while the mic connects), and
  `--voxctrl-audio-level` (0..1, already smoothed). These are the only
  inputs the design can react to.
- **Two files, one folder** — `index.html` + `style.css` in a folder
  named after the design (this becomes the style's name in VoxCtrl's
  Settings dropdown). No build step, no bundler, no external requests
  (assume offline — no web fonts, no remote assets; use system font
  stacks).
- **Include `<link rel="stylesheet" href="style.css">`** near the top of
  `index.html`, right after any leading comment block. VoxCtrl ignores
  it (it injects style.css itself) but it lets a user sanity-check the
  build by opening `index.html` directly in a browser.

## Three core CSS techniques (build every design on these)

1. **Mutually-exclusive state priority chain.** Multiply each tier by
   `(1 - every higher tier)` so exactly one "current state" flag is 1 at
   a time:
   ```css
   --mcp: var(--voxctrl-mcp-recording, 0);
   --proc: calc(var(--voxctrl-processing, 0) * (1 - var(--mcp)));
   --speak: calc(var(--voxctrl-speaking, 0) * (1 - var(--mcp)) * (1 - var(--proc)));
   --rec: calc(var(--voxctrl-recording, 0) * var(--voxctrl-audio-ready, 1) * (1 - var(--mcp)) * (1 - var(--proc)) * (1 - var(--speak)));
   --init: calc(var(--voxctrl-recording, 0) * (1 - var(--voxctrl-audio-ready, 1)) * (1 - var(--mcp)) * (1 - var(--proc)) * (1 - var(--speak)));
   --active: max(var(--voxctrl-recording, 0), var(--voxctrl-processing, 0), var(--voxctrl-speaking, 0), var(--voxctrl-mcp-recording, 0));
   ```
2. **Opacity-stack text swap.** CSS can't change *which words* an
   element shows from a variable. Pre-render every possible string as a
   sibling `<span>` stacked in the same spot (`position: absolute`
   inside a `position: relative` parent with an explicit fixed
   width/height — don't let it auto-size from content, or the stack
   collapses to zero and the text clips). Bind each span's `opacity` to
   its state flag.
3. **Living color.** Blend two colors channel-by-channel, weighted by a
   0..1 driver — a state flag, or (new in this workflow) a continuous
   value like `--voxctrl-audio-level` directly. E.g. white fading to a
   neon accent as the mic level rises (accent = `234 255 0`):
   ```css
   --wr: calc(255 - 21 * var(--voxctrl-audio-level, 0));
   --wg: 255; /* both endpoints are 255 here, so it never needs to move */
   --wb: calc(255 - 255 * var(--voxctrl-audio-level, 0));
   /* then: background: rgb(var(--wr) var(--wg) var(--wb) / <alpha>); */
   ```
   Work out the two endpoint colors, then for each channel write
   `calc(endpointA + (endpointB - endpointA) * driver)` — simplify by
   hand into the constants shown above rather than leaving the raw
   subtraction in the CSS.

## Workflow

### 1. Ask before building

A text prompt always leaves the same handful of things unstated. Use
`AskUserQuestion` (skip only if running unattended — then pick the most
fitting option per category, say so at the top of the reply, and
proceed) with up to four questions, each with options **tailored to
this specific request**, typically covering:

- **Card size/shape** — "short and wide", "a small pill", "a tall
  sidebar strip" all need concrete pixel dimensions. Offer 2-3 concrete
  options (e.g. "560×110px", "480×130px").
- **The audio-reactive centerpiece's style** — if the user said
  "waveform", that could still mean a continuous silhouette, mirrored
  VU bars, or a radial pulse; name the ones that fit their description
  and say why in one clause each.
- **Any dynamic label's behavior** — if they want the current
  target/command shown, ask whether the action word should be static or
  swap per VoxCtrl state (recording/processing/speaking/mcp), since that
  changes the markup (plain text vs. an opacity-stack).
- **Ambient motion beyond the headline animation** — offer 3-4 named
  techniques (scanline sweep, breathing glow, glitch flicker, particle
  drift, sheen sweep — see the full catalog in
  `voxctrl-overlay-designer`'s `references/animation-catalog.md` if
  available) as a multi-select, plus a "keep it minimal" option.

If entrance/exit style doesn't come up naturally in those four, just
pick one that fits the material (fade+scale is always a safe default;
an iris/wipe or flip suits HUD/panel designs) and say which one you
chose and why when you deliver, rather than spending a question on it.

### 2. Generate assets programmatically, don't hand-type them

Anything that should look organic or non-repetitive — a circuit-board
trace pattern, a set of waveform-bar envelope weights, staggered
animation delays — is tedious and visibly repetitive if typed by hand.
Write a short Python script in the scratchpad to generate it instead:

- **Circuit-board / PCB-trace backgrounds**: a random-walk on a grid
  (pick a start point, take orthogonal steps of random length, repeat
  for ~20 traces) produces convincing right-angle traces. Emit each as
  an SVG `<path>` with `id="ctN"`, plus a same-length `<use href="#ctN">`
  copy for an animated "pulse" overlay (see below), plus `<circle>` pads
  at a subset of vertices.
- **Waveform envelope weights**: sum 2-3 sine waves at different
  frequencies/phases plus a little random jitter, per bar index, then
  clamp to a sensible range (e.g. 0.18-1.0). This reads as a natural
  speech waveform rather than a symmetric hump. Emit one `nth-child`
  rule per bar setting `--e` (the envelope weight) and a staggered
  negative `animation-delay` so an idle wobble never looks synchronized.
- Print the generated CSS/HTML straight to stdout in the exact syntax
  you'll paste into the files — saves a transcription pass.

### 3. Animate the circuit/background "shimmer" without a background-image trick

A good CSS-only technique for "traces that shimmer/flow": give each
trace path a dim, always-visible base stroke, then layer a `<use>` copy
of the same path with a bright stroke, `stroke-dasharray: <short-dash>
<huge-gap>`, and animate `stroke-dashoffset` from `0` to
`-2 * pathLength` on a slow linear loop — this reads as a pulse of light
traveling the trace. Stagger each trace's `animation-duration` and a
negative `animation-delay` (computed from its index) so the pulses never
sync up.

### 4. Build, then verify by actually rendering it

A headless Chromium is normally available in this environment
(`/opt/pw-browsers/chromium` via Playwright) — use it. Don't just eyeball
the CSS.

- Grep the output for `<script` (should only appear in comments, if at
  all) and confirm `{{target}}` and the `<link rel="stylesheet">` tag
  are present.
- Load `index.html` in Playwright, replace `{{target}}` with a sample
  string (there's no templating engine outside VoxCtrl itself, so do it
  with a page-level string replace), then set the six `--voxctrl-*`
  properties via `document.documentElement.style.cssText` and screenshot
  across states: idle (must be fully invisible), init
  (`audio-ready: 0`), recording at a couple of different
  `--voxctrl-audio-level` values, processing, speaking, and
  mcp-recording.
- If a color is meant to react to a continuous value (per the
  living-color technique), render at least three level values (e.g. 0,
  0.5, 1.0) and confirm the color actually sweeps the intended range,
  not just that it doesn't crash.
- Screenshot only the card element (`page.locator(...).screenshot(path=...,
  omit_background=True)`), not the full page, when you want to hand the
  user a clean preview with no white page background around it — this
  is also the easiest way to answer "show me what it looks like".
- Check every text region for clipping/overlap at each state, not just
  that something renders.

### 5. Known pitfalls (bugs actually hit building these — check for them)

- **`clip-path: inset()` argument order.** `inset(top right bottom
  left)`. If you want an iris/wipe that only pinches the left/right
  edges, both the top and bottom arguments must be `0%` — writing `50%
  <calc> 50% <calc>` (intending "start pinched, open outward")
  actually clips the top and bottom to 50% *permanently*, leaving a
  zero-height sliver regardless of the animated value. Double check
  which two of the four sides your driver variable is actually in.
- **`align-items: baseline` breaks on a flex item with no in-flow text.**
  If a flex child holds only absolutely-positioned children (e.g. an
  opacity-stack of state-dependent labels) it has no real text baseline
  to align on, and silently falls back to its bottom margin edge —
  visually shifting it relative to sibling text. Use `align-items:
  flex-end` (or `center`) instead, and give the stacked spans and the
  row itself matching `line-height` so they land on the same line.
- **An absolutely-positioned child doesn't size its parent.** If a
  container's only children are `position: absolute`, give the container
  an explicit width/height yourself — otherwise it collapses to zero and
  the text overflows looking clipped.

### 6. Deliver

Package `index.html` + `style.css` into a folder named for the design
and send both files to the user. Tell them to drop the folder into the
custom-overlays directory shown at the bottom of VoxCtrl's Settings →
Visual & Feedback tab, then pick it from the Overlay style dropdown — no
restart needed. If you rendered a preview screenshot in step 4, send
that too; it's usually the fastest way for the user to confirm the look
before dropping it into the app.