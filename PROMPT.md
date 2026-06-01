# Android Dependency Upgrade

> Version: 1.3.0 · Works with Claude Code, OpenCode, Cursor, Kiro, Codex, Gemini CLI, Grok

Upgrades all Android Gradle dependencies to the latest stable versions across every module, then commits and pushes.

Pass `--check-only` to report what is outdated without making any changes.

**Applies automatically:**
- Patch and minor bumps for all dependencies
- AGP + Gradle wrapper together (required minimum pairing — always check the compatibility table)

**Flags but does not apply:**
- Major version bumps where the artifact ID or API changes (e.g. Coil 2.x → 3.x) — reported with migration notes

---

## Step 1 — Locate the Android project root

```bash
find . -name "settings.gradle.kts" | grep -v "/build/" | head -5
```

The directory containing `settings.gradle.kts` is the Android root. All subsequent paths are relative to it. If multiple roots exist (e.g. multiple Android subprojects), ask the user which one to upgrade before continuing.

## Step 2 — Discover all gradle files

```bash
find <android_root> \( -name "build.gradle.kts" -o -name "gradle-wrapper.properties" \) \
  | grep -v "/build/" | sort
```

## Step 3 — Extract all unique versioned dependencies

```bash
grep -rh 'implementation\|ksp\|testImplementation\|androidTestImplementation\|debugImplementation\|classpath\|platform(' \
  <android_root> --include="*.kts" \
  | grep -v "/build/" | grep -v 'project(' | grep '"' \
  | sed 's/.*"\(.*\)".*/\1/' | grep -E ':[0-9]+\.[0-9]+' | sort -u
```

Also read `gradle-wrapper.properties` to get the current Gradle distribution version.

**Firebase KTX detection — run separately:**

```bash
grep -rn 'firebase-.*-ktx\|firebase-ktx\|firebase\.ktx\.Firebase\|firebase\..*\.ktx\.' \
  <android_root> --include="*.kts" --include="*.kt" | grep -v "/build/"
```

Any `com.google.firebase:firebase-*-ktx` artifact is marked **MIGRATE** (not UPGRADE) — see Step 6. Firebase removed all `-ktx` modules from the BOM starting with v34.0.0 (July 2025). The Kotlin APIs were merged into the main modules.

## Step 4 — Look up latest stable versions

**Important:** Use **direct URL fetch** (not web search) for sources marked ⚡. Search results can lag by days and silently miss same-month patch releases (e.g. `2026.05.00` → `2026.05.01`). Fetch the live page instead.

| Dependency group | Source |
|---|---|
| `com.android.application` / `com.android.library` (AGP) | ⚡ fetch `developer.android.com/build/releases/about-agp` — also check minimum Gradle version from the compatibility table |
| Gradle wrapper | check the AGP compatibility table for the minimum required version |
| `org.jetbrains.kotlin.*` | ⚡ fetch `kotlinlang.org/docs/releases.html` |
| `com.google.devtools.ksp` | ⚡ fetch `github.com/google/ksp/releases` |
| `androidx.*` (all AndroidX) | ⚡ fetch `developer.android.com/jetpack/androidx/versions/stable-channel` |
| `androidx.compose:compose-bom` | ⚡ fetch `developer.android.com/develop/ui/compose/bom/bom-mapping` — find the **highest** version listed, including patch releases like `YYYY.MM.01` |
| `com.google.firebase:firebase-bom` | ⚡ fetch `firebase.google.com/support/release-notes/android` |
| `com.google.firebase:*` (explicit, non-ktx) | same Firebase release notes page |
| `com.google.firebase:firebase-*-ktx` | **MIGRATE** — do not look up version. Drop `-ktx` suffix: `firebase-analytics-ktx` → `firebase-analytics`, `firebase-firestore-ktx` → `firebase-firestore`, etc. Add Firebase BOM to the module if not present. Update Kotlin imports (see Step 6). |
| `com.google.android.gms:*` | `developers.google.com/android/guides/releases` |
| `com.google.gms:google-services` | `developers.google.com/android/guides/google-services-plugin` |
| `org.jetbrains.kotlinx:kotlinx-serialization-*` | `github.com/Kotlin/kotlinx.serialization/releases` |
| `org.jetbrains.kotlinx:kotlinx-coroutines-*` | `github.com/Kotlin/kotlinx.coroutines/releases` |
| `com.squareup.okhttp3:*` | `square.github.io/okhttp/changelogs/changelog/` |
| `io.coil-kt:*` (2.x) | `github.com/coil-kt/coil/releases` — flag 2.x → 3.x as MAJOR (group ID changes to `io.coil-kt.coil3`) |
| Everything else | `central.sonatype.com/artifact/<group>/<artifact>` |

## Step 5 — Build and display the upgrade table

Before touching any file, show the full picture:

```
Dependency                                  Current       Latest        Action
------------------------------------------  -----------   -----------   -------
com.android.application (AGP)               9.2.0         9.3.0         UPGRADE
org.gradle (wrapper)                        9.4.1         9.5.0         UPGRADE
androidx.compose:compose-bom                2026.05.00    2026.05.01    UPGRADE
com.google.firebase:firebase-bom            34.14.0       34.14.0       OK
com.google.firebase:firebase-analytics-ktx  (BOM)         —             MIGRATE → firebase-analytics
com.google.firebase:firebase-firestore-ktx  25.1.1        —             MIGRATE → firebase-firestore
io.coil-kt:coil-compose                    2.7.0         3.4.0         MAJOR — skip (group → io.coil-kt.coil3)
...
```

Action legend: **OK** = already latest · **UPGRADE** = safe bump · **MIGRATE** = artifact rename required · **MAJOR** = breaking, skip and report.

If `--check-only` was requested: print the table and stop here.

## Step 6 — Apply upgrades

For each dependency marked **UPGRADE**:
1. Grep for the old version string across all `.kts` and `.properties` files (excluding `build/`)
2. Edit each affected file, replacing all occurrences of the old version with the new one
3. **AGP**: update both `com.android.application` and `com.android.library` in the project-level `build.gradle.kts` in one pass
4. **Gradle wrapper**: edit the `distributionUrl` line — always pair with AGP if AGP changed
5. **BOMs**: update the version in every module that declares it explicitly

For each dependency marked **MIGRATE** (Firebase KTX → main module):
1. In the module's `build.gradle.kts`:
   - Replace `"com.google.firebase:firebase-FOO-ktx:X.Y.Z"` with `"com.google.firebase:firebase-FOO"` (no version — BOM-managed)
   - If the module has no Firebase BOM `platform(...)`, add: `implementation(platform("com.google.firebase:firebase-bom:<version>"))`
2. In Kotlin source files of that module, update imports:
   - `import com.google.firebase.ktx.Firebase` → `import com.google.firebase.Firebase`
   - `import com.google.firebase.FOO.ktx.bar` → `import com.google.firebase.FOO.bar`
   - Find all affected files: `grep -rn "firebase.*\.ktx\." <module_src_dir> --include="*.kt"`
3. Verify the module compiles after each migration before moving to the next

## Step 7 — Verify no old versions remain

For every version that was changed:

```bash
grep -rn "<old_version>" <android_root> --include="*.kts" --include="*.properties" | grep -v "/build/"
```

Fix any remaining occurrences before continuing.

## Step 8 — Commit and push

Stage modified gradle files and any Kotlin sources touched by MIGRATE steps:

```bash
git add $(git diff --name-only | grep -E "build\.gradle\.kts|gradle-wrapper\.properties|\.kt$")
git status  # confirm staged files look correct
```

Commit with a structured message:
```
chore(Android): upgrade Gradle dependencies — YYYY-MM-DD

- <dep> <old> → <new>
- <dep> <old> → <new>
...

Skipped (major version — manual migration required):
- <dep> <old> → <new>: <one-line migration note>
```

Then push.

## Step 9 — Summary

Report:
- Count upgraded
- Count already at latest
- MAJOR skips with migration notes
- Reminder to run a release build smoke-test if R8/minification is enabled
