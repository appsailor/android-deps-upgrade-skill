# android-deps-upgrade — Claude Code Skill `v1.3.0`

A skill that upgrades all Android Gradle dependencies to their latest stable versions across every module, then commits and pushes. Works with Claude Code, OpenCode, Cursor, Kiro, Codex, Gemini CLI, and Grok.

## What it does

- Auto-discovers the Android project root (finds `settings.gradle.kts`)
- Extracts every versioned dependency across all `build.gradle.kts` files
- **Fetches live pages** (not search) for key sources — catches same-month patch releases like `2026.05.01`
- Upgrades AGP + Gradle wrapper together (respects the required minimum pairing)
- **Migrates deprecated artifacts**: detects `firebase-*-ktx` artifacts removed from Firebase BOM at v34.0.0, renames them, and updates Kotlin import paths
- Flags major version bumps (e.g. Coil 2.x → 3.x) with migration notes
- Commits and pushes with a structured changelog

## Agent compatibility

| Agent | File to use | Install path | Invocation |
|---|---|---|---|
| **Claude Code** | `SKILL.md` | `~/.claude/skills/android-deps-upgrade/SKILL.md` | `/android-deps-upgrade [--check-only]` |
| **OpenCode** | `agents/opencode.md` | `.opencode/commands/android-deps-upgrade.md` or `~/.config/opencode/commands/` | `/android-deps-upgrade [--check-only]` |
| **Cursor** | `agents/cursor.mdc` | `.cursor/rules/android-deps-upgrade.mdc` | Ask: *"upgrade Android deps"* |
| **Kiro** | `agents/kiro.md` | `.kiro/steering/android-deps-upgrade.md` or anywhere, then `@android-deps-upgrade` | `@android-deps-upgrade` in chat |
| **Codex** | `agents/codex.md` | Append to `~/.codex/AGENTS.md` or project `AGENTS.md` | *"upgrade Android deps"* |
| **Gemini CLI** | `agents/gemini.toml` | `.gemini/commands/android-deps-upgrade.toml` | `/android-deps-upgrade [--check-only]` |
| **Grok** | `agents/grok.md` | Append to project `AGENTS.md` | *"upgrade Android deps"* |

## Install

### Claude Code
```bash
mkdir -p ~/.claude/skills/android-deps-upgrade
curl -o ~/.claude/skills/android-deps-upgrade/SKILL.md \
  https://raw.githubusercontent.com/appsailor/android-deps-upgrade-skill/main/SKILL.md
```

### OpenCode (project-level)
```bash
mkdir -p .opencode/commands
curl -o .opencode/commands/android-deps-upgrade.md \
  https://raw.githubusercontent.com/appsailor/android-deps-upgrade-skill/main/agents/opencode.md
```

### OpenCode (global)
```bash
mkdir -p ~/.config/opencode/commands
curl -o ~/.config/opencode/commands/android-deps-upgrade.md \
  https://raw.githubusercontent.com/appsailor/android-deps-upgrade-skill/main/agents/opencode.md
```

### Cursor
```bash
mkdir -p .cursor/rules
curl -o .cursor/rules/android-deps-upgrade.mdc \
  https://raw.githubusercontent.com/appsailor/android-deps-upgrade-skill/main/agents/cursor.mdc
```

### Kiro (ad-hoc reference)
```bash
curl -o android-deps-upgrade.md \
  https://raw.githubusercontent.com/appsailor/android-deps-upgrade-skill/main/agents/kiro.md
```
Then reference it in Kiro chat with `@android-deps-upgrade`.

### Kiro (steering — always available)
```bash
mkdir -p .kiro/steering
curl -o .kiro/steering/android-deps-upgrade.md \
  https://raw.githubusercontent.com/appsailor/android-deps-upgrade-skill/main/agents/kiro.md
```

### Codex / Grok (append to AGENTS.md)
```bash
curl https://raw.githubusercontent.com/appsailor/android-deps-upgrade-skill/main/agents/codex.md \
  >> AGENTS.md
```

### Gemini CLI
```bash
mkdir -p .gemini/commands
curl -o .gemini/commands/android-deps-upgrade.toml \
  https://raw.githubusercontent.com/appsailor/android-deps-upgrade-skill/main/agents/gemini.toml
```

## Usage

```
/android-deps-upgrade              # upgrade all deps + commit + push
/android-deps-upgrade --check-only # preview only, no changes
```

For agents without slash commands (Cursor, Kiro, Codex, Grok), ask naturally:
> *"Upgrade all Android Gradle dependencies"*  
> *"Check what Android deps are outdated but don't change anything"*

## Output example

```
Dependency                                  Current      Latest       Action
------------------------------------------  -----------  -----------  -------
com.android.application (AGP)               9.1.0        9.2.0        UPGRADE
org.gradle (wrapper)                        9.3.1        9.4.1        UPGRADE
androidx.compose:compose-bom                2026.05.00   2026.05.01   UPGRADE
kotlinx-serialization-json                  1.10.0       1.11.0       UPGRADE
com.google.firebase:firebase-bom            34.14.0      34.14.0      OK
com.google.firebase:firebase-analytics-ktx  (BOM)        —            MIGRATE → firebase-analytics
io.coil-kt:coil-compose                     2.7.0        3.4.0        MAJOR — skip (artifact → io.coil-kt.coil3)
```

## Requirements

- Android project using Kotlin DSL (`build.gradle.kts`)
- Git repository
- One of the supported AI coding agents

## Changelog

### v1.3.0
- Multi-agent support: added `agents/` for OpenCode, Cursor, Kiro, Codex, Gemini CLI, Grok
- Added `PROMPT.md` — universal agent-neutral version of the skill
- All agent files use plain markdown headers instead of Claude Code XML tags
- Per-agent `curl` install commands in README

### v1.2.0
- Key sources now use **direct URL fetch** instead of web search — prevents missing same-month patch releases like `2026.05.00 → 2026.05.01`
- Added `version` field to SKILL.md frontmatter

### v1.1.0
- Firebase KTX migration: detects `firebase-*-ktx` artifacts removed from Firebase BOM at v34.0.0, renames them, and fixes Kotlin import paths
- New **MIGRATE** action in upgrade table

### v1.0.0
- Initial release

## License

MIT
