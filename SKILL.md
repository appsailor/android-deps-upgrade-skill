---
name: android-deps-upgrade
version: 1.3.0
description: "Upgrade all Android Gradle dependencies to latest stable versions across every module, then commit and push"
argument-hint: "[--check-only]"
allowed-tools:
  - Read
  - Edit
  - Bash
  - WebSearch
  - WebFetch
---

<objective>
Full dependency upgrade pass for any Android project.

Auto-discovers the Android project root (the directory containing `settings.gradle.kts`), scans every `build.gradle.kts` and `gradle-wrapper.properties`, looks up the latest stable version of each dependency from authoritative sources, applies all safe upgrades, then commits and pushes with a detailed changelog.

If `--check-only` is passed in `$ARGUMENTS`, report what is outdated but make no changes.

**Applies automatically:**
- Patch and minor bumps for all dependencies
- AGP + Gradle wrapper together (they have a required minimum pairing — always check the compatibility table)

**Flags but does not apply:**
- Major version bumps where the artifact ID or API changes (e.g. Coil 2.x → 3.x, Room 2.x → 3.x) — report these with migration notes instead.
</objective>

<process>

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
grep -rn 'firebase-.*-ktx\|firebase-ktx\|firebase.ktx.Firebase\|firebase\..*\.ktx\.' \
  <android_root> --include="*.kts" --include="*.kt" | grep -v "/build/"
```

Any `com.google.firebase:firebase-*-ktx` artifact is marked **MIGRATE** (not UPGRADE) — see Step 6 for the migration procedure. Firebase removed all `-ktx` modules from the BOM starting with v34.0.0 (July 2025). The Kotlin APIs were merged into the main modules.

## Step 4 — Look up latest stable versions

Batch related lookups with parallel WebSearch/WebFetch calls. Use these authoritative sources:

**Important:** Use **WebFetch** (not WebSearch) for the sources marked ⚡ below. Search engine results can lag by days and will silently miss same-month patch releases (e.g. `2026.05.00` → `2026.05.01`). WebFetch retrieves the live page.

| Dependency group | Source |
|---|---|
| `com.android.application` / `com.android.library` (AGP) | ⚡ WebFetch `developer.android.com/build/releases/about-agp` — also check minimum Gradle version from the compatibility table |
| Gradle wrapper | check the compatibility table on the AGP page above |
| `org.jetbrains.kotlin.*` | ⚡ WebFetch `kotlinlang.org/docs/releases.html` |
| `com.google.devtools.ksp` | ⚡ WebFetch `github.com/google/ksp/releases` |
| `androidx.*` (all AndroidX) | ⚡ WebFetch `developer.android.com/jetpack/androidx/versions/stable-channel` — search results may be stale |
| `androidx.compose:compose-bom` | ⚡ WebFetch `developer.android.com/develop/ui/compose/bom/bom-mapping` — find the **highest** version listed, including patch releases like `YYYY.MM.01`. Never rely on search for this. |
| `com.google.firebase:firebase-bom` | ⚡ WebFetch `firebase.google.com/support/release-notes/android` |
| `com.google.firebase:*` (explicit, non-ktx) | same Firebase release notes page |
| `com.google.firebase:firebase-*-ktx` | **MIGRATE** — do not look up version; these were removed from the BOM at v34.0.0. Drop `-ktx` suffix: `firebase-analytics-ktx` → `firebase-analytics`, `firebase-firestore-ktx` → `firebase-firestore`, etc. If the module doesn't declare the Firebase BOM yet, add it. Then update Kotlin imports (see Step 6). |
| `com.google.android.gms:*` | `developers.google.com/android/guides/releases` |
| `com.google.gms:google-services` | `developers.google.com/android/guides/google-services-plugin` |
| `org.jetbrains.kotlinx:kotlinx-serialization-*` | `github.com/Kotlin/kotlinx.serialization/releases` |
| `org.jetbrains.kotlinx:kotlinx-coroutines-*` | `github.com/Kotlin/kotlinx.coroutines/releases` |
| `com.squareup.okhttp3:*` | `square.github.io/okhttp/changelogs/changelog/` |
| `io.coil-kt:*` (2.x) | `github.com/coil-kt/coil/releases` — flag 2.x → 3.x as major (different artifact ID) |
| Everything else | `central.sonatype.com/artifact/<group>/<artifact>` |

## Step 5 — Build and display the upgrade table

Before touching any file, show the full picture:

```
Dependency                                  Current       Latest        Action
------------------------------------------  -----------   -----------   -------
com.android.application (AGP)               9.2.0         9.3.0         UPGRADE
org.gradle (wrapper)                        9.4.1         9.5.0         UPGRADE
androidx.compose:compose-bom               2026.05.00    2026.06.00    UPGRADE
com.google.firebase:firebase-bom            34.14.0       34.14.0       OK
com.google.firebase:firebase-analytics-ktx  (BOM)         —             MIGRATE → firebase-analytics (ktx removed in BOM 34.0.0)
com.google.firebase:firebase-firestore-ktx  25.1.1        —             MIGRATE → firebase-firestore (ktx removed in BOM 34.0.0)
io.coil-kt:coil-compose                    2.7.0         3.4.0         MAJOR — skip (artifact changed to io.coil-kt.coil3)
...
```

Action legend: **OK** = already latest, **UPGRADE** = safe version bump, **MIGRATE** = artifact rename/restructure required, **MAJOR** = breaking change, skip and report.

If `--check-only` is in `$ARGUMENTS`: print the table and stop here.

## Step 6 — Apply upgrades

For each dependency marked UPGRADE:
1. Find every file containing the old version string (grep across all .kts and .properties files, excluding build/)
2. Use `Edit` with `replace_all: true` on each affected file
3. **AGP**: update both `com.android.application` and `com.android.library` version strings in the project-level build.gradle.kts in one pass
4. **Gradle wrapper**: edit the `distributionUrl` line — always pair with AGP upgrade if AGP changed
5. **BOMs**: update the version in every module that declares it explicitly

For each dependency marked MIGRATE (Firebase KTX → main module):
1. In the `build.gradle.kts` of the affected module:
   - Replace `"com.google.firebase:firebase-FOO-ktx:X.Y.Z"` with `"com.google.firebase:firebase-FOO"` (no version — managed by BOM)
   - If the module has no Firebase BOM `platform(...)` declaration, add one: `implementation(platform("com.google.firebase:firebase-bom:<current_bom_version>"))`
2. In Kotlin source files in that module, update package imports:
   - `import com.google.firebase.ktx.Firebase` → `import com.google.firebase.Firebase`
   - `import com.google.firebase.FOO.ktx.bar` → `import com.google.firebase.FOO.bar`
   - Run: `grep -rn "firebase.*\.ktx\." <module_src_dir> --include="*.kt"` to find all affected files
3. Verify the module's source compiles after each migration before moving to the next

## Step 7 — Verify no old versions remain

For every version that was changed, run:

```bash
grep -rn "<old_version>" <android_root> --include="*.kts" --include="*.properties" | grep -v "/build/"
```

If any old version still appears in a source file, fix it before continuing.

## Step 8 — Commit and push

Stage only modified gradle files. `gradle.properties` is typically gitignored — never force-add it:

```bash
git add $(git diff --name-only | grep -E "build\.gradle\.kts|gradle-wrapper\.properties")
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

## Step 9 — Summary to user

Report:
- Count of dependencies upgraded
- Count already at latest (no change)
- Any major-version skips with a brief migration note for each
- Reminder to do a release build smoke-test if R8/minification is enabled (new versions can expose missing ProGuard rules)

</process>
