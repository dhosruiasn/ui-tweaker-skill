# Portable Skill Adapters

UI Tweaker is packaged as a Claude Code plugin and as a portable skill folder.
The portable path is intentionally conservative: adapters copy or link the skill
instructions into a host AI's own rules/skills location, then updates are pulled
from GitHub and re-synced locally.

## Targets

- `claude-code`: use Claude Code's native plugin marketplace commands.
- `codex`: install `skills/ui-tweaker/` as a skill folder.
- `cursor`: create a project rule and copy/link the skill folder under `.cursor/`.
- `windsurf`: create a project rule and copy/link the skill folder under `.windsurf/`.
- `portable`: copy/link only the skill folder to a destination you choose.

## Commands

```sh
scripts/install --target codex
scripts/install --target cursor --dest /path/to/project
scripts/install --target windsurf --dest /path/to/project --link
scripts/update --dry-run
scripts/doctor
```

The adapter scripts do not edit application source files and do not execute
downloaded code. Updates require this package to be a Git checkout.
