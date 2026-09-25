# Canter telemetry: PostHog on iOS and Android, Firebase Crashlytics out

Owner decision, September 25, 2026: "we change that privacy policy as of today as analytics are important." PostHog is the single telemetry tool for every Freedom Terminal app (global CLAUDE.md Rule 23). Canter (RunWalk iOS + watchOS, RunWalk-Android) is the first app in the rollout order because it earns 74% of portfolio revenue.

MISSION reminder for every dispatch: RunWalk/Canter is to be the best marathon training app in the world.

## 1. What the change must achieve (specification)

Canter answers two questions it cannot answer today:

1. **Activation and retention.** Do people who install the app start a workout, finish one, start a program, and come back? Canter is a paid-upfront app ($4.99), so there is no in-app purchase funnel. The in-app funnel is: app opened -> first workout started -> first workout completed -> program started -> repeat workouts (D1/D7/D30 retention). These must be measurable in PostHog per platform.
2. **Stability.** Every crash on iOS and Android reaches PostHog error tracking, symbolicated, and opens a GitHub issue on the app's repo.

Constraints that hold on both platforms:

- **One vendor.** PostHog only. Firebase Crashlytics and Firebase Analytics leave the iOS build. Firebase App Check and FirebaseCore STAY on iOS: the Strava proxy (`runwalk-proxy/api/strava/token.ts`) verifies an `X-Firebase-AppCheck` token, and the Strava integration would break without it. `GoogleService-Info.plist` stays for the same reason. Android has no Firebase today and still gets none (`BuildHygieneTest` Firebase pins stay).
- **Privacy defaults (Rule 23).** Session replay OFF. Person profiles only for identified users (the app never identifies anyone, so events are anonymous). No workout content, route, location, health data, or Strava identity in any event property. Distances and durations are allowed as numbers; coordinates never.
- **Off switch.** A Settings row "Share anonymous usage and crash reports", default ON, persisted. When OFF the SDK is never set up on the next launch, and turning it off at runtime calls the SDK's opt-out and stops sending immediately. HabitFlame's `Analytics.swift` (in `~/Code/HabitFlame/HabitFlamePackage/Sources/HabitFlameFeature/Services/Analytics.swift`) is the reference for the "off = SDK never initialized" contract; copy the contract, not the whole file (it carries HabitFlame-specific kill switches).
- **Same event names and properties on both platforms** (Mac/Windows parity rule applies to Canter iOS/Android as well). Property names are snake_case.
- **Super properties on every event:** `app` = `canter`, `bundle_id` = the platform bundle id (`com.vishutdhar.RunWalk` on iOS, `com.vishutdhar.runwalk` on Android), `app_version`, `app_build`. The PostHog SDK adds `$os`, `$device_type`, `$app_version` etc. on its own; `app` and `bundle_id` are ours.
- **PostHog project:** the existing one, id 326535, host `https://us.i.posthog.com`, public client key `phc_J1tSpBnq3nRgP8VrUlkhQCoB0wIORi3m6EU2q2N537O` (a write-only client token, safe to commit; HabitFlame commits it in `Config/Shared.xcconfig`).
- **No AI tool names** in any file that gets pushed (commit messages, PR bodies, code comments, docs). Grep the diff before committing.
- **No emojis, no em dashes** in pushed text.

## 2. Events (the whole vocabulary; do not invent more)

| Event | When | Properties |
|---|---|---|
| `workout_started` | the timer starts a workout | `mode` (`run_walk`, `walking`, and any other mode the app has), `preset_id` (the preset or program-day identifier, or `custom`), `program_id` (nullable), `gps_enabled` (bool), `source` (`phone`, `watch`, `widget`, `shortcut`, `quick_start`) |
| `workout_completed` | a workout is saved to history | `mode`, `preset_id`, `program_id`, `duration_s` (int), `distance_m` (int, 0 when GPS was off), `interval_count` (int), `auto_paused` (bool), `source` |
| `workout_discarded` | the user ends a workout without saving, or the app discards it | `duration_s`, `reason` (`user`, `too_short`, `restore_declined`) |
| `program_started` | a training program is selected/started | `program_id` |
| `program_workout_completed` | a workout that belongs to a program day is saved | `program_id`, `week` (int), `day` (int) |
| `program_completed` | the last day of a program is saved | `program_id` |
| `strava_connected` | Strava OAuth completes | none |
| `strava_upload` | an upload attempt finishes | `result` (`ok`, `failed`) |
| `setting_changed` | one of the main toggles changes | `key` (`bells`, `haptics`, `voice`, `gps`, `health_save`, `auto_pause`, `reminders`, `start_with_walk`, `telemetry`), `value` (bool) |
| `rating_prompt_shown` | the store rating sheet is requested | none |
| `telemetry_disabled` | the user turns the switch off (sent before opt-out) | none |

Screen views: turn the SDK's automatic screen capture ON on both platforms (`$screen`), it costs nothing and gives the funnel its steps. Application lifecycle events (`Application Opened`, `Application Backgrounded`) ON. Autocapture of UI taps OFF.

Exactly which existing code path fires each event is for the implementer to find; the spec fixes the names, the moments, and the properties. If an event's moment does not exist on a platform (for example Strava on Android), skip it there and say so in the PR body.

## 3. iOS (RunWalk, worktree `/Users/vishutdhar/Code/wt-runwalk-posthog`, branch `vd/20260925-posthog`)

Reference implementation to copy from: HabitFlame (`~/Code/HabitFlame`): `Config/Shared.xcconfig` PostHog block, the Info.plist `POSTHOG_API_KEY` / `POSTHOG_HOST` keys, `HabitFlamePackage/Package.swift` pin, `Analytics.swift` for the setup/opt-out contract, `SettingsView.swift` for the switch, `ci_scripts/ci_post_xcodebuild.sh` + `scripts/test-ci-post-xcodebuild.sh` for dSYM upload.

1. `RunWalkPackage/Package.swift`: remove `FirebaseCrashlytics` and `FirebaseAnalytics` products from the iOS target; keep `FirebaseAppCheck` (and whatever FirebaseCore it pulls). Add `.package(url: "https://github.com/PostHog/posthog-ios.git", exact: "3.58.3")` and the `PostHog` product on the iOS feature target only (NOT the watch or widget targets; PostHog exception autocapture does not support watchOS). 3.58.3 is the version HabitFlame audited for privacy and verified for crash capture; a portfolio-wide bump is a separate change.
2. `RunWalk/RunWalkApp.swift`: delete `import FirebaseCrashlytics` and the `Crashlytics.crashlytics().record(error:)` call; replace it with a PostHog exception capture of the same error (the SDK's manual exception capture API) through the new analytics service, so the CloudKit fallback stays visible in the field. Keep App Check + `FirebaseApp.configure()` exactly as they are.
3. New `RunWalkPackage/Sources/RunWalkFeature/Telemetry.swift` (name is free): a small service that (a) reads the key/host from Info.plist, (b) reads the user's choice (default on), (c) sets up PostHog only when allowed, with `sessionReplay = false`, `captureApplicationLifecycleEvents = true`, `captureScreenViews = true`, `errorTrackingConfig.autoCapture = true`, `personProfiles = .identifiedOnly`, and registers the super properties, (d) exposes `capture(event, properties)` that is a no-op when telemetry is off, (e) `setEnabled(_:)` that persists the choice, and on false calls `telemetry_disabled` then opts out and closes the SDK. Under `--uitesting` / UI-test launch arguments telemetry must never send (HabitFlame's `sendsTelemetry(isDebugBuild:arguments:)` shows the seam). DEBUG builds may send; that is how the device pass will be verified.
4. `Config/Shared.xcconfig` + `RunWalk/Info.plist`: `POSTHOG_API_KEY` and `POSTHOG_HOST` following HabitFlame's exact xcconfig trick for the URL slashes.
5. `SettingsTabView.swift`: a new section "Privacy" with the toggle "Share anonymous usage and crash reports" and a one-line footer: "Helps fix crashes and improve Canter. Never includes your workouts, routes, or health data." Wire it to `setEnabled`.
6. Fire the events from section 2 at their moments (phone workouts, watch-handoff workouts with `source = watch`, widgets/shortcuts via the `source` value at the phone-side start).
7. Privacy manifest: add an app-level `PrivacyInfo.xcprivacy` to the iOS app target if one does not exist, declaring collected data types that PostHog sends by default (crash data, performance data, product interaction, device ID via identifierForVendor as `$device_id`), not linked to identity, not used for tracking. Verify each claim against the posthog-ios 3.58.3 source in `.build/checkouts`, and say in the PR body which properties the SDK sends by default.
8. `ci_scripts/ci_post_xcodebuild.sh` and `scripts/test-ci-post-xcodebuild.sh`: port from HabitFlame with `MAIN_DSYM` set to the RunWalk app's dSYM name (check the product name in the project) and the same non-fatal contract. The Xcode Cloud secret `POSTHOG_CLI_API_KEY` on workflow `7CAAFDF2-188A-44D9-B1FB-43D2879B78ED` is a separate step done by the orchestrator through the App Store Connect UI; the script must degrade loudly and exit 0 when the variable is absent.
9. `PRIVACY_POLICY.md` in the repo: replace every Firebase Crashlytics statement with the PostHog statement (see section 5). Keep it consistent with the legal repo text.
10. Update `CLAUDE.md` (project) and any doc that states "Firebase Crashlytics" as the crash reporter.
11. Tests: RED first. Unit tests for the telemetry gate (off = never set up, UI-test args = never send, persisted choice round trip), for the super properties, and for each event's property builder. Mutation-test at least the gate: flip the default and prove a test fails.
12. Gates: `swift build`, the package tests, the app scheme build for the 15 Pro Max destination. Then the device pass (section 6).

## 4. Android (RunWalk-Android, worktree `/Users/vishutdhar/Code/wt-runwalk-android-posthog`, branch `vd/20260925-posthog`; `local.properties` already copied in)

1. `gradle/libs.versions.toml` + `app/build.gradle.kts`: `com.posthog:posthog-android` pinned exact to `3.71.0` (latest as of today; verify the coordinate and that it resolves). Add the PostHog Gradle plugin `com.posthog.android` for R8 mapping upload, configured so that it runs only when `POSTHOG_CLI_API_KEY` is present in the environment or `local.properties` (project id 326535, host `https://us.posthog.com`), and is a no-op with a printed warning otherwise. Never commit the key.
2. `RunWalkApplication.onCreate`: set up PostHog first, gated on the persisted user choice (default on) exactly like iOS: `sessionReplay = false`, `captureApplicationLifecycleEvents = true`, `captureScreenViews = true`, `errorTrackingConfig.autoCapture = true` (verify the exact 3.71.0 API name), `personProfiles = NEVER` is NOT wanted (identifiedOnly, matching iOS), register the super properties. A `Telemetry` object (name is free) with `capture`, `setEnabled`, and the same off contract as iOS; off = `PostHog.optOut()` + no setup on the next launch.
3. `UserPreferences`: a new boolean key `telemetry_enabled`, default true.
4. Settings screen: the same "Privacy" section, same switch text and footer as iOS.
5. Fire the section 2 events from the Android code paths (timer service start/finish, history save, program screens, settings toggles, review prompt). Strava does not exist on Android: skip those two.
6. `BuildHygieneTest`: keep every Firebase pin. Rewrite the comments that say "no analytics SDK and no crash reporter" to the new truth. Add tests pinning: session replay off in the config, the telemetry preference defaults to true, and the PostHog dependency is pinned exact.
7. Docs: `README.md`, `CLAUDE.md`, `ROADMAP.md`, `docs/marketing/play/data-safety.md`, `docs/marketing/play/declarations.md`: PostHog is now a third-party SDK; add its Data Safety rows (crash logs, diagnostics, app interactions, device or other IDs; collected, shared with PostHog as a processor, not ephemeral, optional because of the switch, purposes analytics + app functionality). Source each row from PostHog's Android SDK data disclosure; if PostHog publishes no Play disclosure page, say so and derive the rows from the SDK source (what `PostHogAndroidConfig` sends by default).
8. Tests RED first, same set as iOS. Gates: `./gradlew :app:testDebugUnitTest` and `:app:assembleRelease` (proves the plugin no-ops without a key). Do NOT run `bundleRelease` uploads.
9. Device pass on the Pixel 10 Pro XL (adb serial `59200DLCQ0082C`) if it is connected: install the debug build, toggle the switch off and on, run a 1-minute workout, confirm the events land (the orchestrator queries PostHog). If the Pixel is not connected, say so and stop at merge-ready.

## 5. Privacy policy (runwalk-legal, worktree `/Users/vishutdhar/Code/wt-runwalk-legal-posthog`, branch `vd/20260925-posthog-policy`)

Rewrite `privacy-policy.html` so it is true for the builds described above, effective today, September 25, 2026. Keep the document's existing voice: plain, exact, no marketing. Specifically:

- The "In short" paragraph: the App sends anonymous usage events and crash reports to PostHog on both iOS and Android, with a switch in Settings to turn it off; it still never sends workouts, routes, location, health data, Strava identity, or settings content.
- Replace the Firebase Crashlytics paragraphs with a PostHog paragraph: what a usage event carries (event name, the screen, coarse numbers such as workout duration and distance, app version, device model, OS version, an anonymous per-install identifier PostHog assigns, IP address at the connection level which PostHog is configured to discard), what a crash report carries (stack trace, the same device facts), where it is processed (PostHog Inc., US cloud, under its privacy policy and DPA, link https://posthog.com/privacy), retention (PostHog's defaults for the plan: state them from the pricing/docs page, 1 year for events on this plan is what the billing page says; verify), how to turn it off (Settings > Privacy > Share anonymous usage and crash reports), and that turning it off stops sending immediately and that reports already sent cannot be matched to a person and so cannot be located for deletion; support email for questions.
- Firebase App Check stays in the iOS third-party list (unchanged wording). Firebase Crashlytics and Firebase Analytics leave the list. Android's "no reporting of its own" sentences change to the PostHog truth.
- Update the "Data the App Does NOT Collect" list and the Third-Party Services list accordingly.
- Update `index.html` only if it summarizes data practices.
- Contact email stays support@freedom-terminal.com.

Also mirror the same facts into `PRIVACY_POLICY.md` in the RunWalk repo (the iOS implementer does that file; keep the wording aligned by reading this section).

## 6. Verification the orchestrator runs after the three branches land

- iPhone 15 Pro Max (`00008130-0001084011E8001C`): install the debug build, drive Settings (toggle off, on), run a 1-minute workout, then query PostHog (`execute-sql` over events where `properties.app = 'canter'`) for `$screen`, `workout_started`, `workout_completed`, `setting_changed`, `telemetry_disabled`.
- Force a test crash on a debug build only behind a launch argument, confirm an `$exception` issue appears symbolicated after the dSYM upload.
- Pixel: same events.
- adversarial review round on each branch until clean, then PR via `quick-pr`, one cloud review each, merge.
- Follow-ups owned by the orchestrator, not the implementers: Xcode Cloud secret, PostHog GitHub app on the two repos + crash alert, Play Data Safety form, ASC App Privacy labels, PostHog project dashboard "Canter funnel".

## 7. User feedback (owner, September 25: "we need to make sure people are using our app and we are getting feedback from users")

Phase 2 of this arc, dispatched after phase 1 merges: an in-app feedback survey through PostHog Surveys (free tier 1,500 responses a month), shown once after the third completed workout: "What would make Canter better for you?" (free text) plus a 1 to 5 "How likely are you to keep using Canter?" question. The survey is defined in PostHog, targeted by `app = canter`, and rendered by the SDK's survey support (verify the posthog-ios 3.58.3 and posthog-android 3.71.0 survey APIs before promising; if a pin lacks surveys, the bump is the first task of phase 2).

Consequence for phase 1: the privacy policy (section 5) must already state that the App may show an optional in-app feedback survey, that answers are sent to PostHog with the same anonymous identifier, that free-text answers are read by the developer, and that users should not put personal details in them. Phase 1 code does not ship the survey; the policy is written once.
