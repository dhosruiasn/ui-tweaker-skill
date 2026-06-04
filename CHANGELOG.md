# Changelog

All notable changes to this project are documented here.
The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- Added a stale-panel snapshot rule: when source code changes but an existing
  `ui-tweaker-*.html` still shows old UI, agents must re-sync preview DOM,
  selectors, layers, edit targets, pseudo maps, visual rects, and stage-reset
  fallbacks from the current source and verify with browser `getComputedStyle`.

### Changed
- Updated the Layers tree template so long layer names ellipsize inside the row
  and keep right-side controls such as eye toggles visible when the left pane is
  narrow.

## [1.1.0] - 2026-06-05

### Added
- **Multi-state components**: Step 1 now requires confirming the target screen
  state (loading / detecting / done / empty…) and fully reproducing all of that
  state's fields, not a simplified subset; decide the state from the real JS branch
  that generates the DOM.
- **Text-content editing (optional)**: the control panel can offer a "Text content"
  input at the top of the Typography category to edit display copy directly
  (textContent / input placeholder / first `<option>`), restorable via undo/redo/reset,
  emitted on confirm as `text-content:` with a note to update the real string source.
- **Single-element box/text logical-layer split**: a single element (e.g. one
  `<input>`, where frame and text share one DOM node) can be split in the layer tree
  into a "Box" layer and a "Text" layer via a `ROLES` map (`box`/`text`/`full`) that
  filters categories in `buildPanel` (`showBox`/`showText`). Box props and text props
  are disjoint CSS properties on the same SEL, so they never conflict; confirm emits a
  box block and a text block sharing the same selector.
- **Click-to-drill selection + click-background-to-deselect**: selection now uses
  rect-containment + area sort (`clickStackAt`/`pickAt`) instead of `closest('[data-pick]')`,
  so synthetic layers (text Hug) and child-fills-parent cases (icon in a button) are
  reachable by clicking alone — repeated clicks drill outward (deepest first, cycling),
  and clicking glyphs vs padding picks the text vs the box. Clicking the empty canvas
  deselects (`deselectAll`), with `buildPanel`/`refreshSelectionUI` guarded for the
  no-selection state.
- **Nested selection frames + text "Hug" (Figma-style)**: when a single element is
  split into box/text logical layers, the text layer's frame hugs the actual rendered
  text (measured via an off-stage mirror span, placed by padding/`text-align`/vertical
  center × zoom) through a unified `rectFor(key)` instead of the element box. Selecting a
  text layer dashes its parent (`#tw-parentbox`); hovering shows a solid box + dashed text
  overlay (`#tw-hover`/`#tw-htext`). The Hug frame hides edge handles and maps corner-drag
  to `font-size` (not `_scl`), since box/text share one element and only typography is
  independent.
- **Dual-slot `<input>` text + independent placeholder typography**: `mode:'input'`
  splits an input into `_ph` (before-typing placeholder) and `_val` (after-typing value)
  slots, each with its own text row and an eye toggle to hide/show independently
  (`_phHidden`/`_valHidden`, keeping the original in `ORIG_TEXT`). Because an `<input>`
  shares one font between placeholder and value, independent placeholder typography is
  done via a `::placeholder` rule injected into a per-(state,key) `<style>` from
  `ph:`-prefixed props (not `el.style`), with a slot toggle (after/before font) driving
  the Typography numboxes; confirm emits a separate `[OUT_SEL::placeholder]` block.
- **`<select>` selected-option text editing + data-driven option fidelity**: a new
  text mode `optionSel` reads/writes the *currently-selected* `<option>`
  (`el.options[el.selectedIndex]`), not the first option (`option0`, often a
  "Please choose…" placeholder), because a `<select>` displays the selected option.
  The preview `<select>` must show the real "after-selection" state — one real option
  carrying `selected` — and when option text is composed from data/code (e.g. a theme
  label = `emoji + space + label` from `getThemeParts`/`composeThemeLabelRaw`, emoji
  from `THEME_EMOJI_MAP`), the preview options must reproduce that exact format
  including emoji rather than inventing an emoji-less stand-in set (an extension of the
  "preview must be faithful to the real code" rule). Confirm output notes the option
  text is data, not a literal string — edit the theme data / emoji map, not a literal.
- **SVG icon fill/stroke controls**: when an SVG element is selected, the Color
  category now exposes `fill` / `stroke` separately (each with a "none" transparent
  toggle) plus `stroke-width`, instead of background/text/border — an SVG's tunable
  color surface is fill/stroke. Initial values read via `getComputedStyle`; emitted
  on confirm as `fill:` / `stroke:`.
- **No-paint and SVG-safe effects**: paint-like color rows now keep a permanent
  square-with-slash no-paint control, and inline SVG effects use `filter:
  drop-shadow(...)` instead of `box-shadow` so icons do not gain accidental square
  backgrounds.
- **Filter-safe selection overlays**: SVG or filtered targets no longer receive the
  real-element `.tw-pick` outline, preventing icon drop-shadows from visually
  shadowing the selection frame while keeping overlay selection boxes visible.
- **Computed gradient initialization**: gradient controls now initialize from the
  selected element's real computed `background-image` / `background` instead of
  generic placeholder colors, so generated panels match the visible component.
- **Resizable Layers pane + canvas panning + layer hover frames**: the fixed panel
  template now includes a draggable Layers-pane resize handle, horizontal/vertical
  layer-tree scrolling, Photoshop/Figma-style `Space` + drag canvas panning while
  zoomed, and a non-selecting blue hover overlay (`#tw-hover`) when hovering layer
  rows so the corresponding preview element is visible before selection.
- **Real Pages switching contract**: the skill now requires Pages rows to be actual
  switch targets that update the preview DOM/root, Layers tree, selector maps,
  current selection, and confirm-output context together, or to be labeled as
  separate linked HTML files instead of decorative inactive rows.

### Changed
- **Stage-scoped responsive preview rules**: the skill now requires default stage
  width to match the active device button, device switching to resize inner
  layout containers (not just the outer frame), production desktop media-query
  widths to be overridden inside `#card-stage`, and responsive children to use
  stage-relative sizing (`min()` / `%` / `calc()`) instead of hard-coded widths
  that crop at narrow sizes.
- **Stage-relative fixed element handling**: production `position:fixed` elements
  such as docks / bottom navs must be reset to stage-relative positioning,
  centered with `left:50%` + `translateX(-50%)`, and sized with
  `min(MAX_WIDTH, calc(100% - padding))` so they remain centered and complete at
  every preview width.
- **Preview HTML validity and multi-width browser verification**: the generation
  gate now explicitly checks for invalid `data-pick` placement, truncated closing
  tags, and DOM nesting that would break scoped reset CSS; staged layouts must be
  verified in a real/headless browser across narrow/default/wide widths, including
  reset/undo visual state.
- **Root-key and SVG visibility safeguards**: all template root-key special-cases
  (`card`) must be replaced when a project root key is used, and staged SVG/icon
  previews must preserve inline dimensions plus active/inactive stroke/fill rules
  so reset/undo or stylesheet extraction does not enlarge or hide icons.
- **undo/redo/reset preserves original inline styles**: `restore()` must no longer
  blow away inline styles with `cssText=''` (which erased real inline styles that
  faithful reproduction ships from JS templates, breaking layout until reload). It now
  caches each element's original inline style (`ORIG_STYLE` via `captureOrigStyle()`)
  and restores to it — the same pattern already used for text content (`ORIG_TEXT`).
- **Stage-reset z-index neutralization**: the transform-box guidance now warns that
  when the `#9` stage reset neutralizes a production overlay/modal, it must also reset
  the overlay's `z-index` (production often ships a very high value like `z-index:100300`,
  which forms a stacking context that paints the whole component above `#tw-tbox` and
  hides the blue selection frame). Add `z-index: auto !important;` in the stage reset.
- **Icon placeholder ban**: the faithful-reproduction rule now spells out that icons
  may be inline `<svg>` **or** a class span backed by a CSS background/mask/icon font
  (e.g. a base64-PNG `--cat-src`), and explicitly forbids substituting an
  emoji / unicode / plain-text stand-in for a real icon — copy the real icon element
  and classes verbatim, since a stand-in changes both the visuals and the styleable
  surface (and breaks the confirm output's selector match). Added a matching
  self-check step (scan every icon position for stand-ins before opening the panel).

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

[Unreleased]: https://github.com/dhosruiasn/ui-tweaker/compare/v1.1.0...HEAD
[1.1.0]: https://github.com/dhosruiasn/ui-tweaker/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/dhosruiasn/ui-tweaker/releases/tag/v1.0.0
