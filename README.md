# UI Tweaker — portable AI skill

[繁體中文版 README](./README.zh-TW.md)

Fine-tune any UI component **with an AI coding assistant** through a Figma/Photoshop-style control panel — drag sliders, scrub values, move and resize a live preview — then let the assistant write the adjusted values **straight back into your real source code**. No more round-trips of "make it a bit bigger… no, smaller… more rounded."

> **Status:** v1.0.0 · MIT licensed · packaged for Claude Code plus portable adapters for Codex, Cursor, Windsurf, and plain skill-folder installs.

<!-- TODO: drop a demo GIF here — it sells this tool better than any paragraph.
     e.g. ![demo](./examples/demo.gif) -->

---

## Why

When you ask an AI to tweak UI by describing it in words ("a touch more padding", "not that round"), you burn turns guessing numbers. UI Tweaker flips it: the assistant opens a **live control panel** rendered from *your actual stylesheet*, you adjust visually, hit **Confirm**, and the exact structured values are sent back for the assistant to apply and verify.

Core principle: **every adjustment is a structured, round-trippable number** — never a vibe.

## Features

- **Real styles, not guesses** — the preview links your project's *real* CSS and reads initial values via `getComputedStyle`, so what you see matches production (including layered `!important` overrides).
- **Three-column workspace** — Layers tree (atomic decomposition of your component) · live preview · control panel.
- **Eight control categories** — typography, spacing, size, radius, position/offset, shadow, frosted glass, color.
- **Figma/PS interactions** — type-able number boxes, drag-to-scrub, linked corner radii, icon-based alignment, undo/redo + keyboard shortcuts.
- **Transform box** — select any element to get an 8-handle box: corners scale proportionally, edges resize width/height, drag to move, rotate handle, live px label.
- **Writes back to source** — on Confirm, the assistant parses the values, locates the matching selectors, edits the real files, and verifies (e.g. runs your build / checks computed styles).

## Install

This repo is **both** a Claude Code plugin marketplace **and** a portable skill package. The portable package is described by [`skill.json`](./skill.json) and synced through the scripts in [`scripts/`](./scripts/).

### Option A — Claude Code plugin (recommended, auto-updatable)

```
/plugin marketplace add dhosruiasn/ui-tweaker
/plugin install ui-tweaker
```

To update later, refresh the marketplace and re-install — you'll get whatever has been pushed/tagged.

### Option B — portable adapters

```
git clone https://github.com/dhosruiasn/ui-tweaker.git
cd ui-tweaker
scripts/install --target codex
scripts/install --target cursor --dest /path/to/project
scripts/install --target windsurf --dest /path/to/project
```

Supported targets:

- `codex` — installs `skills/ui-tweaker/` as a skill folder.
- `cursor` — creates `.cursor/rules/ui-tweaker.mdc` and syncs the skill folder into `.cursor/skills/ui-tweaker`.
- `windsurf` — creates `.windsurf/rules/ui-tweaker.md` and syncs the skill folder into `.windsurf/skills/ui-tweaker`.
- `portable` — copies or symlinks the skill folder to a destination you choose.

Use `--link` instead of the default copy mode when you want local package edits to reflect immediately in the adapter target.

Updates:

```
scripts/update --dry-run
scripts/update
scripts/install --target cursor --dest /path/to/project
scripts/publish -m "Update UI Tweaker"
scripts/doctor
```

`scripts/update` requires this folder to be a Git checkout. It backs up the current package before a fast-forward update and never executes downloaded code. Re-run `scripts/install` after updating to sync each adapter target.

Publishing is manual by design. `scripts/publish` stages package changes, creates a commit when needed, and pushes the current branch to `origin`; it does not install a background auto-push hook.

## Usage

Just ask, in plain language:

> "Help me tweak the card component" · "I want to adjust the padding on this button" · "this feels too cramped"

Your AI assistant will:
1. Copy `skills/ui-tweaker/template/panel-template.html` next to your project (served so the browser can open it).
2. Fill the 8 `⟦PROJECT-SPECIFIC⟧` markers from your real component (link your stylesheet, paste the real DOM, build the Layers tree).
3. Open the control panel. You adjust → **Confirm**.
4. The assistant applies the structured values to your source and verifies.

## How the fixed template works

`skills/ui-tweaker/template/panel-template.html` is a **frozen UI**. Every project gets the exact same panel — only the marked `⟦PROJECT-SPECIFIC⟧` spots change:

| Marker | What it holds |
| --- | --- |
| `#1` | `<title>` |
| `#2` | font `<link>` (your project's fonts) |
| `#3` | stylesheet `<link href>` (your real CSS — never hand-copy values) |
| `#4` | preview DOM (your real component markup, each atomic element gets `data-pick`) |
| `#5` | `SEL` map (pick key → selector inside the widget) |
| `#6` | `OUT_SEL` map (pick key → CSS selector used in the confirm output) |
| `#7` | `LAYERS` tree (your DOM decomposed to atomic level) |
| `#8` | confirm-output "component / source" strings |
| `#9` | component stage-reset CSS — only needed when your real component is hidden / absolutely positioned in production |

Everything else (layout, CSS, builder JS) stays byte-for-byte identical. See [`SKILL.md`](./skills/ui-tweaker/SKILL.md) for the full spec.

As shipped, the template is wired to the generic demo component in [`examples/demo.css`](./examples/demo.css), so you can open `skills/ui-tweaker/template/panel-template.html` straight away and see the panel work before wiring up your own project.

## Feedback & contributing

- 🐞 **Bugs / requests** → [open an issue](https://github.com/dhosruiasn/ui-tweaker/issues) (templates provided).
- 💬 **Questions / ideas** → [open an issue](https://github.com/dhosruiasn/ui-tweaker/issues/new/choose).
- PRs welcome — see [`CONTRIBUTING.md`](./CONTRIBUTING.md).

## License

[MIT](./LICENSE) © 2026 `HsuanTung Kao`
