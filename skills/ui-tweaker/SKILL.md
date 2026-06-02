---
name: ui-tweaker
description: Triggers when the user says things like "help me tweak [component]", "I want to fine-tune [style]", "adjust the [property] of this element". Generates a live preview + full control panel; after the user adjusts and confirms, the AI writes the values straight back into the source files.
---

# UI Tweaker

Let users fine-tune UI directly in a Claude conversation with sliders/number boxes; after they confirm, the AI updates the code automatically — no back-and-forth describing numbers in words.

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
2. **Use the real DOM structure and classes**: preview elements must use the component's true tags, classes, and nesting — no simplified custom versions. If content is injected by JS (dynamic ids), reproduce the real JS-generated markup — including icon SVGs, `<span>` wrapping, and formatted strings (e.g. `Today 14:30`, not your own `2025/06/02 14:30`). Read the JS function that generates the DOM first.
3. **Read initial values via `getComputedStyle`**: every control's initial value must be read with `getComputedStyle(realEl).getPropertyValue(prop)` after the stylesheet is confirmed loaded (`window.load` or poll until the sheet is ready). Never fill in base rules or estimates.
4. **Apply with `!important`**: live preview always applies via `el.style.setProperty(prop, val, 'important')`, otherwise the source's `!important` wins and dragging a slider won't move the screen.
5. **Handle JS-hidden children**: some elements default to `display:none` and are shown by JS only in certain states (e.g. a menu button). The preview must force them into their "when visible" state (e.g. inline `style="display:flex"`), or you'll drop elements.
6. **Detect and protect multi-value properties**: multi-layer `box-shadow`, gradient `background`, etc. can't be losslessly restored by a single-layer slider, and `getComputedStyle` can't read back a single value. When you detect layered/gradient values, lock that control or show a warning so a single-layer slider doesn't clobber the original.
7. **Confirm output contains only changed properties**: when sending `[UI Tweaker Confirm]`, output only properties the user actually changed; never include untouched ones, so you don't accidentally alter things they were already happy with.
8. **Verify on a real browser when done**: read `getComputedStyle` or screenshot to confirm the preview's parsed values match the app's actual values (e.g. compare 2–3 key properties) before handing off.

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
- Provide a "zoom" feature so users can inspect detail by zooming only the preview, not the whole page: apply CSS `zoom` to the preview stage container (`zoom` scales both rendering and scroll range; with `overflow:auto` you can scroll to detail). Provide −/percent/+ buttons (click the percent to reset to 100%) and `Ctrl/⌘+wheel` zoom. Note: `zoom` makes `getBoundingClientRect` return post-zoom coords, so any offset derived from measured rects (alignment, centering) must be divided by the current zoom factor. Zoom is view-only; never written to the user's code.
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
- Support PS-style multi-select mutual alignment: `Shift+click` (in the preview or the layer row) toggles add/remove from the multi-select set (the last one can't be cleared), `cur` stays the last-interacted primary element and the panel shows it. When ≥2 are selected, show a row of alignment buttons (left/h-center/right, top/v-center/bottom) on the same row as undo/redo; hide them when <2. Alignment is based on the bbox of the selection set, achieved via offset props (translate `_tx`/`_ty`) and recorded in history. Note: a selected element must be transform-movable (block / inline-block / flex or grid item, etc.); pure `display:inline` elements ignore transform, so first ensure it's a flex item or make it inline-block. Vertically aligning elements already flex-aligned in the same row may move them imperceptibly — that's normal.
- Alignment controls must choose the right property by the container's `display`, or applying it won't move anything:
  - **Horizontal align** on a flex/inline-flex container via `text-align` won't affect its children — use the main-axis property instead; only non-flex elements use `text-align`.
  - **Vertical align** can never be done with `text-align` alone — use the cross-axis property (flex container); for non-flex elements that need vertical centering, wrap with `display:flex; align-items:center` with minimal side effects. Besides the horizontal row (left/center/right), provide a vertical row (top/center/bottom).
  - **Axis depends on `flex-direction`**: `row` → horizontal=`justify-content`, vertical=`align-items`; `column` → swapped. Read `display` and `flex-direction` via `getComputedStyle(el)` first. Value mapping: start→`flex-start`, center→`center`, end→`flex-end`.
  - Alignment output must carry the actually-effective property (flex container → output `justify-content`/`align-items`), so confirm doesn't mistakenly change it to an ineffective `text-align`.

#### Fixed interface design (Figma/PS style · built into the template)

These are implemented in the template and must stay consistent. Reuse them for new components; don't restyle or rearrange:

- **Three-column layout**: `grid-template-columns: 230px 1fr 360px`. Left = Layers/Pages tree, middle = preview stage, right = control panel.
- **Three-section control panel** (fixes the "confirm bar floating mid-list" bug): `.tw-right` is `display:flex; flex-direction:column; height:100vh; overflow:hidden`; fixed header (`flex:none`) + scrollable `#panel` (`flex:1; min-height:0; overflow-y:auto`) + fixed bottom confirm bar (`flex:none`). **Don't use a `position:sticky` bottom confirm bar.**
- **Figma section titles**: the eight categories are collapsible blocks (`cat(title, open, [...])`); titles have no number prefix (plain text, no "① ⑥").
- **Type-able number boxes** (instead of sliders): `numBox(prop,label,min,max,step,unit,init,snap)` produces an `<input type=text inputmode=decimal>`; arrow keys ↑↓ step (Shift×10), Enter blurs, focus selects all, X/Y snap near 0.
- **Multi-column grid**: combine related fields into two/four columns with `fieldGrid(cols, fields)` (e.g. padding TRBL, shadow XYBlur).
- **Drag-to-scrub (Figma scrub)**: `attachScrub(...)`, horizontal drag on a label/icon changes the value (1px≈1 step), live preview while dragging, write one history entry on release. Cursor `ew-resize`. **No sliders.**
- **PS-style radius**: `radiusControl(key)` is a 2×2 four-corner hand-input grid (tl/tr/bl/br) + a central link button; when link is on, adjusting one corner applies the value to the other three (call `applyNum` directly — **don't dispatch events** — to avoid binding-order bugs).
- **Position offset via alignment icons**: `alignTools(key)` in three rows (box position / in-box text / vertical align), using icon buttons (not text); choose the property by container `display`/`flex-direction` (flex → `justify-content`/`align-items`; non-flex → `text-align`; vertical centering can't rely on `text-align` alone).
- **Left Layers/Pages tree**: `LAYERS` nesting mirrors the real DOM; `renderLayerNode` renders recursively, keyed nodes are selectable, Shift multi-select, group nodes collapse; `refreshSelectionUI` syncs the `.on` state and `#cur-name`.
- **Transform box (selection box = transformable)**: the selection box is a pure overlay on `#tw-left` (`#tw-tbox`, **not mixed into box-shadow**); single-select shows 8-direction handles + a top rotate handle + a live px size label. Semantics: **drag corners = proportional scale** (writes `_scl`, origin at center, ratio cancels zoom), **drag edges = change `width`/`height`** (screen delta ÷zoom → CSS px), **drag body = change `_tx`/`_ty`** (snap near 0; an un-dragged single click uses `elementFromPoint` to select the element underneath, Shift multi-select), **rotate handle = change `_rot`**. Multi-select shows only the group bbox + group move, hiding scale/rotate handles. All drags `applyEl` live + sync the matching number boxes (`i-width`/`i-height`/`i-_scl`/`i-_rot`/`i-_tx`/`i-_ty`); only on release `commit()` writes one history entry and feeds the confirm output. Position comes from `updateTBox()` (`getBoundingClientRect` converted relative to `#tw-left`; `scheduleTBox()` fires on `applyEl`/zoom/width-switch/scroll/resize). Note: pure `display:inline` elements (text, tag spans) ignore both transform and width/height, so the transform box won't move them — same limitation as the position-offset controls. The `_rot` number box range must be widened to `-180..180` for the rotate handle. **`#tw-tbox`'s `z-index` must be higher than the preview component's own `z-index`** (e.g. some card containers carry `z-index:100`, so the overlay defaults to `z-index:99999`); otherwise the overlay is covered by the component's content and only a few px of the outermost edge show — appearing as "only the outermost container shows the blue box, inner elements have none". For a new project, if the element tree has a higher `z-index`, raise the overlay further.
- **undo/redo**: buttons near the width switcher + keyboard `Ctrl/⌘+Z` / `Ctrl/⌘+Shift+Z`; same history functions, no reload, no clearing current values.
- **Low-interference selection hint**: use an inner `outline-offset:-2px` or an outer glow; **never mix the selection box into `box-shadow`**, the shadow/glass preview must stay pure.

#### Atomic decomposition (Layers must be split until it can't be split further)

`LAYERS` (template `⟦PROJECT-SPECIFIC #7⟧`) must **decompose every element to atomic level based on the real code**, not just list big blocks:

- First read "the real JSX/JS/HTML that generates that DOM" to enumerate all atomic elements, then build the tree.
- Split mixed elements for separate selection: icon vs text (e.g. badge = clock SVG + time text, location = pin SVG + place text), button vs its icon (menu button + 3-dot SVG), button frame vs the text inside it.
- Each item of a repeated list is its own element (e.g. tag 1 / tag 2 / tag 3, via `:nth-of-type(n)`), and each tag further splits into emoji (`.theme-emoji`) and text (`.theme-label-text`).
- Pure grouping containers (no adjustable styles of their own) become keyless nodes used as group titles.
- **Bare text nodes** (text placed directly in a container in the real code, with no element of its own, e.g. badge time, coord text): wrap them in a classed `<span>` with `data-pick` in the preview DOM so they can be selected/styled individually; this span doesn't exist in the real app, so when applying `[UI Tweaker Confirm]` you must note in the output "add the same span in the real template/JS (the function that generates the DOM)", or the style won't attach.
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
  3. On a real browser, spot-check 2–3 elements via `getComputedStyle` (size / `flex` / color) and confirm they match the real component's rendering.
- Keep a "source map" (in mind or in output): each pick key → real file:line (e.g. `loc → file-that-generates-the-DOM:line`, `menu → template-file:line`). Elements that can't be mapped to a source don't go into the panel.

#### Control panel technical spec

- Each input needs a stable, unique id, e.g. `id="i-{rowId}"`; value display can use `id="v-{rowId}"`.
- All input updates go through one handler, e.g. `ch(id, value, pos)` or an equivalent central handler.
- Preview updates concentrate in `applyDOM()` (or equivalent), applying styles by the current selected element key (e.g. `cur`).
- Shadows write the real `box-shadow` / `filter` values only in the style-apply function, never mixing in selection state.
- Controls must restore from the current CSS initial values; if a property is missing, use an explicit default and keep the field in the output.
- When adding controls, never reset values the user is in the middle of tuning; where possible, use localStorage or an explicit confirm output to preserve state.

The eight control categories:

① Typography
   font-size (px), font-weight (100–900), line-height (em), letter-spacing (em), font-family (select), color

② Spacing
   padding T/B/L/R (px), margin T/B/L/R (px), child gap (gap)

③ Size
   width (px/%), height (px/auto), max-width, aspect-ratio

④ Radius
   overall radius (px/%/cqw), individual corners (TL/TR/BL/BR), toggle overall/individual mode

⑤ Position offset
   X offset (translateX/left/right), Y offset (translateY/top/bottom), rotation (deg), scale, snap + guides

⑥ Shadow
   X offset, Y offset, Blur, Spread (all px/cqw), opacity (0–1), split foreground/background/occlusion shadows when needed

⑦ Frosted glass
   blur strength (blur px), white opacity (0–1), top gradient delta (%),
   hover shadow (px), top highlight edge (0–1), inner glow (0–1)

⑧ Color
   background, text, border color (all support rgba), border width (px)

Suggested font-family select options:
- `inherit`
- `system-ui`
- `-apple-system`
- `Arial`
- `Georgia`
- `Noto Sans TC`
- `PingFang TC`

---

### Step 3: Confirm button triggers the AI update

The "Confirm, update code" button at the bottom of the panel uses `sendPrompt()` to send all current values back to Claude in a structured format.

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

After Claude receives a message starting with `[UI Tweaker Confirm]` (or its localized form, e.g. the Chinese `[UI Tweaker 確認]`):
1. Parse all values.
2. Find the matching CSS class or inline style.
3. Output the full updated code block.
4. List the changed lines.
5. After updating, run whatever verification is available — e.g. if the project has `npm run build`, run the build; check the actually-applied values via browser/computed style when needed.

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
- Before the confirm button sends, the panel shows a summary of the CSS about to update for confirmation.
- For multi-field or multi-selector updates, use a structured parse/dry-run/write flow; don't hand-copy values.
- Verify after updating; prefer running the build if there's a build command, then do browser/computed-style checks when visuals or browser behavior matter.
- When you receive already-tuned values like `[UI Tweaker Confirm]`, commit/push after a successful update + verification; don't auto-commit during the exploration phase.
- After receiving a `[UI Tweaker Confirm]` message, output the code directly without asking again.
