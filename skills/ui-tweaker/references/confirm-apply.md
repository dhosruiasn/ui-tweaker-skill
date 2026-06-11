# Confirm & apply flow (ui-tweaker reference)

> Read this when handling a `[UI Tweaker Confirm]` message (Step 3/4): parsing values, design-system scope, structured updates, commit/push boundary.

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
