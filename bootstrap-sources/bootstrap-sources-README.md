# Bootstrap Sources

Temporary input directory used during the initial bootstrap of the wiki. After bootstrap is complete, this directory may be removed or kept as historical archive (your choice).

## Structure

```
bootstrap-sources/
├── claude-memory.md          ← memory dump from Claude.ai sessions
├── specs/                    ← architecture specifications from projects
└── discussions/              ← selected conversation transcripts
```

## Source reliability hierarchy

When Claude Code processes these materials during bootstrap, sources are weighted by reliability:

1. **Project code** (in `C:\flutter-projects\go_sport\`, not in this folder) — highest authority. Code is what actually exists.
2. **`specs/`** — architecture specifications. Close to reality but may lag behind recent refactorings. Verify against code when in doubt.
3. **`claude-memory.md`** and **`discussions/`** — what was discussed and intended. Treat as hypotheses. Verify every significant claim against code before fixing into the wiki.

When sources disagree, the lower-numbered source wins. Code is always the source of truth for wiki page content.

## What goes where

### `claude-memory.md`

A single file containing the distilled memory dump from Claude.ai sessions. Provided as one cohesive document, not split.

### `specs/`

Copies of architecture specification files from project repositories. For go_sport, these are:
- `ARCHITECTURE_SPEC.md`
- `DOMAIN_SPEC.md`
- `DATA_SPEC.md`
- `SCREENS_SPEC.md`

Copy these from the project's docs directory; do not symlink.

### `discussions/`

Selected conversation transcripts that contain valuable engineering discussion. Curated by the human — do not include the entire Claude Code session history.

Naming convention for files in this directory:
- `discussion-NNN-<topic>.md` where NNN is a sequence number (001, 002, ...)
- Topic is a short kebab-case description (e.g., `player-architecture`, `navigation-back-stack-issue`, `mini-player-modal-pitfall`)

Examples:
- `discussion-001-player-architecture.md`
- `discussion-002-mini-player-fullscreen-navigation.md`
- `discussion-003-wiki-design-conversation.md`

Each file should contain the relevant conversation as plain markdown or text. Original format from Claude.ai or Claude Code logs is acceptable; the curator agent will parse it.

## What NOT to include

- Code files directly. Code lives in the project repositories; reference it by path, do not copy it here.
- Generated artifacts, build outputs, screenshots, or images. Wiki is text-based.
- Routine debug sessions, minor bug fixes, or implementation iterations without significant decisions. These create noise.
- Anything containing secrets (API keys, passwords, tokens). This directory may end up in git.

## After bootstrap

Once the wiki is populated, `bootstrap-sources/` has served its purpose. Three options:

1. **Keep as archive.** Useful if you want to revisit the raw materials later or if future bootstraps need to reference them.
2. **Move out of repo.** Move the directory to an external location for archive; remove from the wiki repo to keep it clean.
3. **Delete.** Git history will preserve them if needed.

The `.gitignore` in the wiki root has a commented-out line for excluding `bootstrap-sources/` if you choose to keep them locally but not in git.
