# Contributing

Thanks for your interest in improving UI Tweaker!

## Ground rules

This is a portable AI **skill**, not an app — the "code" is mostly instructions (`SKILL.md` /
`SKILL.zh-TW.md`, with detailed specs split into `references/*.md` loaded on demand) plus a
frozen HTML template (`template/panel-template.html`). Two rules keep it coherent:

1. **The template UI is frozen.** Layout, CSS, and builder JS must stay byte-for-byte
   identical across projects. Only the 8 `⟦PROJECT-SPECIFIC⟧` markers are meant to change
   per-project. If you change panel behaviour, change it *in the template* so every project
   benefits — never bake project-specific markup outside the markers.
2. **Both languages stay in sync.** `SKILL.md` + `references/*.md` (English) and
   `SKILL.zh-TW.md` + `references/*.zh-TW.md` (繁體中文) describe the same behaviour.
   A behaviour change must land in both, in the matching file (main file for flow/token
   rules, the corresponding reference for detailed specs).

## Workflow

1. Fork & branch.
2. Make your change. If you touch the template, verify the inline `<script>` still parses
   (`node --check` on the extracted script) and that it renders in a browser.
3. Update `CHANGELOG.md` under an `## [Unreleased]` heading.
4. Open a PR describing the behaviour change and, for anything visual, attach a screenshot/GIF.

## Releasing (maintainers)

1. Bump `version` in `skill.json` and `.claude-plugin/plugin.json` (semver).
2. Move `[Unreleased]` notes to a dated version section in `CHANGELOG.md`.
3. Tag and push: `git tag v1.x.y && git push --tags`.

Plugin users update by refreshing the marketplace; manual users `git pull`.
