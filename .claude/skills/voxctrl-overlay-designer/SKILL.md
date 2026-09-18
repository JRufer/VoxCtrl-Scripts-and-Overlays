---
name: voxctrl-overlay-designer
description: Turns a reference image, screenshot, sketch, or mockup into a custom VoxCtrl voice-overlay (an index.html + style.css pair for VoxCtrl's Settings → Visual & Feedback → Overlay style custom-overlays folder). Use this whenever the user shares an image and wants a VoxCtrl overlay, voice card, HUD, or "overlay style" built or restyled to match it, or asks to reskin/reimagine VoxCtrl's recording overlay in some aesthetic (sci-fi, LCARS, retro, glass, minimal, cassette, terminal, etc.) — even if they don't say "VoxCtrl" by name and just describe "the little overlay that shows up when I talk." Also use it for iterating on an existing custom overlay's look or animations. Do not use it for VoxCtrl's other UI (settings window, tray icon) — only the on-screen recording overlay card.
---

# VoxCtrl overlay designer

Builds a VoxCtrl custom overlay — a CSS-only `index.html` + `style.css`
pair VoxCtrl drops into its recording HUD — from a reference image,
recreating that image's layout as closely as possible and then working
with the user to design how it should *move*: how it appears/disappears,
what ambient motion it has while running, and how its audio-reactive
visualizer animates. The visual recreation is largely objective (match
the image); the animation is a creative decision, which is why it gets
its own step with the user rather than being invented silently.

## The hard constraints (read this before writing anything)

VoxCtrl's custom-overlay folders have a fixed contract, documented in
full in `references/voxctrl-overlays-readme.md` and demonstrated in
`references/example-custom-index.html` / `example-custom-style.css`
(the built-in "Custom/" example VoxCtrl ships). Read the README now if
you haven't internalized it this session — it's short. The essentials:

- **No `<script>`, ever.** The overlay window's CSP is `script-src
  'self'` with no `unsafe-inline`. An inline `<script>` — in the file or
  injected at runtime — is silently blocked with zero visible error; the
  overlay just doesn't do whatever the script was supposed to do. Every
  behavior in your output must come from plain CSS: `var()`, `calc()`,
  `@keyframes`, `transition`. If you catch yourself wanting a script,
  that's a sign to reach for one of the CSS-only techniques in
  `references/animation-catalog.md` instead.
- **One substituted placeholder**, `{{target}}` (alias `{{trigger}}`),
  swapped in once when the HTML renders — the active routing target's
  label (e.g. "Focused Window", "Clipboard"). Keep it somewhere visible;
  it's the one piece of real content VoxCtrl gives the overlay.
- **Six live CSS custom properties**, written continuously onto the
  page root, readable with plain `var()`/`calc()`: `--voxctrl-recording`,
  `--voxctrl-processing`, `--voxctrl-speaking`, `--voxctrl-mcp-recording`
  (each 0 or 1), `--voxctrl-audio-ready` (0 while the mic connects), and
  `--voxctrl-audio-level` (0..1, already smoothed). These are the only
  inputs your design has to react to; `references/animation-catalog.md`
  opens with the three techniques (state-flag priority chains,
  opacity-stack text swaps, blended "living color") that make a rich,
  reactive design possible from just these six numbers.
- **Two files, one folder.** The deliverable is `index.html` +
  `style.css` in a folder named after the design (this becomes the
  style's name in VoxCtrl's Settings dropdown). Both are re-read fresh
  every time the user selects the style or a dictation starts — no
  build step, no bundler, no external requests (VoxCtrl overlays should
  be assumed offline; don't load web fonts or remote assets — pick
  well-supported system font stacks instead).
- **Include `<link rel="stylesheet" href="style.css">` at the top of
  `index.html`, right after any leading comment block.** VoxCtrl itself
  ignores it (it already injects `style.css` on its own when it loads
  the folder), but users routinely sanity-check a build by
  double-clicking `index.html` to open it directly in a plain browser
  before dropping it into VoxCtrl. Without that `<link>`, the file opens
  unstyled — a bare, ugly HTML skeleton — and reads as "the design
  doesn't match the image at all" even when the real overlay (with CSS
  applied) is correct. This one line costs nothing and eliminates that
  false signal.

If the user attaches their *own* copy of the README or an example
overlay in this conversation, prefer it over the bundled copy — VoxCtrl
may have shipped contract changes since this skill was written — and
skim it for anything that's changed (new custom properties, a different
placeholder name) before proceeding.

## Workflow

### 1. Study the reference image

Look closely before writing any code. Note concretely, in your own
head or scratch notes:
- **Layout regions** — header/title area, buttons or chips, a main
  "screen" or content area, footer/status readouts, decorative
  chrome (corner caps, accent bars, borders).
- **Colors** — pull actual hex-ish families, not just "blue." Note
  which colors seem to be fixed chrome vs. which might be meant to
  shift with state (a status indicator, a glowing ring).
- **Typography** — condensed sci-fi caps, rounded friendly sans,
  monospace/terminal, serif/print — this drives the font-family stack
  and letter-spacing choices.
- **The audio-reactive element** — almost every VoxCtrl overlay design
  has *something* meant to visualize live audio (a bar matrix, a
  waveform, a ring, a needle, dots). Identify which visual element in
  the image is (or could plausibly be) that piece — you'll need it in
  step 3.
- **Anything state-worthy** — a status badge, a "recording" dot, a
  label that looks like it should change text — these map to the
  --voxctrl-recording/processing/speaking/mcp-recording flags.

### 2. Recreate the layout in HTML + CSS

Build the static structure and styling first, matching the image as
closely as you reasonably can — positions, proportions, colors, type.
A few things learned from building these that will save you rework:

- Give the card a **fixed width and height** (VoxCtrl sizes the overlay
  window to match the card, the way the built-in styles do at 340×152).
  Pick dimensions that comfortably fit what the image shows, rather than
  cramming a busy design into the built-in size or leaving a simple one
  swimming in space.
- **Flexbox for structural rows, absolute positioning for overlay
  chrome** (corner caps, status badges, floating readouts) works well
  in practice. One trap: an absolutely-positioned child does not
  contribute to its parent's size. If a badge or stacked-text container
  only has absolutely-positioned children (common with the
  opacity-stack text-swap technique), give that container an explicit
  width/height yourself — otherwise it collapses to zero and the text
  inside overflows its border looking clipped.
- Use the **state-flag and living-color** techniques from
  `references/animation-catalog.md` §"Three core techniques" for
  anything that should track VoxCtrl's status, even before you've
  decided on the fancier animations in step 3 — e.g. get the "REC" /
  "PROCESSING" / etc. status readout and its color working as part of
  the base build.
- Preserve `{{target}}` in the output exactly, and don't hallucinate
  additional placeholders — only `{{target}}`/`{{trigger}}` exist.

At this point you should have a working, correctly-shaped overlay with
sensible default motion (a simple fade or the flip reveal from the
built-in style) but before calling it done, move to step 3 — the
animation design is part of the deliverable, not an optional extra.

### 3. Design the motion, with the user

Animation choice is a taste decision, not something to silently
invent. Consult `references/animation-catalog.md` for the full menu
and vocabulary, then use `AskUserQuestion` with three questions,
each offering 3-4 options **tailored to this specific image** (name
concrete techniques, and add a short clause on why each fits — "a
flip reveal, since your mockup already reads like a physical card"):

1. **Entrance/exit** — how the whole overlay should appear and
   disappear (fade+scale, flip, iris/wipe, slide+settle, segmented
   build, materialize — pick the ones that fit).
2. **Ambient/cycling animation** — optional continuous motion while
   it's on screen (sheen sweep, scanline drift, breathing glow,
   blinking telemetry, particle drift, idle dial sway). This one
   should allow multiple selections (`multiSelect: true`) since these
   layer — but mention in the question that 1-2 is usually the sweet
   spot, since more than that gets visually noisy.
3. **Audio visualizer animation** — how the audio-reactive element
   specifically should move (bar/dot matrix, continuous waveform,
   radial pulse ring, VU needle, spectrum bloom, color-reactive glow
   only) — matched to whatever you identified as the visualizer
   element in step 1.

If you're running unattended — no one available to answer (a
background/scheduled task, or the user has stepped away and said not
to wait) — don't block. Pick the option from each category that best
fits the image's aesthetic, say plainly at the top of your response
which choices you made and why, and proceed. That's a reasonable
default, not a fallback to apologize for.

### 4. Implement the chosen animations

Wire each choice to the appropriate `--voxctrl-*` property using the
techniques above: `--active` (derived from the four raw booleans) for
entrance/exit, plain `@keyframes` loops for ambient motion, and
`--voxctrl-audio-level` (plus the living-color blend for tint) for the
visualizer. Keep transitions/durations tasteful — most of these read
better at 0.3-1.5s for state changes and 3-6s for ambient loops; a
visualizer reacting to audio level should feel closer to real-time
(under ~150ms) so it doesn't lag behind the user's voice.

### 5. Verify before delivering

- Grep your own output for `<script` — there should be none.
- Confirm `{{target}}` appears and nothing else looks like an
  unsubstituted placeholder.
- Confirm the `<link rel="stylesheet" href="style.css">` tag from the
  hard constraints is present, so opening `index.html` directly in a
  browser renders styled.
- If you have a headless browser available (Playwright/Chromium is
  preinstalled in this environment), render the page with the CSS
  inlined or linked from the same directory (a relative `href` only
  resolves if both files are read from the same folder — don't put the
  test HTML in a different directory than the CSS) and screenshot it
  across a handful of states: idle/inactive (should be fully invisible
  — `opacity` tied to `--active` starting at 0), recording, processing,
  speaking, mcp-recording, and recording-but-not-ready (`--voxctrl-
  audio-ready: 0`). Look for clipped text, overlapping elements, or a
  badge that doesn't actually change between states — these are exactly
  the bugs that showed up during development of the reference builds
  (an absolutely-positioned stack with no fixed container size, two
  decorative elements sharing the same corner) and they're easy to miss
  without actually rendering the states.
- **Do a side-by-side fidelity check against the original reference
  image, not just a "does it render" check.** Take the screenshot of
  whichever state most resembles the reference image's implied state
  (usually idle or recording) and put it next to the source image
  mentally (or, if you have image-viewing available, literally). Check
  region by region: does each layout region from step 1 land in
  roughly the same position and proportion? Are the color families
  actually close, not just "in the same hue family"? Is the typography
  style (condensed caps vs. rounded vs. monospace) matched? A build
  that passes the render/no-clipping check can still drift noticeably
  from the source image on spacing, scale, or color saturation — this
  step is what catches that before the user does.
- If no headless browser is available, at minimum re-read both files
  and manually trace each `--voxctrl-*` reference to confirm the
  priority-chain flags stay mutually exclusive and nothing is
  positioned to overlap something else at the coordinates you chose,
  and re-read the reference image description/notes from step 1 to
  confirm nothing was dropped.

### 6. Deliver

Package `index.html` + `style.css` into a folder named for the design
(e.g. `TNG`, `GlassPill`, `Cassette`) and send both files to the user.
Tell them, briefly: drop the folder into the custom-overlays directory
shown at the bottom of VoxCtrl's Settings → Visual & Feedback tab, then
pick it from the Overlay style dropdown — both files are re-read live,
so switching the dropdown away and back (or just selecting it) is
enough to see it, no restart needed. Summarize the animation choices
you implemented in a sentence or two so they know what to expect
before they open it.
