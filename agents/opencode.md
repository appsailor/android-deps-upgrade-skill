---
description: Upgrade all Android Gradle dependencies to latest stable versions across every module, then commit and push
---

Upgrades all Android Gradle dependencies to the latest stable versions across every module, then commits and pushes.

If `--check-only` is in $ARGUMENTS, report what is outdated but make no changes.

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

The directory containing `settings.gradle.kts` is the Android root. All subsequent paths are relative to it. If multiple roots exist, ask the user which one to upgrade before continuing.

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

Also read `gradle-wrapper.properties` for the current Gradle version.

**Firebase KTX detection — run separately:**

```bash
grep -rn 'firebase-.*-ktx\|firebase-ktx\|firebase\.ktx\.Firebase\|firebase\..*\.ktx\.' \
  <android_root> --include="*.kts" --include="*.kt" | grep -v "/build/"
```

Any `com.google.firebase:firebase-*-ktx` artifact is marked **MIGRATE** (not UPGRADE). Firebase removed all `-ktx` modules from the BOM at v34.0.0 (July 2025). The Kotlin APIs were merged into the main modules.

## Step 4 — Look up latest stable versions

**Important:** Use **direct URL fetch** (not web search) for sources marked ⚡. Search results can lag by days and miss same-month patch releases (e.g. `2026.05.00` → `2026.05.01`).

| Dependency group | Source |
|---|---|
| `com.android.application` / `com.android.library` (AGP) | ⚡ fetch `developer.android.com/build/releases/about-agp` — also check minimum Gradle version |
| Gradle wrapper | check the AGP compatibility table |
| `org.jetbrains.kotlin.*` | ⚡ fetch `kotlinlang.org/docs/releases.html` |
| `com.google.devtools.ksp` | ⚡ fetch `github.com/google/ksp/releases` |
| `androidx.*` (all AndroidX) | ⚡ fetch `developer.android.com/jetpack/androidx/versions/stable-channel` |
| `androidx.compose:compose-bom` | ⚡ fetch `developer.android.com/develop/ui/compose/bom/bom-mapping` — find the **highest** version including patches like `YYYY.MM.01` |
| `com.google.firebase:firebase-bom` | ⚡ fetch `firebase.google.com/support/release-notes/android` |
| `com.google.firebase:*` (explicit, non-ktx) | same Firebase release notes page |
| `com.google.firebase:firebase-*-ktx` | **MIGRATE** — drop `-ktx` suffix, add BOM if missing, update Kotlin imports (see Step 6) |
| `com.google.android.gms:*` | `developers.google.com/android/guides/releases` |
| `com.google.gms:google-services` | `developers.google.com/android/guides/google-services-plugin` |
| `org.jetbrains.kotlinx:kotlinx-serialization-*` | `github.com/Kotlin/kotlinx.serialization/releases` |
| `org.jetbrains.kotlinx:kotlinx-coroutines-*` | `github.com/Kotlin/kotlinx.coroutines/releases` |
| `com.squareup.okhttp3:*` | `square.github.io/okhttp/changelogs/changelog/` |
| `io.coil-kt:*` (2.x) | `github.com/coil-kt/coil/releases` — flag 2.x → 3.x as MAJOR |
| Everything else | `central.sonatype.com/artifact/<group>/<artifact>` |

## Step 5 — Build and display the upgrade table

```
Dependency                                  Current       Latest        Action
------------------------------------------  -----------   -----------   -------
com.android.application (AGP)               9.2.0         9.3.0         UPGRADE
org.gradle (wrapper)                        9.4.1         9.5.0         UPGRADE
androidx.compose:compose-bom                2026.05.00    2026.05.01    UPGRADE
com.google.firebase:firebase-bom            34.14.0       34.14.0       OK
com.google.firebase:firebase-analytics-ktx  (BOM)         —             MIGRATE → firebase-analytics
io.coil-kt:coil-compose                    2.7.0         3.4.0         MAJOR — skip (group → io.coil-kt.coil3)
```

**OK** = latest · **UPGRADE** = safe bump · **MIGRATE** = artifact rename · **MAJOR** = breaking, skip.

If `--check-only` is in $ARGUMENTS: print the table and stop.

## Step 6 — Apply upgrades

For each **UPGRADE**: grep for the old version string, edit every affected `.kts` and `.properties` file replacing all occurrences. Pair AGP + Gradle wrapper upgrades.

For each **MIGRATE** (Firebase KTX):
1. In `build.gradle.kts`: replace `firebase-FOO-ktx:X.Y.Z` → `firebase-FOO` (no version); add Firebase BOM `platform(...)` if absent
2. In `.kt` sources: update `import com.google.firebase.ktx.Firebase` → `import com.google.firebase.Firebase`; update `import com.google.firebase.FOO.ktx.bar` → `import com.google.firebase.FOO.bar`
3. Compile-check each module before moving on

## Step 7 — Verify no old versions remain

```bash
grep -rn "<old_version>" <android_root> --include="*.kts" --include="*.properties" | grep -v "/build/"
```

## Step 8 — Commit and push

```bash
git add $(git diff --name-only | grep -E "build\.gradle\.kts|gradle-wrapper\.properties|\.kt$")
git status
git commit -m "chore(Android): upgrade Gradle dependencies — $(date +%Y-%m-%d)

- <dep> <old> → <new>
..."
git push
```

## Step 9 — Summary

Report: upgraded count · already-latest count · MAJOR skips with notes · reminder to smoke-test release build if R8 is enabled.
