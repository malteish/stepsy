# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Stepsy is a single-module Android app (Kotlin, views + XML — no Compose) that counts steps from the
hardware step-counter sensor and stores a daily history locally. It is offline by design: there is no
`INTERNET` permission and no network code in any build. It is a fork of
[Motionmate](https://github.com/0xf4b1/motionmate), which is why some identifiers and paths still
carry the old naming (see "Naming leftovers").

## Build and test

Gradle wrapper, AGP 8.9.1, Kotlin 2.1.10, JDK 17, `compileSdk`/`targetSdk` 35, `minSdk` 26.

Two product flavors on the `distribution` dimension, so nearly every task name needs a flavor:

```bash
./gradlew assembleFossDebug        # no Google dependencies
./gradlew assembleFullDebug        # adds Play Services (Activity Recognition)
./gradlew installFossDebug
./gradlew testFossDebugUnitTest    # unit tests (JVM/Robolectric)
./gradlew lintFossDebug
```

Bare `assembleDebug` builds *both* flavors and writes to
`app/build/outputs/apk/{foss,full}/debug/`, not `app/build/outputs/apk/debug/`.

Run a single test with the standard filter, e.g.
`./gradlew testFossDebugUnitTest --tests "com.nvllz.stepsy.AppTest"`.

Release builds require `app/keystore.jks` plus `KEYSTORE_PASSWORD`, `KEY_ALIAS`, and `KEY_PASSWORD`
in the environment; the signing config is unconditional, so `assemble*Release` fails without them.

Gradle needs network access to `dl.google.com` and Maven Central to resolve the Android Gradle
Plugin. In a sandbox where those hosts are blocked, no Gradle task will run — verify changes by
reading rather than assuming a build failure means the code is broken.

## Testing reality

There is exactly one test, `app/src/test/.../AppTest.kt`, and CI never runs it — the workflow
(`.github/workflows/android.yml`) only runs `assembleDebug`. That test drives `MotionService`
through reflection on private fields and was written against the pre-v2 database schema, when dates
were `Long` timestamps; it still calls `Field.setLong` on `mCurrentDate`, which is a `String` now.
Treat it as stale. If you touch step-counting logic, don't assume a green history means anything.

## Architecture

### Two processes

`MotionService` is declared with `android:process=":MotionService"`, so the app runs in two processes.
Every `object` singleton — `AppPreferences`, `WidgetManager`, `Database.instance` — exists twice, once
per process, and `App.onCreate()` runs in both. Anything shared has to cross the process boundary
explicitly:

- **Service → Activity**: a `ResultReceiver` the activity passes in via `ACTION_SUBSCRIBE`, carrying
  `KEY_STEPS` and `KEY_IS_PAUSED`.
- **Service → tile**: a plain `com.nvllz.stepsy.STATE_UPDATE` broadcast, consumed by
  `StepsyTileService`.
- **Activity/receivers → Service**: intents with actions (`ACTION_PAUSE_COUNTING`,
  `ACTION_RESUME_COUNTING`) or the extras `FORCE_UPDATE` and `MANUAL_STEP_COUNT_CHANGE`.
- **Shared state**: SQLite, DataStore, and `SharedPreferences`. Widget config uses
  `MODE_MULTI_PROCESS` for this reason.

`AppPreferences` uses the `preferencesDataStore` delegate, which is *not* the multi-process
implementation, yet both processes read and write it. Keep that in mind before adding new
cross-process preference traffic — prefer routing through an intent to the service.

### Step counting (`service/MotionService.kt`)

The core of the app. `TYPE_STEP_COUNTER` reports a cumulative count since boot, so `handleEvent`
tracks deltas against `mLastSteps`, using `-1` as a "re-baseline on next event" sentinel (set after
reboot, manual edits, and day rollovers). A delta is dropped entirely when the vehicle filter is on
and `ActivityRecognitionManager.isInVehicle` is true.

Writes are throttled on separate timers — DataStore, SQLite, widgets, and the foreground notification
each have their own interval, and every interval doubles when `PowerManager.isPowerSaveMode` is on.
This is deliberate battery work; don't collapse the tiers into a single write path.

Day rollover is detected inside `handleStepUpdate` by comparing `Util.todayDateString()` against
`mCurrentDate` (lexicographic string comparison — the `yyyy-MM-dd` format makes `>` mean "later").
`MidnightResetReceiver` schedules an inexact alarm to nudge the service just after midnight, and
`MainActivity.onResume` calls `recreate()` when the date changed while it was backgrounded.

### Flavor source sets

`foss/` and `full/` each provide their own `ActivityRecognitionManager` and `isPlayServicesAvailable()`.
The FOSS versions are no-op stubs (`isInVehicle` is always `false`). **Both copies must keep identical
signatures** — main-source-set code calls them unqualified, so changing one and not the other breaks
that flavor's build only, which bare `assembleDebug` will catch but a single-flavor build will not.
Google Play Services may only be referenced from `app/src/full/`.

### Persistence

- **SQLite** (`util/Database.kt`): one table, `History(date TEXT PRIMARY KEY, stepsy INT)`. Dates are
  `yyyy-MM-dd` strings. Schema version 2; version 1 keyed rows by a local-midnight millisecond
  timestamp, and `migrateV1ToV2` converts them by snapping to the nearest UTC midnight so the result
  doesn't depend on the device's current timezone, merging any DST-induced duplicates. `addEntry` is
  an upsert that *overwrites* the day's total, not an increment.
- **DataStore** (`util/AppPreferences.kt`): all user settings plus the live `STEPS`/`DATE`. Every
  property exposes both a `Flow` and a blocking `var` backed by `runBlocking`; the blocking accessors
  are what the service uses on the sensor path.
- **SharedPreferences**, several separate files: `StepsyPrefs` (pause flag), `TimedPausePrefs`,
  `widget_prefs_<appWidgetId>`, `goals_cache` (Gson-serialized achievements), `backup_prefs`.

### UI

One heavyweight `MainActivity` (~1200 lines) holds the calendar, the range/year selectors, and the
chart. `ui/Chart.kt` subclasses MPAndroidChart's `BarChart`. `SettingsActivity` builds its rows from
plain layouts and `MaterialSwitch` widgets — there is no `PreferenceScreen` XML, so a new setting
means adding a key to `AppPreferences.PreferenceKeys`, a row to `res/layout/settings_activity.xml`,
and wiring in `SettingsActivity`.

Three widgets (icon, compact, plain), each a provider + a configure activity + its own prefs file.
All refreshes funnel through `WidgetManager.updateAllWidgets`, which batches non-immediate calls
behind a 150 ms handler post.

### Backup

CSV, one `date,steps` line per row, files named `stepsy_<timestamp>.csv`. `BackupScheduler` drives
scheduled backups through WorkManager into a SAF tree URI, pruning to a configurable retention count;
`BackupActivity` handles manual export/import. Import is destructive — it clears the table via
`Database.clearAllAndImport` — and is gated behind a confirmation dialog.

### External control

`com.nvllz.stepsy.action.PAUSE` / `.RESUME` broadcasts (constants in `api/StepsyApi.kt`) are a public
API used by automation apps like Tasker, handled by the exported `StepsyControlReceiver`. Treat those
action strings as a compatibility contract.

## Translations

`res/values-<lang>/strings.xml` plus `res/xml/locales_config.xml`. The two drift: `locales_config.xml`
currently lists 11 locales while `bg`, `he`, `jp`, `nl`, and `sv` resource directories exist without
entries, so those languages don't appear in the per-app language picker. Note also that the Japanese
directory is `values-jp`, which is not a valid ISO 639-1 qualifier (`ja` is) and therefore never
matches a device locale. When adding a language, update `locales_config.xml` in the same change.

Store metadata for F-Droid/IzzyOnDroid lives in `fastlane/metadata/android/`; a release adds a
changelog file named after the new `versionCode`.

## Naming leftovers

Inherited from the Motionmate fork and easy to trip over:

- The SQLite step column is literally named `stepsy` (a find-and-replace artifact of "steps").
- `service/MotionActivity.kt` declares `package com.nvllz.stepsy`, not `...stepsy.service`, despite
  its directory.
- The test lives under `app/src/test/java/com/tiefensuche/motionmate/` but declares
  `package com.nvllz.stepsy`.
- The CI workflow uploads `motionmate.apk` from a pre-flavor output path that no longer exists.
