# Ventura-compatible build: pros, cons, and feasibility

Research date: 2026-09-04. This assessment is based on the source inventory in [macOS Sonoma dependency analysis](sonoma-dependencies.md), Apple API documentation, and the project's resolved third-party dependency.

## Decision

**Technically feasible, with a moderate cross-cutting observation refactor and mandatory macOS runtime validation.**

There is no identified Ventura-incompatible collector or third-party dependency. The blockers are concentrated and replaceable:

1. three release-version gates fixed at macOS 14;
2. Swift Observation, which is macOS 14+ and drives both SwiftUI invalidation and six non-view reaction loops;
3. seven SwiftUI `onChange` calls using the macOS 14 overload;
4. one `defaultScrollAnchor` modifier introduced in macOS 14.

This is not a one-line deployment-target change. Lowering the version gates before replacing the APIs will produce availability errors, and weakening observation can produce a build that launches but silently stops refreshing settings, metrics, alerts, hot keys, or update notices.

## Pros and cons

| Consideration | Benefits | Costs and risks |
| --- | --- | --- |
| User reach | Supports users and managed fleets pinned to Ventura. | Adds OS-version reach only. Under AirStats' Apple-silicon requirement, every Ventura-capable Mac can also run Sonoma. |
| Engineering scope | The blockers are known and localized; collectors and Sparkle have no identified compile-time incompatibility. | The Observation migration crosses six models and six live reaction paths. A mistake can cause silent stale UI, settings, alerts, hot keys, or update state rather than an obvious crash. |
| Product architecture | A native Combine cutover can produce one code path and one binary for macOS 13.3 and later. | `ObservableObject` invalidation is broader than Observation and may increase redraws. Post-commit notification semantics and cancellation must be designed explicitly. |
| Release operations | One unified 13.3+ artifact can use the existing signing, notarization, distribution, and Sparkle flow after metadata is aligned. | A separate Ventura artifact would add feed selection, signing/notarization, regression testing, support triage, and release-retention complexity without avoiding the source migration. |
| Tooling | Xcode 16 can compile the current Swift 6 package for Ventura from a supported newer macOS host. | Ventura cannot host Xcode 16, so contributors on Ventura cannot build the current package with Apple's supported toolchain. |
| Compatibility floor | Keeping `scrollBounceBehavior` allows a focused migration with a truthful macOS 13.3 minimum. | This does not support Ventura 13.0 through 13.2. Supporting those releases requires another fallback and test path. |
| Platform longevity | A compatibility build can provide continuity while users complete OS upgrades. | Ventura's compatibility page is archived and its last listed OS security release is 13.7.8 from 2025-08-20. Supporting it increases security and support exposure. This is an inference from Apple's release record, not an Apple EOL declaration. |

## Build strategy decision

| Option | Assessment |
| --- | --- |
| Keep macOS 14+ | Lowest engineering and support cost. Prefer this when there is no measured Ventura demand. |
| Ship one Apple-silicon build for macOS 13.3+ | **Recommended if demand justifies the migration.** It keeps one implementation, artifact, update feed, and release process for Ventura and newer systems. |
| Ship a separate Ventura build | **Not recommended.** The source still has to stop depending unconditionally on macOS 14 APIs, while a second artifact or branch doubles operational and behavioral divergence. It is justified only if an external distribution constraint makes a unified binary impossible; none was identified. |

The decision is therefore conditional, not purely technical: retain macOS 14 unless a user count, fleet commitment, or support obligation pays for the migration and ongoing Ventura test matrix. If that threshold is met, lower the main build to 13.3 rather than creating a legacy edition.

## Toolchain feasibility

The current package requires the Swift 6 toolchain and compiles its targets in Swift 5 language mode. Apple's [Xcode system requirements](https://developer.apple.com/xcode/system-requirements) list:

- Xcode 16 through 16.4: macOS deployment targets 10.13 through 15, which includes Ventura 13;
- Xcode 16.0 through 16.2: Swift 6.0 compiler;
- Xcode 16.0 through 16.2: Sonoma 14.5 or later as the Xcode host OS.

Therefore a Sonoma build machine can use the existing Swift 6 toolchain to emit a Ventura-targeted app. Ventura cannot itself host Xcode 16, so “runs on Ventura” and “can be built on Ventura” must remain separate claims. The practical baseline is macOS 13.3 rather than 13.0 because AirStats uses `scrollBounceBehavior`, which Apple marks as macOS 13.3+.

Sparkle is not a blocker. `Package.resolved` pins 2.9.5, whose package manifest declares macOS 10.13.

## Recommended implementation

### 1. Set one truthful deployment floor

After source migration, set the minimum to **macOS 13.3** in all release paths:

- `Package.swift`: use SwiftPM's custom-version form `.macOS("13.3")`. The typed `.macOS(.v13)` constant means 13.0 and would advertise a lower floor than the unguarded 13.3 API.
- `Resources/Info.plist`: `LSMinimumSystemVersion` to `13.3`.
- `Scripts/appcast.py`: generated `sparkle:minimumSystemVersion` to `13.3` for the first compatible release and later releases.
- `README.md`: runtime requirement to macOS 13.3 or later.
- `CONTRIBUTING.md`: keep the Xcode host requirement separate from the app deployment target.

Historical appcast entries must retain their original minimum version. Only releases built and tested for Ventura should advertise 13.3.

An alternative is to guard or replace `scrollBounceBehavior` and support macOS 13.0. That buys only the first three Ventura point releases and adds another conditional path; 13.3 is the safer, smaller support contract.

### 2. Replace Observation as one clean cutover

Recommended: use native Combine `ObservableObject`/`@Published` for the six models and Combine subscriptions for non-view reactions. This adds no dependency and is available throughout Ventura.

Required work:

- Convert `MetricsEngine`, `SettingsStore`, `SoftwareUpdater`, `NotificationAuthority`, `PanelLayoutState`, and `DesktopWidgetLayout` from `@Observable` to `ObservableObject`.
- Publish exactly the state SwiftUI reads. Preserve `private(set)` and main-actor confinement.
- Mark externally owned models in SwiftUI with `@ObservedObject` at the ownership boundary. Models are created by `AppCoordinator`/controllers, so `@StateObject` would claim the wrong ownership.
- Restructure optional live dependencies in renderable settings views. `Optional<ObservableObject>` cannot directly be an `@ObservedObject`; use a conditional child view with a non-optional observed model, or pass an immutable render model for the nil/offscreen path.
- Replace `ObservedChanges` with explicit `AnyCancellable` subscriptions for the six consumer paths, then remove `Sources/AirStatKit/Core/ObservedChanges.swift` and its Observation-specific tests/comments.
- Give non-view consumers a post-commit publisher or revision. Combine's `@Published` emits in `willSet`; a sink that re-reads the object can see old state, and a snapshot sink can run before `MetricsEngine.ingest(_:)` has recorded history. Emit only after each logical mutation is complete, or deliberately schedule the reaction onto the next main-actor turn.
- Retain the current semantics: ignore identical settings writes; react after the new value is stored; run on the main actor; avoid retaining the observer; cancel subscriptions in each existing stop/shutdown path.

Suggested publisher mapping:

| Current path | Combine source |
| --- | --- |
| Engine settings application | Post-commit settings revision, dropping the initial value because `start()` already calls `applySettings()` |
| Menu bar rendering | Merge a post-ingest engine revision with the post-commit settings revision |
| Desktop widget synchronization | Post-commit settings revision |
| Hot-key reconciliation | Post-commit settings revision |
| Threshold evaluation | Merge a post-ingest engine revision with the post-commit settings revision |
| Update announcement | Post-commit update/authority event, or combined new values without re-reading will-set state |

Performance risk: Observation tracks individual property reads, while `ObservableObject` invalidates every observing view when `objectWillChange` fires. Publishing too broadly can redraw unrelated settings panes or metric surfaces. Keep subscriptions and view observation at the smallest existing root boundaries, and measure the existing performance benchmarks plus idle UI activity after migration.

#### Alternative: Perception backport

Point-Free's Perception package backports the Observation model and can reduce changes to model declarations and manual tracking. Tradeoffs:

- advantage: semantics are closer to the current `@Observable`/tracking design;
- costs: a new runtime dependency, wrapper requirements in SwiftUI on pre-Sonoma systems, another macro/toolchain surface, and continued reliance on a non-Apple compatibility layer for core state propagation.

Use it only if a prototype shows that the native Combine cutover causes unacceptable view churn or complexity. Maintaining parallel Observation-on-Sonoma and Combine-on-Ventura implementations is not recommended: two state-propagation paths double the failure surface for every future property.

### 3. Back-deploy the SwiftUI calls

- Rewrite five non-initial two-argument callbacks to Ventura's `onChange(of:perform:)`; none uses the old value.
- For `DesktopWidgetPreview` height and `SettingsSupport` width, perform the initial measurement in `onAppear` and subsequent measurements in the older `onChange` overload. Verify that the first geometry value is final enough to avoid a one-frame size jump.
- Remove `.defaultScrollAnchor(.top)` first and test the panel at its maximum height while expanding/collapsing modules. If the scroll position changes, hide the Sonoma modifier behind a small availability-gated `View` extension or restore the top position through the panel's AppKit scroll view.

Do not blanket-wrap entire views in `if #available(macOS 14, *)`: that preserves two UI implementations and makes the Ventura path the less-exercised one. Back-deploy each isolated behavior.

### 4. Update tests around observable behavior

`Tests/AirStatKitTests/SettingsChangeStreamTests.swift` directly specifies `ObservedChanges` behavior and will become obsolete. Replace it only with tests for consumer-visible contracts that remain failure-prone after the migration, such as:

- one accepted settings mutation re-applies the engine configuration after the new value is stored;
- identical settings writes do not trigger redundant work;
- stopping a controller/monitor prevents later reactions;
- merged engine/settings changes update threshold and menu-bar behavior once the new state is visible.

Existing engine activity, settings, update, rendering, and torture tests should continue to run against the lowered deployment target. Do not retain tests that assert the removed helper's implementation rather than app behavior.

## Required compatibility validation

A build succeeding with `-target arm64-apple-macosx13.3` is necessary but insufficient. Run this matrix before changing the public requirement:

| Area | Ventura 13.3 or latest 13.7.x | Sonoma latest 14.x | Evidence |
| --- | --- | --- | --- |
| Compile/link/package | Required | Required | Clean `swift build`, `swift test`, and release `.app` assembly |
| Launch/lifecycle | Required | Required | Signed app opens as an accessory app; relaunch opens Settings; clean quit |
| Menu bar | Required | Required | Values update; layout changes immediately; visibility throttling recovers |
| Panel | Required | Required | Open, scroll at height cap, expand/collapse modules, no jump or wrong scroll anchor |
| Desktop widget | Required | Required | Show/hide, resize, click-through escape hatch, settings sync |
| Settings | Required | Required | Every pane refreshes after edits; color and geometry callbacks initialize correctly |
| Hot keys | Required | Required | Add/change/remove bindings without relaunch |
| Notifications | Required | Required | Permission states, threshold alert, click action, reset/stop behavior |
| Updates | Required | Required | Appcast selects only compatible releases; check UI and pending-update announcement |
| Collectors | Required on representative Apple silicon | Required regression check | `--probe` compared with `top`, `vm_stat`, `netstat -ib`, `ioreg`, and `pmset` |
| Offscreen rendering | Required | Required | `--render` completes; compare menu bar, panel, widget, and settings images |
| Sleep/wake and lock | Required | Required | Sampling suspends, rate baselines reset, UI resumes with fresh data |

Collector coverage should include at least one laptop and one desktop if both are supported. Battery/adapter keys, fan availability, SMC sensors, and IORegistry GPU counters differ by model and cannot be validated by a single machine.

## Product and security feasibility

### Hardware reach

AirStats publicly requires Apple silicon. Apple's official [Ventura compatibility list](https://support.apple.com/en-us/102861) and [Sonoma compatibility list](https://support.apple.com/en-us/105113) show that every Apple-silicon Mac capable of Ventura is also capable of Sonoma. Ventura support therefore adds no officially supported hardware under the current Apple-silicon policy. It serves users or managed fleets pinned to an older OS.

### Security support

Apple's [security releases list](https://support.apple.com/en-us/100100) records Ventura 13.7.8 on 2025-08-20 as the last Ventura OS update, while Sonoma continued receiving OS security releases through 14.8.9 on 2026-08-06 and Safari updates afterward. Apple's [Sonoma update history](https://support.apple.com/en-us/109035) describes those releases as important security fixes.

**Inference, not an Apple EOL declaration:** Ventura is no longer in the routine macOS security-update stream as of this assessment. Supporting it increases exposure to unpatched OS vulnerabilities and expands the test matrix for users who could generally upgrade the same hardware to Sonoma.

## Recommendation

Proceed only if there is a measured user or fleet requirement for Ventura. The engineering work is bounded and feasible, but the business value is OS-version reach rather than hardware reach, and it comes with a legacy-OS security/support cost. If proceeding, ship one unified Apple-silicon build with a macOS 13.3 minimum rather than a separate Ventura artifact.

If proceeding:

1. prototype the six-model Combine cutover on a branch;
2. lower the compile target to 13.3 and eliminate every availability error;
3. complete the two-OS validation matrix, especially observation-driven refresh and collectors;
4. lower plist/appcast/public requirements only after a Ventura release build passes runtime validation.

Until those checks pass, retain macOS 14 as the published and appcast minimum.

## Verification status

Completed here:

- repository-wide static search for deployment gates, Observation usage, SwiftUI 14-only call sites, availability guards, and platform framework imports;
- call-path review for all six `ObservedChanges` consumers;
- collector architecture and high-risk IOKit/SMC review;
- primary-source availability checks for the blocking and nearby APIs;
- Xcode, hardware compatibility, security-release, and Sparkle manifest research.

The current workstation has `/usr/bin/swift`, but it is Swift 5.8.1 targeting x86_64. The selected Command Line Tools installation cannot build this Swift 6 package: `swift build` exits before compilation because `xcrun` cannot resolve the SDK `PlatformPath`. Consequently, the following remain unverified:

- Swift compilation against a macOS 13.3 deployment target with Xcode 16;
- Ventura Apple-silicon app launch, UI, collector, sleep/wake, notification, login-item, and Sparkle checks.

Those unperformed checks are release blockers, not optional follow-up work.
