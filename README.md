# android-deps-upgrade — Claude Code Skill

A [Claude Code](https://claude.ai/code) skill that upgrades all Android Gradle dependencies to their latest stable versions across every module, then commits and pushes.

## What it does

- Auto-discovers the Android project root (finds `settings.gradle.kts`)
- Extracts every versioned dependency across all `build.gradle.kts` files
- Looks up latest stable versions from authoritative sources (AndroidX stable channel, Firebase release notes, kotlinlang.org, Maven Central, etc.)
- Upgrades AGP + Gradle wrapper together (respects the required minimum pairing)
- **Migrates deprecated artifacts**: detects `firebase-*-ktx` artifacts removed from the Firebase BOM at v34.0.0, renames them to the main module equivalent, and updates Kotlin import paths in source files
- Flags major version bumps (e.g. Coil 2.x → 3.x) with migration notes instead of auto-applying
- Commits and pushes with a structured changelog message

## Install

Copy `SKILL.md` into your Claude Code global skills directory:

```bash
mkdir -p ~/.claude/skills/android-deps-upgrade
curl -o ~/.claude/skills/android-deps-upgrade/SKILL.md \
  https://raw.githubusercontent.com/appsailor/android-deps-upgrade-skill/main/SKILL.md
```

## Usage

From any Android project in Claude Code:

```
/android-deps-upgrade
```

Preview what's outdated without changing anything:

```
/android-deps-upgrade --check-only
```

## Output example

```
Dependency                                  Current      Latest       Action
------------------------------------------  -----------  -----------  -------
com.android.application (AGP)               9.1.0        9.2.0        UPGRADE
org.gradle (wrapper)                        9.3.1        9.4.1        UPGRADE
androidx.compose:compose-bom                2026.03.01   2026.05.00   UPGRADE
kotlinx-serialization-json                  1.10.0       1.11.0       UPGRADE
com.google.firebase:firebase-bom            34.14.0      34.14.0      OK
com.google.firebase:firebase-analytics-ktx  (BOM)        —            MIGRATE → firebase-analytics
io.coil-kt:coil-compose                     2.7.0        3.4.0        MAJOR — skip (artifact → io.coil-kt.coil3)
```

## Requirements

- [Claude Code](https://claude.ai/code)
- Android project using Kotlin DSL (`build.gradle.kts`)

## License

MIT
