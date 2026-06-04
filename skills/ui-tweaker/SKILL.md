---
name: ui-tweaker
description: Triggers when the user says things like "help me tweak [component]", "I want to fine-tune [style]", "adjust the [property] of this element". Generates a live preview + full control panel; after the user adjusts and confirms, the AI writes the values straight back into the source files.
---

# UI Tweaker

Let users fine-tune UI directly with an AI coding assistant through sliders/number boxes; after they confirm, the AI updates the code automatically — no back-and-forth describing numbers in words.

Core principle: every adjustment is a structured, round-trippable number — never just a "feeling".

> 繁體中文版規格見 [`SKILL.zh-TW.md`](./SKILL.zh-TW.md)（兩份內容等價，行為一致）。

## Distribution and updates

Claude Code is the primary supported install/update target for this skill. This repository is packaged as a Claude Code plugin marketplace (`.claude-plugin/`) plus the actual skill folder (`skills/ui-tweaker/`), so users can install it from the GitHub-backed marketplace and receive updates by refreshing/reinstalling the plugin after new pushes or tags.

Other AI tools may still use the skill instructions, but there is no universal cross-AI skill install/update standard. For Cursor, Windsurf, ChatGPT-style custom instructions, or other agents, treat `skills/ui-tweaker/` as a portable instruction folder and wire it into that tool's own rules/skills mechanism. Updates should be pulled from GitHub with `git pull`, a submodule, a symlink, or a small platform-specific sync script. Do not describe this as live push updates unless the host AI actually supports that behavior.

## Triggers

Any of the following:
- The user says "help me tweak / fine-tune / I want to change" + a component or style name.
- The user pastes code and says "adjust the XX of this".
- The user says "nudge the numbers", "doesn't feel XX enough" (not 3D enough, too big, too cramped, etc.).
- The user uploads a screenshot and points at the part to change.

---

## Flow

### Step 1: Confirm the target component and code

1. Confirm which component the user wants to tweak.
2. If they already pasted the code → go to Step 2.
3. If not → ask: "Paste the code for [component] and I'll open the tweak panel."
4. **Confirm the target state**: if the component has multiple states (loading / detecting / done / empty / expanded vs. collapsed / hover…), first confirm which screen state the user wants to tweak. The preview must **faithfully reproduce ALL fields of that state**, not a simplified subset (e.g. don't reproduce only the thumbnail + name while dropping the coordinate input, dropdown, type toggle, buttons, etc.). A screenshot or description from the user helps pin down the target screen, but since this skill already mandates faithful reproduction of the real code, a screenshot is only an aid for verification, not a prerequisite. Decide the state from the real JS that generates the DOM (e.g. the `status === 'detecting'` branch) and copy that branch's full markup.

---

### Step 2: Generate the "preview" + "control panel" together

#### Always start from the fixed template (highest priority · never re-draw the UI)

This skill ships a fixed-interface template `template/panel-template.html` (relative to this SKILL.md's folder). **Every time you open a control panel, copy this template as the base.** Do not hand-write the UI from scratch and do not change the template's layout/CSS/builder, so that anyone, on any project, opens a byte-for-byte identical interface.

Flow:
1. Read `template/panel-template.html` and copy it to a browser-reachable location in the user's project (e.g. project root, filename like `ui-tweaker-<component>.html`).
2. Replace only the spots marked `⟦PROJECT-SPECIFIC⟧` (everything else stays untouched):
   - **#1** `<title>`
   - **#2** font `<link>` (the project's real fonts)
   - **#3** stylesheet `<link href>` (link the project's real CSS; never hand-copy values into `<style>`)
   - **#4** the preview DOM inside `#card-stage` (paste the real component markup; add `data-pick` to every atomic element)
   - **#5** `SEL` map (pick key → selector used inside the widget)
   - **#6** `OUT_SEL` map (pick key → CSS selector name used in the confirm output)
   - **#7** `LAYERS` tree (decompose the real DOM to atomic level — see "Atomic decomposition")
   - **#8** the confirm-output "component:" / "source:" strings
   - **#9 stage reset**: the rule that resets your component root from hidden/positioned to visible/static inside `#card-stage` (target your real root selector; keep the `--tw-card-transform` var so the card-level transform still works). Only needed when your real component is hidden/absolutely positioned in production.
3. After replacing, every keyed `LAYERS` node must have a matching entry in `SEL` / `OUT_SEL` / the preview DOM's `data-pick`. Before handing off, cross-check key consistency with a script and validate the JS with `node --check`.
4. The template already ships the fixed UI: left Pages/Layers tree, three-section control panel, eight categories, type-able number boxes, two/four-column grids, Figma section titles, PS linked corner radii, position/offset alignment icons, drag-to-scrub, undo/redo + keyboard shortcuts. **Don't rebuild or restyle these.**

> Only fall back to "hand-write per this spec" if the template file is missing/deleted; as long as the template exists, it is the source of truth.

The widget is one HTML page in three columns:
- Left: Pages/Layers tree (Figma-style) showing the component nesting; click to select, Shift to multi-select.
- Middle: live preview (restored by linking the real stylesheet); every adjustable atomic child is clickable.
- Right: control panel (eight categories as collapsible blocks).

Expand the category most relevant to the current element; collapse the rest.

#### Real-style fidelity (highest priority · no hand-copying)

This is the most important rule, overriding any other "restore styles" wording. Preview and control-panel values **must be derived by the browser parsing the real code** — never eyeballed, never hand-copied into the widget. Why: real screens are built from layered `!important` overrides (base rules + design-system layer + pixel-polish layer + final overrides…); copying any single layer will mismatch reality.

1. **Link the real source, don't rewrite it**: the widget always loads the project's real CSS via `<link rel="stylesheet" href="...">` (or `@import` the real file), and loads the project's real fonts in the same `<head>`. Multiple source files → link them all. **Never** paste a selector's values into the widget's `<style>`.
2. **Use the real DOM structure and classes**: preview elements must use the component's true tags, classes, and nesting — no simplified custom versions. If content is injected by JS (dynamic ids), reproduce the real JS-generated markup — including icons, `<span>` wrapping, and formatted strings (e.g. `Today 14:30`, not your own `2025/06/02 14:30`). Read the JS function that generates the DOM first.
   - **Icon placeholder ban (copy the real icon element, never substitute)**: icons are implemented in more than one way — an inline `<svg>`, or a class span + CSS background/mask (e.g. `<span class="cat-icon cat-flower">` driven by a `--cat-src:url(data:image/png;base64,…)` background, or `mask-image`, or an icon font). **Always copy the real icon element and its classes verbatim** and let the linked real stylesheet render it. **Never** substitute an emoji / unicode character / plain text (e.g. 🌸🍄🌱, ★, ↕) to "represent" or "stand in for" a real icon: that changes both the appearance **and the styleable surface the control panel exposes** (a real icon's tunable props are `background`/`mask`/`filter`/`fill`; an emoji's are completely different), which makes the preview unfaithful and the confirm output no longer match the real selector. If you can't tell how an icon is built, go read the JS/CSS that produces the DOM before writing it.
3. **Read initial values via `getComputedStyle`**: every control's initial value must be read with `getComputedStyle(realEl).getPropertyValue(prop)` after the stylesheet is confirmed loaded (`window.load` or poll until the sheet is ready). Never fill in base rules or estimates.
4. **Apply with `!important`**: live preview always applies via `el.style.setProperty(prop, val, 'important')`, otherwise the source's `!important` wins and dragging a slider won't move the screen.
5. **Handle JS-hidden children**: some elements default to `display:none` and are shown by JS only in certain states (e.g. a menu button). The preview must force them into their "when visible" state (e.g. inline `style="display:flex"`), or you'll drop elements.
6. **Detect and protect multi-value properties**: multi-layer `box-shadow`, gradient `background`, etc. can't be losslessly restored by a single-layer slider, and `getComputedStyle` can't read back a single value. When you detect layered/gradient values, lock that control or show a warning so a single-layer slider doesn't clobber the original.
7. **Confirm output contains only changed properties**: when sending `[UI Tweaker Confirm]`, output only properties the user actually changed; never include untouched ones, so you don't accidentally alter things they were already happy with.
8. **Verify on a real browser when done**: read `getComputedStyle` or screenshot to confirm the preview's parsed values match the app's actual values (e.g. compare 2–3 key properties) before handing off.
9. **Stale panel snapshots must be resynced from source**: a generated `ui-tweaker-*.html` is only a snapshot of the project at generation time. If the user says the real code changed but the control panel still shows the old UI, immediately re-read the real JS/HTML/CSS and refresh the preview DOM, `SEL`, `OUT_SEL`, `LAYERS`, `EDIT_TARGETS`, pseudo maps, visual-rect maps, and project-specific stage reset. Search for stale selectors, obsolete hidden nodes, old pseudo-layer names, and outdated CSS variables/fallbacks. Stage-reset CSS may only make the component visible/stage-relative; when it must fallback a visual value, the fallback must point at the current source-of-truth variable from the real stylesheet, not a previous design token. Re-verify in a real browser with `getComputedStyle` against the current source values before handing the panel back.

#### Dynamic JS templates and zero-jargon rule

- If the target component is generated from dynamic JS string templates or components (for example template literals), the AI must proactively reconstruct/mock the final browser-rendered HTML DOM structure inside the preview panel.
- Inline styles bound to variables and dynamically generated text nodes must be parsed and materialized onto the preview DOM. Never extract or treat them as plain strings.
- The AI must proactively identify visually meaningful nodes (for example name inputs, location rows, icons, labels, badges, buttons) and decompose them into the smallest independently clickable elements, each with a corresponding click target or key.
- **Strictly forbidden**: do not require the user to use engineering terms such as "restore DOM", "mock", or "atomic level" in their request. The AI must do this structural conversion in the background and show only an intuitive clickable preview for fine-tuning.

#### Preview interaction rules

- Always restore styles via "link real source + real DOM + `getComputedStyle`" (see above); no fake data, no hand-copied numbers.
- Every adjustable child needs an identifiable key, e.g. `data-focus-target="photo"` / `data-focus-target="glass"`, or `onclick="pick('elementId')"`.
- For icon+text elements (e.g. badge = SVG + time string), let each be **selected and adjusted separately** — don't bind them as one: give the icon a selector (e.g. `.badge svg`); but a bare text node can't be selected or styled alone, so wrap it in a classed `<span>` (e.g. `.badge-text`) in the preview DOM to make it an independent pick target. Note this span usually doesn't exist in the real app — when applying `[UI Tweaker Confirm]` to source, besides editing CSS you must add the same span to the template/JS that generates the DOM, or the style won't attach; flag this prerequisite in the confirm output or your reply first.
- After clicking a preview child, the right panel auto-expands the matching category, scrolls to the relevant controls, and briefly highlights them.
- Switching elements must clear all old selection state, so multiple elements aren't shown selected at once.
- Selection hints must not interfere with shadow/glass/box-shadow previews. Prefer low-interference hints: a thin color bar on the left of the control row, a faint tint on the category title, or a slight glow outside the target.
- Never mix the selection box into the element's own `box-shadow` in `applyDOM()`; the shadow preview must stay pure.
- If you need to outline the target, use a method that doesn't affect the shadow, e.g. an inner outline with `outline-offset: -2px`, or a separate overlay/hit area. Don't use an inset shadow as the selection box.

#### Preview width & history

- Provide device/width switching, at least three widths: narrow, mobile, wide.
- Use an icon-style segmented control for device switching, not long text buttons.
- Width switching only affects the panel preview; it must not rewrite the user's code.
- **The default stage width and the active segmented button must match**: if the widget boots at a custom project width, the active device button must carry that same `data-w`; clicking the already-active button must not resize the stage. Do not leave template defaults such as `390px` / `pick('card')` when the project root key or default width is different.
- **Width switching must drive responsive children, not just the outer frame**: after changing `#card-stage` width, any project-specific stage reset must make the inner layout containers size from the stage (`width:100%; max-width:100%; box-sizing:border-box`) and must override production desktop media-query widths that are based on the browser viewport (e.g. a real app may set a grid to `820px` at `min-width:768px`, which is wrong inside the isolated stage). Use container-derived sizing such as `min()`, `max()`, `clamp()`, `%`, and `calc()` so child components shrink/grow with the selected stage width.
- **Never hard-code responsive child dimensions without a stage-relative cap**: a fixed cell width is acceptable only as a maximum. For a 2-column grid, prefer a pattern like `repeat(2, minmax(0, min(MAX_CELL, calc((100% - GAP) / 2))))`, so narrow stages don't crop the second column.
- Provide a "zoom" feature so users can inspect detail by zooming only the preview, not the whole page: apply CSS `zoom` to the preview stage container (`zoom` scales both rendering and scroll range; with `overflow:auto` you can scroll to detail). Provide −/percent/+ buttons (click the percent to reset to 100%) and `Ctrl/⌘+wheel` plus `Option/Alt+wheel` zoom. Plain wheel/trackpad scrolling must still pan the preview pane after zooming; only modifier+wheel zoom gestures are allowed to zoom or prevent the normal wheel action. Note: `zoom` makes `getBoundingClientRect` return post-zoom coords, so any offset derived from measured rects (alignment, centering) must be divided by the current zoom factor. Zoom is view-only; never written to the user's code.
- Make the toolbar span the full preview width; use `justify-content:space-between` to split into three zones: device width on the far left, zoom controls on the far right, undo/redo/align centered — don't cram everything in the middle.
- Provide undo / redo so the user can revert and reapply slider changes.
- Put undo/redo buttons near the width switcher, as icon buttons.
- Besides buttons, bind keyboard shortcuts: `Ctrl/⌘+Z` undo, `Ctrl/⌘+Shift+Z` or `Ctrl+Y` redo (on `document` `keydown`, `preventDefault` on match). Shortcuts go through the same history functions as the buttons; don't reload the page or clear current values.
- History should record each completed control change; while dragging you may preview live on `input`, but only write one history entry on `change` (or equivalent commit event).
- Undo/redo must not reload the page or clear the user's current adjustments.
- If you persist panel state in `localStorage`, also save current state after undo/redo.

#### Position offset & alignment guides

- X/Y offset sliders should snap near 0: auto-zero within `±3px` (or equivalent units).
- When snapped to 0, clearly mark "centered" or show a blue state.
- When X zeroes, show a vertical guide + "horizontally centered" label.
- When Y zeroes, show a horizontal guide + "vertically centered" label.
- Guides should appear briefly then fade, around `0.7s`.
- **Smart Guides while moving**: dragging the transform box body must show dynamic alignment guides without requiring manual ruler guides. During drag, compare the moving selection's left/center/right and top/middle/bottom against the preview stage and all other selectable elements' left/center/right and top/middle/bottom. Within a small screen-space threshold (about `6px`), magnetically snap the drag delta to the nearest guide, draw a green vertical/horizontal guide line at about 50% opacity, and show a small live movement readout such as `X 12 px   Y -4 px`. The readout and guides are view-only overlays; they must clear on mouseup and must not be emitted in confirm output.
- **Rulers + manual guides**: the toolbar guide button must mean "show/hide rulers", not "pin center lines". Bind `Ctrl/⌘+R` to toggle rulers and `preventDefault()` so the browser does not reload. Rulers should look like design-tool edge rulers: light neutral strips with dark short ticks along the lower/right edge, not a generic grid fill. The ruler toolbar icon must visually match the weight of nearby toolbar icons. When rulers are visible, show horizontal and vertical ruler strips pinned to the preview viewport while the canvas scrolls. Dragging from the horizontal ruler creates a horizontal manual guide; dragging from the vertical ruler creates a vertical manual guide. Clicking a manual guide selects it; selected manual guides may deepen the line color but must not add a thick frame/box-shadow around the line. Clicking preview background or a normal preview element clears manual-guide selection. `X`, `Delete`, or `Backspace` removes the selected manual guide when focus is not inside an input. Manual guides and their pixel readouts are view-only overlays, remain available while the panel is open, update on scroll/zoom/resize, avoid overlapping toolbar buttons by sitting below the toolbar when near the top edge, and must not be emitted in confirm output.
- **Manual guides participate in snapping**: when dragging out or moving a manual guide, snap it to the preview stage's edges/center and nearby selectable elements' edges/center/middle within the same small threshold used by Smart Guides, and show the matched target label in the pixel readout. When moving selected elements, include existing manual guides as Smart Guide candidates so components can snap to user-created guides.
- Support PS-style multi-select mutual alignment: `Shift+click` (in the preview or the layer row) toggles add/remove from the multi-select set (the last one can't be cleared), `cur` stays the last-interacted primary element and the panel shows it. When ≥2 are selected, show a row of alignment buttons (left/h-center/right, top/v-center/bottom) on the same row as undo/redo; hide them when <2. Alignment is based on the bbox of the selection set, achieved via offset props (translate `_tx`/`_ty`) and recorded in history. Note: a selected element must be transform-movable (block / inline-block / flex or grid item, etc.); pure `display:inline` elements ignore transform, so first ensure it's a flex item or make it inline-block. Vertically aligning elements already flex-aligned in the same row may move them imperceptibly — that's normal.
- Alignment controls must choose the right property by the container's `display`, or applying it won't move anything:
  - **Horizontal align** on a flex/inline-flex container via `text-align` won't affect its children — use the main-axis property instead; only non-flex elements use `text-align`.
  - **Vertical align** can never be done with `text-align` alone — use the cross-axis property (flex container); for non-flex elements that need vertical centering, wrap with `display:flex; align-items:center` with minimal side effects. Besides the horizontal row (left/center/right), provide a vertical row (top/center/bottom).
  - **Axis depends on `flex-direction`**: `row` → horizontal=`justify-content`, vertical=`align-items`; `column` → swapped. Read `display` and `flex-direction` via `getComputedStyle(el)` first. Value mapping: start→`flex-start`, center→`center`, end→`flex-end`.
  - Alignment output must carry the actually-effective property (flex container → output `justify-content`/`align-items`), so confirm doesn't mistakenly change it to an ineffective `text-align`.

#### Fixed interface design (Figma/PS style · built into the template)

These are implemented in the template and must stay consistent. Reuse them for new components; don't restyle or rearrange:

- **Three-column layout with resize handle**: the shell uses `grid-template-columns: var(--tw-left-w, 230px) 6px 1fr 360px`. Left = Layers/Pages tree, the 6px track = draggable resize handle, middle = preview stage, right = control panel. The left pane must be horizontally/vertically scrollable (`overflow:auto`), layer names must not be hard-truncated, and the resize handle should persist the width in `localStorage` so deeply nested layers remain readable.
- **Pages are real switch targets**: if a panel contains multiple pages/screens/states, the left Pages section must list each one as a selectable target. Clicking a page must switch the preview DOM/root, Layers tree, `SEL`/`OUT_SEL`, current selection, and confirm-output context together. Do not add decorative page rows that cannot switch anything. If pages are split across separate HTML files instead, label/link them clearly rather than implying in-panel switching.
- **Three-section control panel** (fixes the "confirm bar floating mid-list" bug): `.tw-right` is `display:flex; flex-direction:column; height:100vh; overflow:hidden`; fixed header (`flex:none`) + scrollable `#panel` (`flex:1; min-height:0; overflow-y:auto`) + fixed bottom confirm bar (`flex:none`). **Don't use a `position:sticky` bottom confirm bar.**
- **Figma section titles**: the eight categories are collapsible blocks (`cat(title, open, [...])`); titles have no number prefix (plain text, no "① ⑥").
- **Type-able number boxes** (instead of sliders): `numBox(prop,label,min,max,step,unit,init,snap)` produces an `<input type=text inputmode=decimal>`; arrow keys ↑↓ step (Shift×10), Enter blurs, focus selects all. X/Y fields may show a "Center" state only at exact `0`; never coerce typed values like `1`, `2`, or `3` back to `0`.
- **Multi-column grid**: combine related fields into two/four columns with `fieldGrid(cols, fields)` (e.g. padding TRBL, shadow XYBlur).
- **Drag-to-scrub (Figma scrub)**: `attachScrub(...)`, horizontal drag on a label/icon changes the value (1px≈1 step), live preview while dragging, write one history entry on release. Cursor `ew-resize`. **No sliders.**
- **PS-style radius**: `radiusControl(key)` is a 2×2 four-corner hand-input grid (tl/tr/bl/br) + a central link button; when link is on, adjusting one corner applies the value to the other three (call `applyNum` directly — **don't dispatch events** — to avoid binding-order bugs).
- **Position offset via alignment icons**: `alignTools(key)` in three rows (box position / in-box text / vertical align), using icon buttons (not text); choose the property by container `display`/`flex-direction` (flex → `justify-content`/`align-items`; non-flex → `text-align`; vertical centering can't rely on `text-align` alone).
- **Multi-select style controls apply to every selected key**: when `multiKeys().length > 1`, normal style controls (typography, spacing, size, radius, position, color, and similar shared CSS properties) must update all selected keys, not only `cur`. Keep text-content editing single-key by default so selecting multiple labels does not accidentally rewrite all copy to the same string.
- **Left Layers/Pages tree**: `LAYERS` nesting mirrors the real DOM; `renderLayerNode` renders recursively, keyed nodes are selectable, Shift multi-select, group nodes collapse; keyed row clicks must call `e.stopPropagation()` before `pick(...)` so nested child rows do not bubble into parent rows and overwrite the intended selection; `refreshSelectionUI` syncs the `.on` state and `#cur-name`. Hovering any keyed layer row must show a blue preview hover frame (`#tw-hover`) around the corresponding element using `rectFor(key)`; hover must not select the layer, change `cur`, write history, or pollute the element's own `box-shadow`. The hover overlay is independent from `#tw-tbox`, sits just below it in `z-index`, has `pointer-events:none`, and updates on scroll, resize, zoom, width switching, and spacebar panning. Selecting a preview element must automatically expand all ancestor layer groups and scroll the matching layer row into view. Each keyed layer row must include an eye toggle that hides/shows only the preview element with a preview-only class; the eye button should be hidden until the row is hovered or keyboard-focused so the tree stays readable. Layer visibility toggles are not emitted in confirm output.
- **Photoshop/Figma-style canvas pan**: the preview pane must support holding `Space` and dragging to pan the scrollable preview canvas, especially while zoomed. Ignore the shortcut while typing in inputs/selects/textareas/contenteditable elements, keep toolbar clicks usable, temporarily disable preview hit-testing while panning, and call the overlay update functions during pan so selection/hover frames stay aligned.
- **Transform box (selection box = transformable)**: the selection box is a pure overlay on `#tw-left` (`#tw-tbox`, **not mixed into box-shadow**); single-select shows 8-direction handles + a top rotate handle + a live px size label. Semantics: **drag corners = proportional scale** (writes `_scl`, origin at center, ratio cancels zoom), **drag edges = change `width`/`height`** (screen delta ÷zoom → CSS px), **drag body = change `_tx`/`_ty`** (screen delta ÷zoom → CSS px; do not auto-snap small values to zero; an un-dragged single click uses `elementFromPoint` to select the element underneath, Shift multi-select), **rotate handle = change `_rot`**. Multi-select shows only the group bbox + group move, hiding scale/rotate handles. All drags `applyEl` live + sync the matching number boxes (`i-width`/`i-height`/`i-_scl`/`i-_rot`/`i-_tx`/`i-_ty`); only on release `commit()` writes one history entry and feeds the confirm output. Position comes from `updateTBox()` using `rectFor(key)` converted relative to `#tw-left`; `rectFor` must filter zero-size rects and provide SVG/icon fallbacks (e.g. `getBBox()` + `getScreenCTM()`) so selected SVGs, masked icons, and visual spans still show a frame. `scheduleTBox()` fires on `applyEl`/zoom/width-switch/scroll/resize. Add global arrow-key nudge when focus is not inside an input/select/textarea/contenteditable: Left/Right/Up/Down move the selected key(s) by 1px, Shift by 10px; Up decreases `_ty` and Down increases `_ty` so movement matches screen direction. Note: pure `display:inline` elements (text, tag spans) ignore both transform and width/height, so the transform box won't move them — same limitation as the position-offset controls. The `_rot` number box range must be widened to `-180..180` for the rotate handle. **`#tw-tbox`'s `z-index` must be higher than the preview component's own `z-index`** (e.g. some card containers carry `z-index:100`, so the overlay defaults to `z-index:99999`); otherwise the overlay is covered by the component's content and only a few px of the outermost edge show — appearing as "only the outermost container shows the blue box, inner elements have none". For a new project, if the element tree has a higher `z-index`, raise the overlay further. **Important: when the stage reset (#9) neutralizes a production overlay, also reset its `z-index`**: many full-screen overlays / modals ship a very high `z-index` in production (e.g. `z-index:100300`), and that high value plus `position` forms a stacking context that paints the whole component above `#tw-tbox` (even at 99999), hiding the blue frame entirely. So when #9 resets the overlay into a stage element, besides neutralizing `position`/`display`/`background`, add `z-index: auto !important;` (or a low value) so it stacks normally inside the stage — the production high z-index is meaningless on the isolated stage.
- **Hidden DOM / pseudo-element visual rects**: if a layer key points at a real DOM node that is intentionally hidden by production CSS (`display:none`, `visibility:hidden`, `0×0`) while the visible surface is rendered by a parent, `::before`/`::after`, `mask`, or `background-image`, do not leave the layer frameless. Add a project-specific `VISUAL_RECTS` map and route `rectFor(key)` through `visualRectFor(key)` when the real rect is invalid; for visible parent nodes that represent a pseudo-element subregion, allow a forced visual rect (`force:true`) so the pseudo layer frames the pseudo pixels rather than the parent box. Use `type:'alias'` for a hidden node whose visible surface is represented by another key, and `type:'rect'` / custom project rules for pseudo-element subregions. If a semantic key's DOM is hidden or pseudo-rendered, also add `EDIT_TARGETS` or a pseudo-element write path so control values affect the visible surface; a layer with a blue frame but controls writing to `display:none` DOM or the wrong parent surface is invalid. The Layers tree may still expose the semantic layer name, but selection/hover/click-to-drill must frame and edit the actual visible pixels.
- **Pinned toolbar**: the preview toolbar (device width, undo/redo, reset/zoom) must stay fixed to the viewport above every preview overlay, but bounded to the center preview pane only. Do not use full-window `left:0; right:0`, because that covers the Layers pane and the right control pane. With the standard shell, use `left:calc(var(--tw-left-w, 230px) + 6px); right:360px;` plus a z-index higher than `#tw-tbox` (for example `100002`) and `pointer-events:none` on the wrapper with `pointer-events:auto` on toolbar groups. Toolbar click/mousedown must `stopPropagation()` so device, zoom, undo, and multi-align buttons do not bubble to the preview background and trigger deselect.
- **undo/redo**: buttons near the width switcher + keyboard `Ctrl/⌘+Z` / `Ctrl/⌘+Shift+Z`; same history functions, no reload, no clearing current values.
- **Project root key replaces every template root special-case**: if the template's demo root key (`card`) is replaced with `homeScreen`, `modalRoot`, etc., update every matching special-case: initial `cur`, boot-time `pick(...)`, card/root transform var handling, Size/Radius category eligibility, alignment parent logic, and any boot-time stage width. A stale demo key silently breaks reset/transform/selection behavior.
- **Fixed / production-positioned elements become stage-relative**: production `position:fixed` or global overlays (bottom navs, modals, docks) must be reset inside `#card-stage` with scoped project CSS. Use stage-relative positioning, e.g. `position:absolute; left:50%; transform:translateX(-50%); width:min(MAX_WIDTH, calc(100% - SIDE_PADDING));`, not a fixed `left + width` pair that only works at one stage size. The element must stay centered and fully visible at every device width.
- **SVG/icon visibility protection in the preview**: when an icon's visibility depends on the real stylesheet, make sure the staged preview preserves the same rendered surface. Inline SVGs that rely on CSS should have scoped fallback rules for `width`/`height` plus the correct inactive/active `stroke`/`fill` behavior (`stroke:currentColor` for line icons, `fill:currentColor` for active filled icons, and any cutouts restored). Do not let reset/undo erase an SVG's original inline `width`/`height` style.
- **SVG source migration must update Layers/selectors too**: if the real project changes an icon from CSS pseudo/background/mask or a placeholder `<span>` into inline SVG, the control panel must immediately mirror that new rendered DOM. Do not keep old pseudo-layer names, hidden span targets, `PSEUDO_BADGES`, `NON_PICKABLE`, or `SEL` selectors that point at the obsolete implementation. Expose the real `<svg>` as its own atomic child in `LAYERS`, make it pickable, route `SEL`/`OUT_SEL` to the SVG element, and use SVG paint controls (`fill`/`stroke`/`stroke-width`) so color edits actually affect the visible icon.
- **SVG paint writes style and attributes**: SVG descendants may carry hard-coded `fill` / `stroke` attributes, so live paint edits must set both `node.style.setProperty(prop, value, 'important')` and `node.setAttribute(prop, value)` on the SVG root and paintable descendants. Reset/undo must cache and restore original child inline styles plus original `fill`/`stroke` attributes.
- **Low-interference selection hint**: selected keys (single or multi-select) may add a low-noise visual fallback on the target element itself, e.g. `.tw-pick { outline:2px solid rgba(26,115,232,.9); outline-offset:-2px; }`, but **skip this real-element outline for SVG targets or any target with an active CSS `filter`**. CSS filters include outlines in the filtered rendering, so an SVG drop-shadow can make the selection outline look shadowed or duplicated. Always draw a separate selected-key overlay (`#tw-selected` with `.tw-sel-frame` children) from `rectFor(key)` for every selected key; this is required because SVGs, masked icons, inline spans, and tiny visual nodes may not render CSS `outline` reliably. `#tw-tbox` is the transform handle / group bounds, not the only visible selection indication; if its rect is tiny, clipped, or temporarily unavailable, the selected element must still be visible. **Never mix the selection box into `box-shadow` or `filter`**, the shadow/glass preview must stay pure.
- **Click-to-drill + click-background-to-deselect (Figma-style selection, replacing `closest('[data-pick]')`)**: `closest('[data-pick]')` selection has two blind spots — (1) synthetic layers (e.g. a text Hug, or a bare-text span that doesn't exist yet) have no DOM node to hit; (2) when a child fills its parent (e.g. an icon filling a button), you can only ever hit the innermost, never the parent; the user is forced to use the layer tree, which is unintuitive. Fix: **use rect-containment + area sort instead**. `clickStackAt(x,y)` scans every `SEL` key, gets its rect via `rectFor(key)` (text role → hug rect), collects all keys whose rect contains the point, and sorts by area ascending (smallest = deepest). `pickAt(x,y)`: if the current `cur` is in the stack → select the next one (one level outward, cycling); if not → select `stack[0]` (deepest). Result: clicking on the glyphs selects the text layer first, click again → the box, again → the card… drilling outward; clicking the input's empty padding (outside the text hug rect) selects the box directly; clicking a button's icon → icon → button → … all reachable by click alone, no layer tree needed. `shift+click` adds to the selection (`stack[0]`).
- **Click empty canvas to deselect**: clicking the canvas outside the component (not in `#card-stage`, not on the transform box `#tw-tbox`) should `deselectAll()`: `multi={}; cur=null;` then `refreshSelectionUI()` + `buildPanel()`. Make the "no selection" state safe: start `buildPanel` with `if(!key){ show a hint, return; }` (otherwise `getComputedStyle(elFor(null))` throws); have `refreshSelectionUI`'s `cur-name` handle `cur===null`; `updateTBox` already hides the transform box and parent frame when `multiKeys()` is empty. The `#card-stage` click handler uses capture + `stopPropagation`, so clicks inside the component don't bubble to the background; only clicks on the outer `#tw-left` gray area trigger deselect.
- **Nested frames + text Hug (selection frame when a single element is split into box/text, Figma-style)**: when "box" and "text" are two logical layers of the *same* element (see the single-`<input>` split under "Atomic decomposition"), the text layer's selection frame **must not** use `el.getBoundingClientRect()` directly — that makes it identical to the box layer. The text has no DOM node of its own, so **measure the actual rendered text** to hug it (Figma's "Hug"): use an off-stage mirror `<span>` (`position:fixed` off-screen, `white-space:pre`, copy `font-family/size/weight/style/letter-spacing/line-height`), set it to the currently displayed text (`value||placeholder` for `input`, else `textContent`), read `offsetWidth/Height`, then compute the text rect inside the element from its padding/border, `text-align` (left/center/right) and vertical centering; multiply sizes by `zoom` (`getComputedStyle` returns unzoomed CSS px, `getBoundingClientRect` returns zoomed screen px). Route the transform box through one `rectFor(key)`: text role → measured rect, otherwise → element rect. Companions: when a text layer is selected, outline its **parent** with a dashed overlay (`#tw-parentbox`) as a Figma nesting cue; on hover of any element that has a text child, show both a solid box overlay and a dashed text overlay (`#tw-hover` / `#tw-htext`, both `pointer-events:none`, never polluting box-shadow). Since the Hug frame's width is content-driven, **hide the edge handles** (keep the four corners + rotate); **corner-drag changes `font-size`** (not `_scl`, which would scale the whole element), edges disabled. Note: box and text share one element, so move/rotate/scale transforms can't be independent between them (one element); the only truly independent surface of the text layer is typography — which is why the Hug frame's corners should map to `font-size`.

#### Atomic decomposition (Layers must be split until it can't be split further)

`LAYERS` (template `⟦PROJECT-SPECIFIC #7⟧`) must **decompose every element to atomic level based on the real code**, not just list big blocks:

- First read "the real JSX/JS/HTML that generates that DOM" to enumerate all atomic elements, then build the tree.
- Split mixed elements for separate selection: icon vs text (e.g. badge = clock SVG + time text, location = pin SVG + place text), button vs its icon (menu button + 3-dot SVG), button frame vs the text inside it.
- **Do not fake-split CSS-generated pseudo/composite paint**: if a visual part is a single CSS pseudo-element (`::before` / `::after`) or one `background-image`/PNG/mask composite, treat it as **one atomic layer** unless you deliberately build a synthetic DOM replacement and can also write the equivalent change back to real source. Do not list fake children such as "slot / badge / icon" when all of them are painted by one pseudo-element; their X/Y/size/color controls will be bound together or ignored. For these composite layers, expose only properties that can actually be written to the source selector (for example pseudo transform/scale, `background-size` for icon size, background-color, border, box-shadow). If the icon is a PNG background, do not expose SVG-only `fill`/`stroke` controls. When exposing PNG icon size, write a **single-value** `background-size` such as `70%` so the browser preserves the image aspect ratio; avoid two-axis values like `70% 70%`, and keep the range bounded so the image does not crop against its circular/rounded container.
- Each item of a repeated list is its own element (e.g. tag 1 / tag 2 / tag 3, via `:nth-of-type(n)`), and each tag further splits into emoji (`.theme-emoji`) and text (`.theme-label-text`).
- Pure grouping containers (no adjustable styles of their own) become keyless nodes used as group titles.
- **Bare text nodes** (text placed directly in a container in the real code, with no element of its own, e.g. badge time, coord text): wrap them in a classed `<span>` with `data-pick` in the preview DOM so they can be selected/styled individually; this span doesn't exist in the real app, so when applying `[UI Tweaker Confirm]` you must note in the output "add the same span in the real template/JS (the function that generates the DOM)", or the style won't attach.
- **A single element can also split into two logical layers (Box vs Text)**: there are two kinds of split — (a) genuinely two DOM nodes (e.g. a star button = `<button>` + `<svg>`, a badge = SVG + text) are already separate elements, just give each a key; (b) **a single element** (most typically one `<input>`, where the frame and the text are the same DOM node) often still needs to be split in the layer tree into a "Box" layer and a "Text" layer so the user can tune them separately. Key insight: box properties (`width`/`height`/`padding`/`border-radius`/`background`/`border`/`box-shadow`) and text properties (`font-size`/`font-weight`/`font-family`/`color`/`letter-spacing`/`line-height`) on the same element are **disjoint CSS properties to begin with**, so the "they're bound together" feeling is purely a category-presentation issue, not a technical constraint. How: keep a `ROLES` map tagging each key as `'box'`/`'text'`/`'full'`, and have `buildPanel` filter categories by role — `showText=(role!=='box')` wraps "① Typography"; `showBox=(role!=='text')` wraps the other seven (Spacing/Size/Radius/Position/Shadow/Frosted glass/Color). The box key and the text key **share the same SEL (same element)** but each writes only its own property set, so `applyEl` never conflicts. In the layer tree hang the text layer as a child of the box layer (e.g. "Name box ▸ Text"); clicking the element in the preview selects the box layer (`data-pick="nameField"`), the text layer is selected from its layer-tree row. On confirm, emit two blocks (a box block with frame props, a text block with typography props), noting the selector is identical and only the property grouping differs.
- Example (an info card, split into ~24 atomic layers): whole card ▸ image area ﹙image, time badge ﹙clock icon, time text﹚﹚▸ info area ﹙menu button ﹙3-dot icon﹚, title, location ﹙pin icon, place text﹚, tags area ﹙tag 1/2/3, each ﹙emoji, text﹚﹚, bottom button frame ﹙text inside﹚﹚.

#### Preview DOM must mirror the real code (prevent inventing styles/structure · mandatory check before opening the panel)

The panel's preview DOM (template `⟦PROJECT-SPECIFIC #4⟧`) may only "reproduce" the real code, never "invent" it. To avoid adding attributes/styles/structure the real code doesn't have (e.g. a wrongly-added `style="flex:none"`), pass this gate every time after generating, before opening the panel:

- **Iron rule**: every element's tag, class, id, attributes, child structure, and text nodes in the preview DOM must map to a source in "the real file that generates that DOM" (file:line). Anything that can't be mapped, except the allowlist, is deleted.
- **The only allowlist for deviating from the real code** (annotate each one inline in the preview DOM):
  1. `data-pick` (selection marker);
  2. an `id` added for selection (when the real code lacks one but selection needs it);
  3. a synthetic `<span>` for bare text nodes (see "Atomic decomposition"; note it's not in the real code and must be added to the app on confirm);
  4. **necessary visibility override**: elements that default to `display:none` / appear only on interaction (e.g. a menu button) are overridden to visible just for previewing — the annotation must state "the real default value + why it's overridden".
- **Explicitly forbidden**: using invented inline `style="…"` (e.g. `flex:none`, `width:…`, `margin:…`) to "patch" layout or styling. All styling is decided by the linked real stylesheet; if you think you need an inline style, go back and check the real code — you almost never should (except allowlist #4). Placeholder images/text are for display only and must not change structure or attributes.
- **Mandatory self-check after generating (three steps, do them before opening the panel)**:
  1. For every `class=`/`id=` in the preview DOM, `grep` the real source to confirm it exists; not found → invented → delete or fix.
  2. Search the preview DOM for all `style="`, check each against allowlist #4; remove any inline style that doesn't match (if the real code doesn't have it, it shouldn't be there).
  3. Scan every icon position in the preview DOM: each must be a real icon element (inline `<svg>` / class span + CSS background or mask / icon font), never an emoji / unicode / plain-text stand-in; if you find a stand-in, go back to the source and copy the real icon markup.
  4. Run an HTML validity scan on the generated preview fragment: `data-pick` belongs on opening tags only (never `</span data-pick="...">`), every closing tag must be complete (no truncated `</div`), and the browser DOM tree must keep `#card-stage`, the root component, and any staged fixed elements nested as intended. Invalid preview HTML can make scoped reset CSS fail even when the CSS is correct.
  5. On a real browser, spot-check 2–3 elements via `getComputedStyle` (size / `flex` / color) and confirm they match the real component's rendering.
- **Multi-width browser verification is mandatory for staged layouts**: after generating the panel, test narrow / default / wide stage widths in a real browser or headless browser. Verify: no horizontal cropping, responsive children resize with the stage, fixed/dock elements are centered and complete, SVG icons render, and reset/undo returns to the original visual state. A key-consistency script alone is not enough.
- Keep a "source map" (in mind or in output): each pick key → real file:line (e.g. `loc → file-that-generates-the-DOM:line`, `menu → template-file:line`). Elements that can't be mapped to a source don't go into the panel.

#### Control panel technical spec

- Each input needs a stable, unique id, e.g. `id="i-{rowId}"`; value display can use `id="v-{rowId}"`.
- All input updates go through one handler, e.g. `ch(id, value, pos)` or an equivalent central handler.
- Preview updates concentrate in `applyDOM()` (or equivalent), applying styles by the current selected element key (e.g. `cur`).
- Shadows write the real `box-shadow` / `filter` values only in the style-apply function, never mixing in selection state.
- Controls must restore from the current CSS initial values; if a property is missing, use an explicit default and keep the field in the output.
- When adding controls, never reset values the user is in the middle of tuning; where possible, use localStorage or an explicit confirm output to preserve state.
- **undo/redo/reset must restore TO the original inline style, never wipe it**: faithful reproduction often leaves preview elements carrying **real inline styles** (e.g. `.batch-loc-row` ships `display:flex;gap:4px;…` from a JS template). `restore()` must NOT do `el.style.cssText=''` — that erases those real inline styles too, so after undo/reset the layout breaks (only a page reload fixes it). Correct pattern: up front (in init, before applying any override) cache each keyed element's `getAttribute('style')` into `ORIG_STYLE` via `captureOrigStyle()`, then `restore()` does `el.style.cssText = ORIG_STYLE[k] || ''` before re-applying the state overrides. Same "cache the original, restore to the original" pattern as `ORIG_TEXT` for text content.
- **Every live apply must be reversible**: before applying the current state for a key, restore that key's edit target to its cached original style, then reapply all current state for keys that share the same edit target. This prevents stale properties from surviving after a user deletes an effect, removes a gradient, toggles no-paint, or changes SVG paint. Do not solve stale effects by clearing arbitrary properties globally; restore the original target first, then replay the active state.
- **SVG paint undo/reset must restore child inline styles too**: if `applySvgPaint()` writes `fill`/`stroke` onto SVG child nodes (`path`, `circle`, `rect`, `line`, etc.), cache each child node's original `style` during init and restore those child styles inside `restoreBaseStyle()`. Restoring only the keyed `<svg>` element leaves child inline paint overrides behind, so undo/reset appears broken.

The eight control categories:

① Typography
   font-size (px), font-weight (100–900), line-height (em), letter-spacing (em), font-family (select), color

② Spacing
   padding T/B/L/R (px), margin T/B/L/R (px), child gap (gap)

③ Size
   width (px/%), height (px/auto), max-width, aspect-ratio, proportional Scale % control (writes the same `_scl` transform scale state as corner-drag / transform-box scaling, so users can enlarge or shrink an element with one field instead of changing W and H separately)

④ Radius
   overall radius (px/%/cqw), individual corners (TL/TR/BL/BR), toggle overall/individual mode

⑤ Position offset
   X offset (translateX/left/right), Y offset (translateY/top/bottom), rotation (deg), scale, snap + guides

⑥ Shadow
   X offset, Y offset, Blur, Spread (all px/cqw), opacity (0–1), split foreground/background/occlusion shadows when needed

⑥b Effects
   Stackable effect layers with an eye toggle, type selector, and delete button: Drop shadow, Inner shadow, Layer blur, Background blur, Glass. Use the panel's own compact line icons, not copied app-specific icons. Drop/inner write `box-shadow`; layer blur writes `filter`; background blur/glass write `backdrop-filter` / `-webkit-backdrop-filter`; glass may also write a translucent background color. **SVG exception**: for inline SVG targets, Drop shadow must write `filter: drop-shadow(...)` instead of `box-shadow`, and Background blur / Glass must not add a background color or backdrop box to the SVG element; otherwise the icon gains an unwanted square background.
   Deleting an effect must remove the previewed CSS immediately by restoring the target's original style and replaying remaining active state; never leave old `box-shadow`, `filter`, `backdrop-filter`, or glass background values behind.

⑦ Frosted glass
   blur strength (blur px), white opacity (0–1), top gradient delta (%),
   hover shadow (px), top highlight edge (0–1), inner glow (0–1)

⑧ Color
   background, text, border color (all support rgba), border width (px)
   - **No-paint toggle is a standard paint control**: every paint-like color row (background, border, fill, stroke, pseudo badge fill/border, mask icon color where transparent is valid) must include the small square-with-diagonal no-paint button. It writes `none` for SVG fill/stroke, `background:none` or transparent for background-like surfaces, and transparent for border/text-like surfaces as appropriate. The button must stay available after panel rebuilds and must be undo/redo/reset-safe.
   - **Linear gradient row**: non-SVG color surfaces may include a compact linear-gradient control with a live gradient bar, two draggable color stops on the bar, and opacity controls. Dragging a stop changes the stop percentage directly on the bar; color/alpha fields update the same state. It writes `background: linear-gradient(...)`, is history-aware, and appears in confirm output as `background-gradient`. Initial stop colors must be parsed from the selected element's real `getComputedStyle(...).backgroundImage/background` when a gradient already exists; do not initialize the row from generic placeholder colors that differ from the visible component.
   - **SVG icon exception**: when the selected element is an SVG (e.g. a line-style pin icon), the Color category should expose **fill `fill` / stroke `stroke`** (separately, each with a no-paint toggle that writes `none`) plus **`stroke-width`**, instead of background/text/border — an SVG's tunable color surface is `fill`/`stroke`, not background/border. The no-paint toggle should be an icon button (small outlined square with a diagonal slash), not a text button labeled "None". Read initial values via `getComputedStyle` as usual (`fill:none` → default the no-paint toggle on and disable/dim the swatch; `stroke` reads back the resolved currentColor rgb). Also preserve state values stored as `#rrggbb` when the panel rebuilds. Apply via `el.style.setProperty('fill'|'stroke', val, 'important')`; emit `fill:` / `stroke:` in the confirm output. Detect with `el.tagName.toLowerCase()==='svg'`.
   - **Default SVG Icon controls**: for a selected inline SVG, add a dedicated "SVG Icon" category above Color with: proportional icon size (writes width and height together), padding, stroke thickness, scale %, flip/mirror buttons, rotation presets, and direction presets. These controls must be history-aware, reset/undo-safe, and included in confirm output. Do not expose trace/expanded-outline controls unless the panel actually performs vector path outlining or the project has a real source API for it; trace width is not the same as CSS `stroke-width`.

Suggested font-family select options:
- `inherit`
- `system-ui`
- `-apple-system`
- `Arial`
- `Georgia`
- `Noto Sans TC`
- `PingFang TC`

#### Text-content editing (default)

Every generated control panel must support editing the "display text" itself (system copy, button labels, placeholders, titles…) by adding a "Text content" input at the **top** of the "① Typography" category for meaningful text-bearing elements. Do this by default; users should not need to ask for a special text-edit mode.

- **Read/write the right target per element type**: plain elements edit `textContent`; `<input>` edits `placeholder` (the hint shown to the user; edit `value` only if that's what's displayed); a `<select>` shows the **currently-selected `<option>`**, not the first one — so usually use `mode:'optionSel'` (read/write `el.options[el.selectedIndex]`), not `option0` (the first option, which is often a "Please choose…" placeholder). Keep a small map noting which mode each editable key uses.
- **Default detection + override map**: auto-detect editable text for plain leaf text elements, `<input>`, `<textarea>`, and `<select>`. Keep `TEXT_TARGETS` available for overrides (`false` disables a key; `{mode:'text'|'input'|'value'|'optionSel'}` forces a mode) when the auto-detection would pick the wrong source.
- **A `<select>` preview must show the "after-selection" state + data-driven option text**: don't fill a preview `<select>` with just a placeholder or invented options. If what the user cares about is "how it looks once an item is chosen," make one **real option carry `selected`** and point the text target at it via `optionSel`. More important: when the option text is **composed from data/code** (e.g. a theme = `emoji + space + label`, built by `getThemeParts(t).value` / `composeThemeLabelRaw`, emoji from `THEME_EMOJI_MAP`), the preview options must reproduce that exact format (including emoji) — never hand-invent an emoji-less stand-in set. This is an extension of "preview must be faithful to the real code." The confirm output should note on that selector that the option text is **data, not a literal string** — what to edit is the theme data / emoji map, not some literal.
- **Dual-slot "before/after typing" editing for `<input>` (two slots)**: users often want to see/tune both the **before-typing hint (placeholder)** and the **after-typing actual value (value)** — text *and* font for each state. Set the input's text mode to `mode:'input'`, splitting into two slots: `_ph` (before / placeholder) and `_val` (after / value), each with its own "Text content" row (label them clearly: "Before-typing text (placeholder)" / "After-typing text (actual value)"). Other modes keep the single slot `_text`. `ORIG_TEXT[key]` changes from a string to an object `{_ph,_val}`; `slotsFor(key)` returns the relevant slot array.
- **If text hiding is part of the requested tweak, make each slot hideable**: add an eye toggle next to each text row, storing `_phHidden`/`_valHidden` flags; when hidden, render that slot as an empty string but **keep the original value in state/ORIG_TEXT**, so toggling it back restores it.
- **placeholder vs value typography must be independent (KEY: go through a `::placeholder` rule, not `el.style`)**: an `<input>`'s placeholder and its typed value **share one font**, so "placeholder font-size 30px, value font-size 16px" **cannot be done via `el.style`** — `el.style.fontSize` changes both. The fix: store the placeholder's typography props with a `'ph:'` prefix (e.g. `state[key]['ph:font-size']='30px'`), then `applyPlaceholderStyle(key)` collects all `ph:` props and injects them into a per-(state,key) `<style id="tw-ph-{state}-{key}">` whose content is `SEL[key]+'::placeholder{…!important}'`. The element-property loop in `applyEl` must **skip `_`-prefixed and `ph:`-prefixed keys** so placeholder typography isn't wrongly written onto the element itself. Add a slot toggle ("After-typing font / Before-typing font", var `TXSLOT`) that decides whether the "① Typography" numboxes edit the element's own props (`pfx=''`) or the placeholder's props (`pfx='ph:'`); when reading initial values for `ph:` props, `cssVal` must read them back from `getComputedStyle(el,'::placeholder')`. `restore()` must clear the content of every `style[id^="tw-ph-"]`.
- **Live-update the preview**: write to the preview DOM on keystroke; push one history entry only on the completion event (`change`).
- **Bind text rows to their key, not the global current selection**: each text input row must store its owning key (e.g. `data-key`) and update `state[textKey]` / `applyText(textKey)`. Do not rely on global `cur`, because multi-select, toolbar interactions, or panel rebuilds can change `cur` while the user is editing. Text rows use `.tw-text-edit`, not `.tw-row`, so `bindRows()` must bind `.tw-text-edit input` separately; scanning only `.tw-row` leaves the text field visibly editable but disconnected from the preview DOM.
- **undo/redo/reset must restore it**: cache each editable element's original text up front (`ORIG_TEXT`); `restore()` resets text to the original before re-applying overrides, so nothing lingers.
- **Confirm output**: emit `- text-content: <new string>`, plus a NOTE making clear this is **display copy, not styling**, and that the real string source must be edited (HTML and/or JS templates, e.g. literal strings inside a template literal). Bare text nodes on buttons/labels follow the "wrap bare text in a `<span>`" rule, and remind that the real template needs the same span added. For a dual-slot `<input>`, emit `- placeholder: …` (before) and `- value (typed text): …` (after) separately; the placeholder's typography props (the `ph:` group) must be emitted as their own `[OUT_SEL::placeholder]` block (with the `ph:` prefix stripped), separate from the element's own `[OUT_SEL]` block, noting that the real code needs a `::placeholder` rule for them to attach.
- Only expose this field for meaningful system copy; emoji / icon-only buttons don't need it.

---

### Step 3: Confirm button triggers the AI update

The "Confirm, update code" button at the bottom of the panel uses `sendPrompt()` to send all current values back to the AI assistant in a structured format.

If the component has multiple children, prefer the per-element format:

```
[UI Tweaker Confirm]
Component: Home folder
Source: /path/to/source.css

[folder-wrapper]
  - width %: 114
  - left %: -7.5

[folder-photo]
  - white overlay alpha: 0.18
  - box-shadow blur cqw: 6

[folder-glass-panel]
  - backdrop-filter blur px: 4
  - filter photo shadow alpha: 0.34

Please update the matching code directly
```

For a simple single element, the flat format also works:

```
[UI Tweaker Confirm]
Component: FolderCard's .glass-card
Update these values:
- width: 160px
- border-radius: 20px
- backdrop-filter: blur(16px) saturate(1.4)
- background: linear-gradient(160deg, rgba(255,255,255,0.72), rgba(255,255,255,0.55))
- border: 1px solid rgba(255,255,255,0.85)
- box-shadow: inset 0 1px 0 rgba(255,255,255,0.9), 0 18px 36px rgba(0,0,0,0.18)
Please update the matching code directly
```

After the AI assistant receives a message starting with `[UI Tweaker Confirm]` (or its localized form, e.g. the Chinese `[UI Tweaker 確認]`):
1. Parse all values.
2. Find the matching CSS class or inline style.
3. Output the full updated code block.
4. List the changed lines.
5. After updating, run whatever verification is available — e.g. if the project has `npm run build`, run the build; check the actually-applied values via browser/computed style when needed.

Design system scope confirmation:
- Before applying changes to reusable styles, detect whether the selected value comes from a design token, shared utility class, component-level selector, or local override.
- If the style appears reusable, ask the user to choose the update scope before writing:
  - **Global**: update the design system token/shared class so all matching usages change.
  - **Component**: update this reusable component family only.
  - **Local**: update only the current page/selected instance with a scoped override.
- Default to **Local** when the scope is unclear. Do not describe Local as "detaching" from the design system; describe it as a scoped override.
- For **Global** changes, search usages first and summarize likely impacted files/selectors before editing.

Structured-update rules:
- If the confirmed values exceed a handful of fields, or span multiple selectors / children, use a structured parse-and-update flow — don't hand-copy values line by line.
- Suggested flow: parse the confirm text → check `missing / unknown / changed` → dry-run listing the selectors/properties to be changed → confirm `skipped=0` or clearly handle the skipped reasons → then write to file.
- Use an in-project script, a temporary updater, an AST/CSS parser, or any other re-runnable structured method; no fixed filename required.
- Dry-run output should include counts, e.g. `parsed`, `changes`, `skipped`, so you can confirm completeness by number.
- After writing, dry-run or parse once more to confirm the baseline values match the file state.

commit / push boundary:
- When the user pastes `[UI Tweaker Confirm]`, or clearly says "it's tuned / apply this set / just update", treat it as a confirmed UI change; after applying and verifying, commit/push directly.
- When the user explicitly asks to `commit`, `push`, "submit", "push it up", also commit/push after verification succeeds.
- While still exploring, asking, opening the panel, or before the user has sent confirmed values, only modify the panel/prep tooling; don't auto commit/push.
- Before committing, check the working tree; only commit files related to this UI change, avoiding unrelated changes.

---

### Step 4: Output the updated code

Format:
✅ Updated [component name] — [class name]

[full code block]

Changes:
- border-radius: 16px → 20px
- backdrop-filter: blur(12px) → blur(16px)
- box-shadow strengthened the hover shadow

---

## Notes

- The left preview should restore the user's real styles as closely as possible; no fake data.
- The preview DOM may only reproduce the real code, never invent: before opening the panel, `grep` every `class`/`id` to confirm it exists, and remove non-allowlisted inline `style` (see "Preview DOM must mirror the real code").
- Control-panel initial values come from the user's code, not defaults.
- If a property is undefined in the code, set the initial value to the CSS default.
- Children are clickable to switch controls; on click clear old selection state, keep only the current target.
- Selection hints must not pollute `box-shadow`, `filter`, frosted glass, or shadow previews.
- Position offset should support snap near 0 and a brief alignment guide.
- Frosted glass category: collapsed by default if the component has no backdrop-filter.
- Multi-state components (loading / detecting / done / empty…): confirm the target state first and fully reproduce all of that state's fields, not a simplified subset.
- By default, offer a "Text content" input at the top of "① Typography" for meaningful text-bearing elements so the user can edit display copy directly; emit it as `text-content:` / `placeholder:` / `value (typed text):` on confirm and note that the real string source must be edited.
- Before the confirm button sends, the panel shows a summary of the CSS about to update for confirmation.
- For multi-field or multi-selector updates, use a structured parse/dry-run/write flow; don't hand-copy values.
- Verify after updating; prefer running the build if there's a build command, then do browser/computed-style checks when visuals or browser behavior matter.
- When you receive already-tuned values like `[UI Tweaker Confirm]`, commit/push after a successful update + verification; don't auto-commit during the exploration phase.
- After receiving a `[UI Tweaker Confirm]` message, output the code directly without asking again, except when a reusable design-system style is detected and the update scope is not specified.
