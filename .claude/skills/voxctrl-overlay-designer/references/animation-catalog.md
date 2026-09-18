# Animation catalog for VoxCtrl custom overlays

VoxCtrl overlays cannot run `<script>` (the window's CSP is `script-src
'self'`, with no `unsafe-inline` — it silently blocks inline script
everywhere, including anything injected into a custom overlay's
`index.html`). Every animation in this catalog is pure CSS: `@keyframes`
for continuous motion, and `var()`/`calc()` reading the `--voxctrl-*`
custom properties VoxCtrl writes onto the page root for anything that
needs to react to live state. Skim this file for language and technique
when proposing options to the user — don't just copy an option verbatim;
adapt the color, timing and shape to the specific design you're building.

## Three core techniques everything below builds on

**1. Mutually-exclusive state flags.** VoxCtrl exposes four raw booleans
(`--voxctrl-recording`, `--voxctrl-processing`, `--voxctrl-speaking`,
`--voxctrl-mcp-recording`) plus `--voxctrl-audio-ready`. Multiple can be
true in theory, but you usually want exactly one "current state" to
drive color/text/animation at a time. Build a priority chain by
multiplying each tier by `(1 - every higher tier)`:

```css
--mcp: var(--voxctrl-mcp-recording, 0);
--proc: calc(var(--voxctrl-processing, 0) * (1 - var(--mcp)));
--speak: calc(var(--voxctrl-speaking, 0) * (1 - var(--mcp)) * (1 - var(--proc)));
--rec: calc(var(--voxctrl-recording, 0) * var(--voxctrl-audio-ready, 1) * (1 - var(--mcp)) * (1 - var(--proc)) * (1 - var(--speak)));
--init: calc(var(--voxctrl-recording, 0) * (1 - var(--voxctrl-audio-ready, 1)) * (1 - var(--mcp)) * (1 - var(--proc)) * (1 - var(--speak)));
```

Now every downstream effect (color, text, glow) can just read `--rec`,
`--proc`, etc. as clean 0/1 switches.

**2. Opacity-stack text swap.** CSS can't change *which words* an
element shows based on a variable — only numeric properties. So for any
label that must say different things in different states (a status
chip, a "DATA STREAM: …" readout), pre-render every possible string as
a sibling `<span>` stacked in the same spot (`position: absolute` inside
a `position: relative` parent with a fixed size — don't let them be
auto-sized by content or the stack collapses), and bind each one's
`opacity` to its state flag. Only the one whose flag is 1 is visible.

**3. Living color.** Instead of branching, blend each state's RGB
channel-by-channel, weighted by that state's (mutually-exclusive) flag:

```css
--wr: calc(245*var(--init) + 244*var(--rec) + 56*var(--proc) + 204*var(--speak) + 255*var(--mcp));
--wg: calc(158*var(--init) + 63*var(--rec) + 189*var(--proc) + 153*var(--speak) + 102*var(--mcp));
--wb: calc(11*var(--init) + 94*var(--rec) + 248*var(--proc) + 204*var(--speak) + 0*var(--mcp));
```

Because exactly one flag is 1 at a time, this resolves to one clean
color — usable anywhere via `rgb(var(--wr) var(--wg) var(--wb))` or, with
an alpha channel, `rgba(var(--wr), var(--wg), var(--wb), 0.4)`. Apply it
to a glow ring, the audio visualizer, a status dot — the whole design
recolors itself as VoxCtrl's state changes, with one source of truth.

Every option below assumes these three techniques are available; use
them rather than reinventing a branchy equivalent.

---

## 1. Entrance / exit (show / hide the whole card)

This is the transition driven by `--active` (typically
`max(recording, processing, speaking, mcp-recording)`) going 0 → 1 and
back. Pick — or propose — one that matches the design's material: a
glass panel materializes differently than a printed card or a hologram.

- **Fade + scale ("soft power-on")** — `opacity` and a subtle
  `scale(0.92 → 1)` together. Calm, works for almost any aesthetic,
  good default when the design doesn't suggest something punchier.
- **Flip reveal** — `transform: rotateY(88deg → 0)` with `perspective`
  on the parent. What the built-in Voice Card style uses; reads as a
  panel rotating into view edge-first. Good for anything card-like or
  skeuomorphic.
- **Iris / wipe** — animate a `clip-path` (circle or inset) from a
  point or edge to full coverage. Reads as a lens opening or a blast
  door sliding away — strong fit for sci-fi/HUD designs.
- **Slide + settle** — `translateY`/`translateX` from off-panel with a
  slight overshoot (`cubic-bezier` with a small bounce), like the card
  is racking into place. Fits mechanical/industrial or game-HUD looks.
- **Segmented build (LCARS-style power-on)** — break the chrome into a
  few pieces (a top bar, a corner cap, the main panel) and stagger each
  piece's own transition with a small `transition-delay`, so the frame
  assembles itself before the content fades in. Fits panel-and-console
  designs with distinct colored chrome segments.
- **Materialize / dissolve** — a radial or linear gradient mask
  animated via `mask-image`/`-webkit-mask-image` position, so the card
  seems to resolve out of static or light. Good for a more "energy
  being"/holographic feel; heavier to get right, mention the tradeoff.

Exit is usually just the reverse of whichever entrance you pick (same
transition, `--active` going back to 0) — call this out rather than
designing a second, different animation, unless the design clearly
wants an abrupt cut instead of a mirrored reveal (e.g. a "power loss"
snap-off for a more mechanical design).

## 2. Ambient / cycling animations (while the card is on screen)

These run continuously and don't need any VoxCtrl state — they're
texture that makes the panel feel alive even when nothing is changing.
Don't pile on more than 2-3 at once, or the design gets visually noisy;
pick ones that suit the material and the specific regions of the
design (a glass panel wants a sheen, a console wants scanlines, a
gauge wants a needle-idle wobble).

- **Sheen sweep** — a soft diagonal highlight band drifting across the
  panel on a loop (`background-position` or `left` animated). Reads as
  light catching glass or brushed metal.
- **Scanline drift** — a faint horizontal band of slightly-raised
  brightness moving top-to-bottom on a slow loop. Reads as a CRT/HUD
  readout; pairs well with monospace data readouts.
- **Breathing glow** — the outer `box-shadow`/`filter: drop-shadow`
  alpha oscillating slowly (3-5s). Reads as "idle but alive," good on
  almost anything with a glow ring already.
- **Blinking telemetry** — small decorative readouts (status codes,
  coordinates, a "REC" dot) blink independently on staggered timers, so
  the panel feels like it's always quietly doing something.
- **Corner/accent pulse** — a small accent shape (an LED, a chip
  corner, an LCARS end-cap) pulses opacity or scale on its own cadence,
  independent from the main state color. Cheap, effective accent.
- **Particle drift** — a handful of small dots/motes drifting slowly
  upward or across with randomized `animation-delay` per element.
  Good for an ethereal/AI-presence aesthetic; costs more markup (one
  element per particle) so mention that tradeoff.
- **Idle needle/dial sway** — for gauge or dial motifs, a slow
  `rotate()` oscillation even at rest, so the instrument doesn't look
  frozen. Distinct from the audio-reactive sweep described below —
  this is what it does with *no* signal.

## 3. Audio visualizer animation (the audio-reactive centerpiece)

This is usually the focal point, so give it the most detailed
treatment and tailor it hard to what the reference image actually
shows (a bar matrix vs. a continuous waveform vs. a radial ring vs. a
single VU needle all want different techniques). In every case, drive
*amplitude* from `--voxctrl-audio-level` (0..1, already smoothed — no
need to smooth it further) and *color* from the living-color blend
above, so the visualizer itself announces the state change.

- **Center-weighted bar/dot matrix** — a row of columns, each with a
  fixed positional weight (`--w`, higher in the middle, tapering at the
  edges) so the meter has a real "VU meter" shape instead of flat bars.
  Per-bar height/opacity is `--w * audio-level` against that bar's row
  threshold. What the built-in Voice Card style uses.
- **Continuous waveform silhouette** — many thin bars with a
  hand-tuned envelope (`--e` per bar, forming a hump/squiggle shape
  rather than a flat line) so it reads as a speech waveform, not a
  generic equalizer. Add a slow idle `scaleY` wobble with staggered
  `animation-delay` per bar so it's never perfectly static, then let
  `audio-level` scale the amplitude on top.
- **Radial pulse ring** — one or more circles whose `scale` and/or
  `border`/`box-shadow` spread is driven by `audio-level`, like a sonar
  ping or a voice-activated halo. Strong fit for minimal/circular
  designs (a pill, a badge, an orb). Layer 2-3 rings with slightly
  different `transition` speeds for a richer, less mechanical pulse.
- **Single VU needle** — a `rotate()` transform on a needle element,
  angle mapped from `audio-level` via `calc()` (e.g. `-40deg` at 0 to
  `40deg` at 1), for a retro analog-meter design. Add a touch of
  overshoot with `transition-timing-function` so it doesn't feel
  digital.
- **Spectrum bloom** — bars or dots that also shift *size* (not just
  height), scaling up from a resting dot into a taller bar as level
  rises, rather than just changing height from a fixed-width bar. Reads
  softer / more organic than a hard bar matrix.
- **Color-reactive glow only** — for very minimal designs where the
  reference image doesn't show an explicit meter at all, the
  "visualizer" can just be the panel's own glow ring or a thin border
  pulsing brightness with `audio-level`. Worth offering when the image
  is minimal and a literal meter would clutter it.

When proposing options to the user, name 3-4 that plausibly fit the
*specific* image they gave you (not the full list) and say in one
clause why each fits — e.g. "a radial pulse ring, since your mockup's
audio element is a circular badge rather than a bar strip."
