# VoxCtrl overlay-design skills

Two companion Claude Code skills for building custom VoxCtrl overlays
(`index.html` + `style.css` pairs for VoxCtrl's Settings → Visual &
Feedback → Overlay style custom-overlays folder). Both produce
CSS-only overlays (VoxCtrl's overlay window blocks `<script>`) that
react live to the six `--voxctrl-*` custom properties VoxCtrl writes
onto the page while recording.

- **voxctrl-overlay-designer** — give Claude a reference image,
  screenshot, sketch, or mockup and it recreates that layout, then asks
  a few questions about how it should animate before building it.
- **voxctrl-overlay-designer-from-prompt** — no image, just describe the
  look you want ("cyberpunk, dark with neon yellow, circuit board
  background") and Claude builds it from scratch, generating things
  like waveform envelopes and circuit-trace patterns programmatically
  so they don't look hand-typed/repetitive.

Both skills share the same hard constraints (documented in
`voxctrl-overlay-designer/references/voxctrl-overlays-readme.md`) and
the same core CSS techniques (state-flag priority chains, opacity-stack
text swaps, "living color" blends). Every overlay in `Overlays/` in this
repo was built with one of these two skills.

With Claude Code open on this repo, these are picked up automatically —
just ask Claude to build or restyle an overlay.
