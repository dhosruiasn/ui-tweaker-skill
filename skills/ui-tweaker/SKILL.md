---
name: ui-tweaker
description: Triggers when the user says things like "help me tweak [component]", "I want to fine-tune [style]", "adjust the [property] of this element". Generates a live preview + full control panel; after the user adjusts and confirms, the panel produces structured values and selectors for the AI to apply to source files.
---

# UI Tweaker

Let users fine-tune UI directly with an AI coding assistant through sliders/number boxes; after they confirm, the panel copies structured values for the AI to update code precisely — no back-and-forth describing numbers in words.

Core principle: every adjustment is a structured, round-trippable number — never just a "feeling".

> 繁體中文版規格見 [`SKILL.zh-TW.md`](./SKILL.zh-TW.md)（兩份內容等價，行為一致）。

## Token-efficiency hard rules (read first)

The panel template is ~177KB. To keep token cost low, these rules override any other phrasing in this skill or its references:

1. **Never Read the whole template into context.** Copy it with Bash `cp`. Never re-emit its content with Write.
2. **Edit placeholders surgically**: Grep for `⟦PROJECT-SPECIFIC` to get marker line numbers, Read only those small regions, and use Edit to replace each spot. Everything outside the markers stays byte-identical.
3. **Iterate with Edit, never full rewrites**: when fixing or re-syncing a generated `ui-tweaker-*.html`, edit the specific region; never Write the whole file again.
4. **Verify with Grep / targeted Read**, not by reading the generated panel back in full.
5. **Load references on demand**: read a file under `references/` only when its phase applies (see pointers below); don't read all of them up front.

## Distribution and updates

This repo is a Claude Code plugin marketplace (`.claude-plugin/`) plus the skill folder (`skills/ui-tweaker/`); install/update via the GitHub-backed marketplace. Other AI tools can treat `skills/ui-tweaker/` as a portable instruction folder synced via `git pull`/submodule/symlink; don't describe this as live push updates unless the host AI supports that.

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
4. **Confirm the target state**: if the component has multiple states (loading / detecting / done / empty / expanded vs. collapsed / hover…), first confirm which screen state the user wants to tweak, then faithfully reproduce ALL fields of that state (never a simplified subset). Decide the state from the real JS that generates the DOM (e.g. the `status === 'detecting'` branch) and copy that branch's full markup. A user screenshot is an aid for verification, not a prerequisite.

---

### Step 2: Generate the "preview" + "control panel" together

#### Always start from the fixed template (highest priority · never re-draw the UI)

This skill ships a fixed-interface template `template/panel-template.html` (relative to this SKILL.md's folder). **Every time you open a control panel, copy this template as the base.** Do not hand-write the UI from scratch and do not change the template's layout/CSS/builder, so that anyone, on any project, opens a byte-for-byte identical interface.

Flow:
1. **Copy the template with Bash `cp`** to a browser-reachable location in the user's project (e.g. project root, filename like `ui-tweaker-<component>.html`). Do NOT Read the template in full and do NOT re-emit it with Write (see token-efficiency rules).
2. Grep the copied file for `⟦PROJECT-SPECIFIC` and replace **only** those marked spots, using targeted Read + Edit (everything else stays untouched):
   - **#1** `<title>`
   - **#2** font `<link>` (the project's real fonts)
   - **#3** stylesheet `<link href>` (link the project's real CSS; never hand-copy values into `<style>`)
   - **#4** the preview DOM inside `#card-stage` (paste the real component markup; add `data-pick` to every atomic element)
   - **#5** `SEL` map (pick key → selector used inside the widget); plus optional **#5b** `VISUAL_RECTS`, **#5c** `EDIT_TARGETS`, **#5d** `TEXT_TARGETS`, **#5e** `NON_PICKABLE`
   - **#6** `OUT_SEL` map (pick key → CSS selector name used in the confirm output)
   - **#7** `LAYERS` tree (decompose the real DOM to atomic level)
   - **#8** the confirm-output "component:" / "source:" strings
   - **#9 stage reset**: the rule that resets your component root from hidden/positioned to visible/static inside `#card-stage` (target your real root selector; keep the `--tw-card-transform` var; reset production `z-index` too). Only needed when your real component is hidden/absolutely positioned in production.
3. After replacing, every keyed `LAYERS` node must have a matching entry in `SEL` / `OUT_SEL` / the preview DOM's `data-pick`. Cross-check key consistency with a script and validate the JS with `node --check`.
4. The template already ships the fixed UI (Layers tree, three-section panel, nine categories, number boxes, scrub, undo/redo, transform box, rulers, context menu…). **Don't rebuild or restyle these.**

> Only fall back to "hand-write per spec" if the template file is missing/deleted; in that case read `references/template-behaviors.md` for the full interface spec. As long as the template exists, it is the source of truth.

The widget is one HTML page in three columns: left Pages/Layers tree (Figma-style), middle live preview (restored by linking the real stylesheet), right control panel (nine collapsible categories). Expand the category most relevant to the current element.

#### Required reading before filling placeholders

- **`references/fidelity-and-dom.md`** — read BEFORE filling #4/#5/#6/#7. Covers: real-style fidelity (link real CSS, real DOM/classes, `getComputedStyle` initial values, apply with `!important`, no emoji icon stand-ins), dynamic JS template reconstruction, atomic decomposition of `LAYERS` (icon vs text splits, bare-text `<span>` wrapping, Box/Text role split), and the mandatory mirror-gate self-check (grep every class/id against real source, no invented inline styles, HTML validity, multi-width browser verification).
- **`references/template-behaviors.md`** — read only the parts you need when: adapting per-project special cases (replacing the demo root key `card`, stage reset & z-index, `VISUAL_RECTS`/`EDIT_TARGETS`, SVG icon source migration, default stage width), debugging template behavior, or hand-writing the fallback. The template already implements everything in it.
- **Zero-jargon rule**: never require the user to say "restore DOM", "mock", or "atomic level" — do the structural conversion in the background and show only an intuitive clickable preview.

---

### Step 3: Confirm button triggers the AI update

The "Confirm, update code" button at the bottom of the panel uses `sendPrompt()` to send all current values back to the AI assistant in a structured format, e.g.:

```
[UI Tweaker Confirm]
Component: Home folder
Source: /path/to/source.css

[folder-wrapper]
  - width %: 114
  - left %: -7.5

[folder-glass-panel]
  - backdrop-filter blur px: 4

Please update the matching code directly
```

(A flat single-element format without per-key blocks is also valid.)

After receiving a message starting with `[UI Tweaker Confirm]` (or its localized form, e.g. `[UI Tweaker 確認]`), **read `references/confirm-apply.md`** and follow it: parse values → design-system scope confirmation (Global/Component/Local, default Local) → structured parse/dry-run/write flow for multi-field updates → verify (build + computed-style checks) → commit/push boundary rules.

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

- The preview restores the user's real styles; no fake data, no invented DOM (pass the mirror-gate in `references/fidelity-and-dom.md` before opening the panel).
- Control-panel initial values come from the user's code (via `getComputedStyle`), not defaults; undefined properties fall back to the CSS default.
- Selection hints must never pollute `box-shadow`, `filter`, frosted glass, or shadow previews.
- Multi-state components: confirm the target state first and fully reproduce all of that state's fields.
- Text-bearing elements get a "Text content" input at the top of "① Typography" by default; emit `text-content:` / `placeholder:` / `value (typed text):` on confirm and note that the real string source must be edited.
- A generated `ui-tweaker-*.html` is a snapshot: if the user says the real code changed, re-read the real source and re-sync the affected placeholder regions with Edit (preview DOM, `SEL`/`OUT_SEL`/`LAYERS`, stage reset), then re-verify with `getComputedStyle` in a real browser.
- After a `[UI Tweaker Confirm]` message, apply directly without asking again, except when a reusable design-system style is detected and the update scope is unspecified (see `references/confirm-apply.md`).
