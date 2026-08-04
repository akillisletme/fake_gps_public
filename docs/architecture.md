# Architecture

> **Map Tools: Fake GPS & Tracker** is a production Flutter application with a
> layered, scalable architecture built around the BLoC pattern. Every layer is
> designed for maintainability, testability and separation of concerns.

> This document is a high-level technical overview of the shipped app. It is not
> a complete source listing — internal keys, wire identifiers and billing
> internals are intentionally left out.

---

## Technology Stack

```
Flutter (Dart)
├── BLoC / Cubit          — feature-scoped state management across all modules
├── Drift (SQLite ORM)    — type-safe local database with reactive streams
├── GetIt                 — service locator for dependency injection
├── GoRouter              — declarative navigation with deep link support
├── Freezed               — immutable state & data models with sealed unions
├── EasyLocalization      — 20-language runtime locale switching
└── Google Maps Flutter   — map rendering, markers, polylines, heatmaps

Native (Kotlin / Android)
├── MockLocationService          — ForegroundService, dual-provider GPS injection
├── RecordingForegroundService   — keeps walk recording alive with the screen off
├── SchedulerAlarmReceiver       — AlarmManager + BOOT_COMPLETED rescheduling
├── SmartShieldMonitorService    — UsageStats-based foreground app detection
├── HomeWidget                   — AppWidget entry point for background activation
├── OverlayService               — system overlay window for out-of-app control
└── MainActivity                 — MethodChannel bridge to Flutter
```

---

## Architectural Layers

The application follows a **feature-first** layout with a shared product layer:

```
lib/
├── product/                  — shared infrastructure
│   ├── init/                 — app bootstrap, global provider tree, localization
│   ├── db/                   — ALL persistence lives here
│   │   ├── tables/           — Drift table definitions
│   │   ├── repositories/     — abstract contracts + Drift implementations
│   │   └── preferences/      — SharedPreferences facade, split into mixins
│   ├── service/              — service locator + platform/service wrappers
│   ├── subscription/         — billing integration (RevenueCat)
│   ├── models/               — Freezed domain models
│   ├── navigation/           — GoRouter routes and transitions
│   ├── theme/                — theming and user accent colour
│   ├── utils/                — geo maths, parsers, unit system, simplifiers
│   └── widgets/              — shared UI building blocks
│
└── future/                   — feature modules
    ├── login/                — onboarding, splash, terms
    ├── map/                  — the map screen (mode-based, see below)
    └── settings/             — settings tree with its own inner Navigator
```

---

## The Map Screen — Three-Layer Mode Architecture

The map is the heart of the app, and its structure is the main architectural
decision in the project:

| Layer | What it selects | Owner |
|---|---|---|
| **AppMode** | top-level mode: FakeGPS ⇄ Tools ⇄ Tracker | `AppModeCubit` |
| **MapMode** | FakeGPS sub-mode: fixed / joystick / route / human | `MapCoordinatorCubit` |
| **MapToolId** | which of the 10 measurement tools is active | `MapToolsCubit` |

The screen shell iterates over the registered top-level modes and asks each
mode's **presenter** for its UI slots (FAB, info pill, portrait overlay,
landscape rail, help content). Presenters are wired in a small registry, so
**adding a new top-level mode is a presenter class plus one registry line** — the
map shell itself is never touched.

Each measurement tool implements a single `MapTool` contract (map taps, markers,
polylines, result, optional live controls and point dragging), so a new tool is
one file plus one enum entry.

### Coordinator Pattern

`MapCoordinatorCubit` orchestrates the simulation modes and enforces mutual
exclusion — only one mode can be active at a time. Mode transitions are
serialised behind a lock so that a slow cleanup can never overlap with the next
activation, and the native service always receives clean, non-conflicting
updates.

Starting a simulation goes through a single gate that checks, in order: location
permission → mock-location setup → usage allowance. Ordering matters: a failed
attempt must not consume the user's daily allowance.

---

## State Management — BLoC Tree

```
EasyLocalization
└── StateInitialize
    └── MultiBlocProvider
        ├── ThemeCubit                   # Material You, custom colour, contrast, AMOLED
        ├── SubscriptionCubit            # subscription state
        ├── SavedItemsCubit              # 5 Drift repositories, watch() streams
        ├── MapCoordinatorCubit          # orchestrates all simulation modes
        ├── AppModeCubit                 # FakeGPS ⇄ Tools ⇄ Tracker
        └── Builder
            ├── BasicMockCubit           # fixed location
            ├── JoystickCubit            # real-time directional control
            ├── RouteCubit               # 2–10 point route simulation
            ├── ProRouteCubit            # per-waypoint speed / altitude / dwell
            ├── HumanSimulationCubit     # natural movement within a radius
            ├── MapToolsCubit            # measurement tool state machine
            ├── RouteRecordingCubit      # real GPS track recording
            ├── WelcomeCubit
            └── SplashCubit              # RemoteConfig-driven version gate
```

Module cubits are attached to the coordinator **before the first frame**, which
closes the race window between a mode switch and native state sync.

---

## How Mock Location Works

The native `MockLocationService` (Android `ForegroundService`) injects fake
coordinates into **both** Android location stacks:

| Provider | Used by |
|---|---|
| **FusedLocationProviderClient** | Google Maps, modern apps, Play Services |
| **LocationManager** (GPS + Network) | Legacy apps, AOSP location stack |

This dual-injection approach gives system-wide compatibility without root
access. The service is the single owner of the active mock: whichever entry
point starts it (app, widget, overlay, alarm or Smart Shield), the state lives
in one place and the UI re-syncs from it.

---

## Native ↔ Flutter Communication

All platform-specific work is bridged over `MethodChannel`, with a thin Dart
wrapper per channel so platform calls never leak into the UI layer:

| Bridge | Direction | Purpose |
|---|---|---|
| Mock location | Flutter → Native | start / update / stop mock GPS, read current status |
| Scheduler | Flutter → Native | set and cancel `AlarmManager` triggers |
| Smart Shield | Flutter → Native | usage-access check, start/stop the monitor |
| Walk recording | Flutter → Native | start / pause / stop the recording service |
| Notifications | Flutter → Native | native notifications shown while Flutter is paused |
| System settings | Flutter → Native | open the relevant Android settings screens |
| Route import | Native → Flutter | hand over a GPX/KML/TCX file opened from outside |

The walk-recording bridge is deliberately **advisory**: if the native call
fails, the failure is swallowed and recording continues. A notification problem
must never cost the user their track.

---

## Background & Out-of-App Entry Points

A key architectural challenge was activating mock location **without the app
being open**. Four independent entry points handle this:

```
Home Widget (AppWidget)          → MockLocationService.start(slot)
Floating Overlay (system window) → MockLocationService.start(slot)
Scheduled Alarm (AlarmManager)   → SchedulerAlarmReceiver → MockLocationService.start()
Smart Shield (UsageStats poll)   → foreground app matches a rule → start / stop
```

- The alarm receiver runs **without a Flutter engine** and reschedules recurring
  alarms itself; a `BOOT_COMPLETED` receiver restores them after a reboot
- Smart Shield tracks whether it started the current mock, so it can stop its own
  session while **never** killing a mock the user started manually

---

## Walk Recording

Recording real GPS is a separate lifecycle from playing back a simulation:

- A foreground service plus a partial wake lock keeps sampling alive with the
  screen off; the location stream itself stays in Dart
- Points are flushed to a draft table on a balanced schedule, so a crash mid-walk
  leaves a recoverable draft rather than nothing
- Pause time is compressed out of the timestamps, so it never inflates speed
- Recording is blocked while a mock is active — otherwise the "real" track would
  be the fake one
- The drawn track is split into a **settled** and a **live** polyline. The
  settled part is value-equal between frames, so it is not re-serialised to the
  platform channel on every sample — this keeps channel traffic flat on long
  recordings instead of growing with track length
- The playback path reuses the existing advanced-route engine

---

## Navigation Flow

Declarative routing via **GoRouter**, with deep link support (`fakegps://`):

```
Splash  (RemoteConfig version check)
  ├── updateRequired  → UpdateRequiredView   (changelog from Firestore, Play Store CTA)
  ├── first run       → Onboarding → Setup Guide → MapView
  └── returning user  → MapView
                          ├── FavoritesDrawer
                          └── SettingsView (inner Navigator)
```

Most settings sub-pages are pushed onto the settings screen's **inner
Navigator** rather than the global router, which keeps the settings background
layer stationary while pages slide over it.

---

## Data Persistence

| Data | Storage | Strategy |
|---|---|---|
| Favourites, routes, recordings, schedules | **Drift** (SQLite) | Repository pattern, reactive `watch()` streams, versioned migrations |
| Theme, language, units, map settings | **SharedPreferences** | Read through a mixin-composed facade |
| Data the native side also reads | **SharedPreferences** | Deliberate: native services cannot read the SQLite layer |
| Remote flags & minimum version | **Firebase Remote Config** | Fetched on splash, cached locally |

List-shaped fields (waypoints, repeat days) are stored as JSON text columns, and
every record is an independent row — one corrupt record cannot take the rest of
the collection with it.

---

## Units System

A dedicated unit layer supports **metric, imperial and nautical** systems plus a
set of area units. The formatter is a **pure function** that takes the preference
as a parameter, so it is testable without a widget tree, and a small environment
reader lets the map tools (which have no `BuildContext`) read the current
preference. Changing the unit re-renders live tool results immediately.

---

## Firebase Infrastructure

### Firebase Crashlytics — Error Monitoring

Active in production builds; disabled in debug. Uncaught Dart exceptions,
Flutter framework errors and native Kotlin crashes are all captured.

```dart
// main.dart
FlutterError.onError = FirebaseCrashlytics.instance.recordFlutterFatalError;

PlatformDispatcher.instance.onError = (error, stack) {
  FirebaseCrashlytics.instance.recordError(error, stack, fatal: true);
  return true;
};
```

### Firebase Remote Config — Version Gate & Feature Flags

Drives the splash version check and lets configuration change without an app
update. If the minimum required version is above the installed one, the user is
routed to the update screen and blocked until they update via Play Store.

### Cloud Firestore — Dynamic Content

Serves per-language changelog content, so release notes can be updated
server-side without shipping a new build.

---

## Testing

Roughly **400 tests** cover everything that does not need a device: cubits,
Drift repositories and migrations, the measurement tools, geo maths, the unit
formatter, translation parity across all 20 languages, and selected widget
layouts.

Testability patterns used throughout:

- Cubits take the outside world through the constructor (location stream,
  permissions, clock, platform channels), so tests inject fakes
- Pure calculation layers are kept separate from UI and tested directly
- Widget tests run without translations loaded, so keys render as their own
  names — conveniently the worst case for layout overflow
- Narrow-layout regressions are pinned with fixed-width widget tests

---

## Flutter Dependencies

> App version: **2.4.0+68** · Dart SDK: `^3.10.7`

### Production Dependencies

| Category | Package | Version |
|---|---|---|
| **Maps & Location** | google_maps_flutter | ^2.14.0 |
| | geolocator | ^14.0.2 |
| | geocoding | ^4.0.0 |
| | flutter_compass | ^0.8.1 |
| | permission_handler | ^12.0.1 |
| **State Management** | flutter_bloc | ^9.1.0 |
| **Dependency Injection** | get_it | ^8.0.3 |
| **Navigation** | go_router | ^17.1.0 |
| | app_links | ^6.4.0 |
| **Models** | freezed_annotation | ^3.0.0 |
| | json_annotation | ^4.9.0 |
| **Firebase** | firebase_core | ^4.4.0 |
| | firebase_remote_config | ^6.1.4 |
| | firebase_crashlytics | ^5.0.7 |
| | cloud_firestore | ^6.0.2 |
| **Billing** | purchases_flutter | ^10.4.1 |
| **Localization** | easy_localization | ^3.0.7 |
| **Storage** | shared_preferences | ^2.5.4 |
| | drift | ^2.28.0 |
| | sqlite3_flutter_libs | 0.5.32 |
| | path_provider | ^2.1.5 |
| **Connectivity** | connectivity_plus | ^6.1.0 |
| **Photo GPS Editor** | image_picker | ^1.1.2 |
| | native_exif | ^0.6.2 |
| | gal | ^2.3.0 |
| **File Import (GPX/KML/TCX)** | file_picker | ^8.1.2 |
| | xml | ^6.5.0 |
| **Smart Shield** | installed_apps | ^1.6.0 |
| **UI** | lottie | ^3.3.1 |
| | font_awesome_flutter | ^11.0.0 |
| | google_fonts | ^8.1.0 |
| **Utils** | url_launcher | ^6.3.1 |
| | share_plus | ^12.0.1 |
| | uuid | ^4.5.3 |
| | package_info_plus | ^9.0.0 |
| | vector_math | ^2.2.0 |

### Dev Dependencies

| Category | Package | Version |
|---|---|---|
| **Linting** | very_good_analysis | ^10.1.0 |
| **Testing** | bloc_test | ^10.0.0 |
| | mocktail | ^1.0.4 |
| | fake_async | ^1.3.1 |
| | sqlite3 | ^2.4.0 |
| **Code Generation** | build_runner | ^2.7.0 |
| | freezed | ^3.0.0 |
| | json_serializable | ^6.9.4 |
| | drift_dev | >=2.28.0 <2.32.0 |
| **Build** | flutter_launcher_icons | ^0.14.4 |
| | flutter_native_splash | ^2.4.7 |
