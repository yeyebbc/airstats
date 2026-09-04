# macOS Sonoma dependency analysis

Research and source audit date: 2026-09-04.

## Executive summary

AirStats currently requires macOS 14 Sonoma in both release metadata and source code. The release floor is explicit in three machine-readable locations. The substantive source dependency is concentrated in Swift's Observation framework and two SwiftUI API families introduced in macOS 14. The metric collectors themselves do not have an identified Sonoma-only dependency.

This is a static source and API-availability analysis. It identifies compile-time blockers to a macOS 13 deployment target and runtime areas that need validation; it is not evidence that the current binary runs on Ventura.

## Project architecture

AirStats is a Swift Package Manager application with three product layers:

| Layer | Responsibility | Platform surface |
| --- | --- | --- |
| `Sources/AirStatKit` | Nine collectors, scheduling, snapshots/history, settings persistence, formatting | Foundation, Observation, Mach, `sysctl`, `libproc`, IOKit, CoreWLAN, Metal; deliberately no AppKit or SwiftUI |
| `Sources/AirStatUI` | Menu bar rendering, panel, desktop widget, settings, notifications, login item, Sparkle updates | AppKit, SwiftUI, Observation, ServiceManagement, UserNotifications, Sparkle |
| `Sources/AirStats` | Process entry point, app lifecycle, dependency wiring, probe/render CLI modes | AppKit plus the two internal libraries |

`AppCoordinator.start()` creates and starts the core objects and UI controllers. `MetricsEngine.start()` creates `SamplingCore`, applies settings, starts collection, and observes settings and power state. `SamplingCore` owns the nine collectors on one private serial queue and publishes changed snapshots to the main actor. The menu bar, panel, desktop widget, threshold monitor, and update announcer consume that state.

## Hard macOS 14 release gates

These gates prevent a Ventura build or installation even before source availability is considered:

| File | Gate | Effect |
| --- | --- | --- |
| `Package.swift:6` | `.macOS(.v14)` | SwiftPM deployment target is macOS 14. |
| `Resources/Info.plist:25-26` | `LSMinimumSystemVersion` = `14.0` | Launch Services rejects the assembled app below macOS 14. |
| `Scripts/appcast.py:39,94` | `MINIMUM_SYSTEM_VERSION = "14.0"` | New Sparkle feed items are marked as requiring macOS 14. |

Human-facing requirements repeat the same floor in `README.md:26-29` and `CONTRIBUTING.md:6-8`.

All three machine-readable gates must describe the same minimum version. Lowering only one is unsafe: for example, a Ventura-compatible binary with a `14.0` plist still cannot launch, while a feed item set lower than the actual binary can let Sparkle install an update that the host cannot open.

## Confirmed Sonoma-only source dependencies

### 1. Observation framework

Apple marks the [Observation framework](https://developer.apple.com/documentation/observation) as available from macOS 14. AirStats uses all three relevant pieces: `@Observable`, `@ObservationIgnored`, and `withObservationTracking`.

Six reference types use `@Observable`:

| Type | File | State consumed by |
| --- | --- | --- |
| `MetricsEngine` | `Sources/AirStatKit/Core/MetricsEngine.swift:9-11` | SwiftUI metric surfaces; menu bar renderer; threshold evaluation |
| `SettingsStore` | `Sources/AirStatKit/Settings/SettingsStore.swift:9-13` | All settings panes and render surfaces; collector configuration; widget and hot-key synchronization |
| `SoftwareUpdater` | `Sources/AirStatUI/Support/SoftwareUpdater.swift:52-59` | General/About settings, panel update row, update announcer |
| `NotificationAuthority` | `Sources/AirStatUI/Support/NotificationAuthority.swift:19-32` | Notification settings and update announcement |
| `PanelLayoutState` | `Sources/AirStatUI/Panel/PanelRootView.swift:9-14` | Panel disclosure state and animation |
| `DesktopWidgetLayout` | `Sources/AirStatUI/DesktopWidget/DesktopWidgetRootView.swift:10-18` | Live widget width and click-through escape-hatch state |

`@ObservationIgnored` excludes controller resources in `SoftwareUpdater` and the callback in `PanelLayoutState` from tracking.

`ObservedChanges` in `Sources/AirStatKit/Core/ObservedChanges.swift` wraps `withObservationTracking` as a continuously re-armed async sequence. It is not incidental infrastructure: six live behavior paths depend on it.

| Consumer function | Tracked state | Behavior triggered |
| --- | --- | --- |
| `MetricsEngine.beginObservingSettings()` | `SettingsStore.revision` | Re-applies cadence, enabled collectors, public-IP lookup, power behavior, and history capacity. |
| `StatusItemController.beginRendering()` | Engine snapshot and settings revision | Rebuilds and redraws menu bar readouts. |
| `DesktopWidgetController.beginObservingSettings()` | Settings revision | Shows/hides and reconfigures the desktop widget. |
| `GlobalHotKeyCenter.beginObservingSettings()` | Settings revision | Reconciles system hot-key registrations. |
| `ThresholdMonitor.start()` | Engine snapshot and settings revision | Re-evaluates notification thresholds. |
| `UpdateAnnouncer.start()` | Pending update, automatic-install state, notification authority | Announces an actionable update once permission and state allow it. |

SwiftUI also observes the six models implicitly when their properties are read from a view body. Consequently, replacing only `ObservedChanges` would not remove the macOS 14 requirement; view-model observation must migrate too.

### 2. SwiftUI `onChange(of:initial:_:)`

Apple marks the current [`onChange(of:initial:_:)`](https://developer.apple.com/documentation/swiftui/view/onchange%28of%3Ainitial%3A_%3A%29) overload as macOS 14+. AirStats has seven two-parameter-closure call sites:

| File | Sites | Uses `initial: true` | Uses old value |
| --- | ---: | ---: | ---: |
| `Sources/AirStatUI/Settings/AppearancePane.swift` | 2 | 0 | 0 |
| `Sources/AirStatUI/Settings/ColorEditor.swift` | 2 | 0 | 0 |
| `Sources/AirStatUI/Settings/DesktopWidgetPreview.swift` | 1 | 1 | 0 |
| `Sources/AirStatUI/Settings/SettingsSupport.swift` | 1 | 1 | 0 |
| `Sources/AirStatUI/Settings/SettingsWindowController.swift` | 1 | 0 | 0 |

The five non-initial sites can use Ventura's older [`onChange(of:perform:)`](https://developer.apple.com/documentation/swiftui/view/onchange%28of%3Aperform%3A%29) overload because every closure uses only the new value. The two initial sites need an explicit initial measurement, normally `onAppear`, plus the older change callback.

### 3. SwiftUI `defaultScrollAnchor(_:)`

`PanelRootView` uses `.defaultScrollAnchor(.top)` at `Sources/AirStatUI/Panel/PanelRootView.swift:82`. Apple marks [`defaultScrollAnchor(_:)`](https://developer.apple.com/documentation/swiftui/view/defaultscrollanchor%28_%3A%29) as macOS 14+.

The modifier keeps the panel list top-aligned initially and during content-size changes. Removing it may be sufficient because top is the natural vertical starting position, but the disclosure animation and maximum-height scrolling behavior must be checked on both Ventura and Sonoma. If removal changes behavior, the modifier needs a macOS 14 availability wrapper or an AppKit scroll-position fallback.

## APIs inspected that do not require Sonoma

These relatively recent calls are already available on Ventura:

| API | Use in AirStats | Apple minimum |
| --- | --- | ---: |
| [`scrollBounceBehavior(_:axes:)`](https://developer.apple.com/documentation/swiftui/view/scrollbouncebehavior%28_%3Aaxes%3A%29) | Panel scrolling | macOS 13.3 |
| [`scrollIndicators(_:axes:)`](https://developer.apple.com/documentation/swiftui/view/scrollindicators%28_%3Aaxes%3A%29) | Panel and preview scrolling | macOS 13.0 |
| [`contentTransition(_:)`](https://developer.apple.com/documentation/swiftui/view/contenttransition%28_%3A%29) | Numeric transitions in metric views | macOS 13.0 |
| [`SMAppService`](https://developer.apple.com/documentation/servicemanagement/smappservice) | Launch at login | macOS 13.0 |
| [`ProcessInfo.isLowPowerModeEnabled`](https://developer.apple.com/documentation/foundation/processinfo/islowpowermodeenabled) | Collector state and sampling pause | macOS 12.0 |
| [`ProcessInfo.thermalState`](https://developer.apple.com/documentation/foundation/processinfo/thermalstate-swift.property) | Public thermal-pressure reading | macOS 10.10.3 |
| [`MTLCopyAllDevices()`](https://developer.apple.com/documentation/metal/mtlcopyalldevices%28%29) | GPU enumeration | macOS 10.11 |
| [`CGDirectDisplayCopyCurrentMetalDevice(_:)`](https://developer.apple.com/documentation/coregraphics/cgdirectdisplaycopycurrentmetaldevice%28_%3A%29) | Display-driving GPU selection | macOS 10.11 |
| [`UNUserNotificationCenter`](https://developer.apple.com/documentation/usernotifications/unusernotificationcenter) | Threshold and update notifications | macOS 10.14 |

The package resolves Sparkle 2.9.5 (`Package.resolved:5-10`). Sparkle's 2.9.5 package manifest declares macOS 10.13, so the update framework is not a Ventura deployment blocker.

## Collector-specific runtime risk

No collector imports Observation, SwiftUI, or AppKit, and the static API scan found no collector symbol introduced in Sonoma. The collector stack is therefore not a known compile-time reason to require macOS 14.

That does not establish equivalent runtime behavior. Two areas depend on undocumented data contracts rather than OS-versioned public API:

- `ThermalCollector` uses public IOKit transport with the undocumented AppleSMC request layout, command codes, and sensor keys. It fails closed when the service, key directory, or readings are unavailable.
- `GPUCollector`, `DiskCollector`, `PowerCollector`, and parts of `SystemInfoCollector` read IORegistry properties whose key names and contents can vary by driver, hardware, and OS release. `GPUCollector` explicitly treats performance-counter names as non-contractual.

These are hardware/driver coverage risks, not evidence of Sonoma-only behavior. A Ventura compatibility claim still requires running `AirStats --probe` on representative Ventura Apple-silicon machines and comparing CPU, memory, network, disk, power, GPU, and thermal output with system tools.

## Dependency boundary

The Sonoma requirement is concentrated above the sampling core:

```text
Package/plist/appcast floor (macOS 14)
                 |
       AppCoordinator.start()
                 |
     +-----------+--------------------+
     |                                |
Observation-driven models        SwiftUI 14-only modifiers
     |                                |
ObservedChanges + SwiftUI        onChange(initial:) and
view invalidation                defaultScrollAnchor
     |
UI refresh, settings sync, thresholds, hot keys, updates

SamplingCore + collectors
(no identified Sonoma-only API; Ventura runtime validation required)
```

## Analysis limits

- Static source inspection covered `Package.swift`, release metadata, all source references to the identified API families, framework imports, and collector platform calls.
- Apple documentation provides API availability evidence, not proof of AirStats behavior on Ventura.
- The current workstation has Swift 5.8.1 targeting x86_64, not the Swift 6 toolchain required by this package. Its selected Command Line Tools installation also fails to resolve the macOS SDK `PlatformPath`, so `swift build` exits before compilation. A lowered-target build, Ventura Apple-silicon app launch, UI render, and collector probe cannot be performed here.

The implementation and validation plan is in [Ventura compatibility feasibility](ventura-compatibility.md).
