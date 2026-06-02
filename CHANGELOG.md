# Changelog

All notable changes to this project are documented here.
The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2026-06-02

### Added
- Initial open-source release of the **UI Tweaker** Claude skill.
- Fixed-interface template (`template/panel-template.html`): three-column layout
  (Layers tree / live preview / control panel), three-section control panel,
  type-able number boxes, Figma-style scrub, Photoshop-style linked corner radii,
  icon-based alignment tools, undo/redo with keyboard shortcuts.
- **Transform box** overlay: 8-direction resize handles, rotate handle, live px
  size label. Corners = proportional scale, edges = width/height, body drag = move.
- Atomic layer decomposition driven by the real source code.
- Real-style fidelity rules: link the project's real stylesheet, read initial
  values via `getComputedStyle`, apply with `!important`, never hand-copy values.
- Self-check gate ("preview DOM must mirror real code"): grep every class/id,
  strip non-allowlisted inline styles before opening the panel.
- Generic demo component under `examples/` so the template runs out of the box.

[1.0.0]: https://github.com/dhosruiasn/ui-tweaker/releases/tag/v1.0.0
