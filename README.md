# NEXUS GAMER

An anime-inspired native Android gaming-analytics app. It reads **real**
on-device app usage (via `UsageStatsManager`) to show how much time you
actually spend in games — no fabricated numbers anywhere in the codebase.

## Tech stack

- Kotlin, Jetpack Compose, Material 3
- `UsageStatsManager` for real usage data
- Room for local persistence (game classifications, cached daily usage,
  achievements)
- Jetpack DataStore for lightweight settings (daily goal, AI toggle, last
  refresh time)
- Coroutines + `AndroidViewModel`, simple manual DI (see `NexusApplication`)
- Navigation-Compose for the 5-tab bottom nav + game-detail screen

## How to open it

1. Open the `nexus-gamer/` folder in Android Studio (Koala or newer)
   as an existing project.
2. Let Gradle sync — it will pull AGP 8.5.2 / Kotlin 1.9.24 / Compose BOM
   2024.06.00.
3. Run on a device or emulator with **API 26+**. Usage-stats data is far
   more meaningful on a real device with real app usage history than on a
   freshly-booted emulator.

## Why some things behave the way they do

- **No fake data, anywhere.** `UsageStatsDataSource` is the only place
  that talks to `UsageStatsManager`. `GameStatsRepository` is the only
  place that turns those numbers into daily/weekly/monthly totals, and it
  is the single source of truth the whole UI layer reads from. If you
  grep the codebase for `Random`, `Math.random`, or hard-coded duration
  literals used *as data*, you won't find any — the `2h 34m` example in
  the product brief is illustrative copy, not a literal value in code.
- **Usage Access can't be requested like a normal runtime permission.**
  There is no system dialog for `PACKAGE_USAGE_STATS`; the only path is
  sending the user to `Settings.ACTION_USAGE_ACCESS_SETTINGS` and then
  re-checking `AppOpsManager` ourselves when they return
  (`UsagePermissionManager`, checked again in `MainActivity.onResume`).
- **No `QUERY_ALL_PACKAGES`.** The manifest deliberately does not request
  broad package visibility. `UsageStatsManager` reports package names
  regardless of visibility restrictions; resolving a *name/icon* for one
  of those packages via `PackageManager` can occasionally fail on some
  OEM/Android version combinations if the package isn't otherwise
  visible to us. When that happens the UI falls back to showing the raw
  package name rather than guessing — see `GameClassifier.resolveDisplayName`.
- **Game classification never invents a name.** A package is only ever
  treated as a "game" because (a) it's on a small curated known-package
  list, (b) Android itself reports `ApplicationInfo.category ==
  CATEGORY_GAME`, or (c) the user tapped "Mark as Game". Users can always
  correct a misclassification from the Games list's overflow menu.
- **Achievements and streaks are computed, not scripted.** See
  `AchievementEngine` and `GameStatsRepository.computeCurrentStreak` —
  every unlock condition is evaluated against the cached real-usage rows.
- **The "AI Gaming Analyst" is a deterministic, rule-based summarizer**
  (`AiAnalystEngine`), not a call to an external generative model. Every
  sentence it produces is derived from numbers already visible elsewhere
  in the app, by design (see spec section 11) — it never fabricates a
  statistic, and it says "Not enough real data yet." when there isn't
  enough history to say anything.
- **Nothing leaves the device by default.** All storage is local Room /
  DataStore. The Privacy Center screen states this plainly and includes
  a real, irreversible "Delete My NEXUS Data" action
  (`GameStatsRepository.deleteAllLocalData`).

## What's a placeholder vs. what's real logic

- The mascot art (`MascotArt.kt`) is intentionally simple original
  geometric glyphs for KAI / YUNA / RAI — meant to be swapped for final
  illustrations by an artist later. The *system* behind them (usage
  math, classification, achievements, streaks) is fully implemented and
  is not placeholder logic.
- The bar charts in `UsageBarChart.kt` are a lightweight custom Canvas
  implementation rather than a third-party charting library, to keep the
  dependency footprint small; they only ever draw values the ViewModel
  passes in, which trace back to the repository.

## Known scope notes for a "production-quality" pass

This is a complete, coherent scaffold covering every screen and data flow
in the spec. Before shipping you'd still want to add: unit tests for
`UsageStatsDataSource`'s midnight-spanning-session logic and the streak
calculator, a Room migration strategy (currently a single `version = 1`
schema), a proper app icon/splash illustration, and manual QA on a few
OEM skins (Samsung/Xiaomi/etc. sometimes rename or relocate the Usage
Access settings screen).
