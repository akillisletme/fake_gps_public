# Architecture

> **Map Tools: Fake GPS & Trails** is a production-grade Flutter application with a layered, scalable architecture built around the BLoC pattern. Every layer is designed for maintainability, testability, and separation of concerns.

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
└── Google Maps Flutter   — map rendering, markers, polylines, area overlays

Native (Kotlin / Android)
├── MockLocationService      — ForegroundService, dual-provider GPS injection
├── SchedulerAlarmReceiver   — BroadcastReceiver + BOOT_COMPLETED rescheduling
├── SmartShieldService       — UsageStatsManager-based foreground app detection
├── HomeWidgetReceiver        — AppWidget entry point for background mock activation
├── FloatingOverlayService   — System overlay window for out-of-app control
└── MainActivity             — MethodChannel bridge to Flutter
```

---

## Architectural Layers

The application follows a **feature-first Clean Architecture** approach:

```
lib/
├── core/
│   ├── services/          — RemoteConfigService, GeocodingService, ChannelService
│   ├── repositories/      — abstract interfaces (SavedItemsRepository, ScheduleRepository …)
│   ├── models/            — Freezed domain models
│   └── utils/
├── features/
│   ├── map/               — MapView, MapCoordinatorCubit, AppModeCubit
│   ├── basic_mock/        — BasicMockCubit, BasicMockState
│   ├── joystick/          — JoystickCubit, JoystickState
│   ├── route/             — RouteCubit, RouteState
│   ├── pro_route/         — ProRouteCubit, ProRouteState
│   ├── human_simulation/  — HumanSimulationCubit, HumanSimulationState
│   ├── map_tools/         — MapToolsCubit (Ruler, Area, Circle, Compass, Sun)
│   ├── settings/          — ThemeCubit, SchedulerCubit, SmartShieldCubit
│   ├── saved_items/       — SavedItemsCubit (Drift-backed, reactive streams)
│   ├── photo_gps/         — PhotoGpsCubit (EXIF read/write via MethodChannel)
│   ├── splash/            — SplashCubit (RemoteConfig version check)
│   └── welcome/           — WelcomeCubit (onboarding & terms)
└── injection/             — GetIt service locator setup
```

---

## State Management — BLoC Tree

The entire application state is managed through a structured `MultiBlocProvider` hierarchy. A central **`MapCoordinatorCubit`** enforces mutual exclusion across simulation modes — only one mode can be active at a time, preventing state conflicts.

```
EasyLocalization
└── StateInitialize
    └── MultiBlocProvider
        ├── ThemeCubit                   # Material You, custom color, contrast, presets
        ├── SavedItemsCubit              # 5 Drift repositories, watch() stream subscriptions
        ├── MapCoordinatorCubit          # orchestrates all simulation modes
        ├── AppModeCubit                 # top-level mode switch (FakeGPS ⇄ Tools)
        └── Builder
            ├── BasicMockCubit           # fixed location mode
            ├── JoystickCubit            # real-time directional control
            ├── RouteCubit               # A→B route simulation
            ├── ProRouteCubit            # multi-waypoint with speed/altitude/signal control
            ├── HumanSimulationCubit     # natural movement within radius
            ├── MapToolsCubit            # measurement tools state machine
            ├── WelcomeCubit
            └── SplashCubit              # RemoteConfig-driven version gate
```

### Coordinator Pattern

`MapCoordinatorCubit` holds references to all five simulation cubits and exposes a single `activateMode(SimulationMode)` method. When a mode is activated, all others are commanded to stop — ensuring the native `MockLocationService` receives clean, non-conflicting location updates.

---

## How Mock Location Works

The native `MockLocationService` (Android `ForegroundService`) injects fake coordinates into **both** Android location stacks simultaneously:

| Provider | Used by |
|---|---|
| **FusedLocationProviderClient** | Google Maps, modern apps, Play Services |
| **LocationManager** (GPS + Network) | Legacy apps, AOSP location stack |

This dual-injection approach guarantees system-wide compatibility without root access.

---

## Native ↔ Flutter Communication (MethodChannel)

All platform-specific operations are bridged via `MethodChannel`:

| Channel | Direction | Purpose |
|---|---|---|
| `mock_location_channel` | Flutter → Native | Start / stop / update mock GPS |
| `scheduler_channel` | Flutter → Native | Set / cancel `AlarmManager` triggers |
| `smart_shield_channel` | Flutter → Native | Register per-app GPS profiles via `UsageStatsManager` |
| `photo_gps_channel` | Flutter → Native | Read / write EXIF metadata on gallery photos |
| `widget_channel` | Native → Flutter | Home widget & floating overlay activation |

---

## Background & Out-of-App Entry Points

A key architectural challenge was enabling mock location activation **without the app being open**. Four independent entry points handle this:

```
Home Widget (AppWidget)
  └── HomeWidgetReceiver → MockLocationService.start(profile)

Floating Overlay (System Window)
  └── FloatingOverlayService → MockLocationService.start(profile)

Scheduled Alarm
  └── AlarmManager.setExactAndAllowWhileIdle
        └── SchedulerAlarmReceiver → MockLocationService.start(profile)

Smart Shield (App Detection)
  └── SmartShieldService (UsageStatsManager polling)
        └── Foreground app matches profile → MockLocationService.switchProfile(profile)
```

- Android 12+ uses `setExactAndAllowWhileIdle` with FGS start exemption
- `BOOT_COMPLETED` receiver reschedules all persisted alarms after device reboot

---

## Navigation Flow

Declarative routing via **GoRouter** with deep link support (`fakegps://` scheme):

```
Splash  (RemoteConfig version check)
  ├── updateRequired  → UpdateRequiredView   (changelog from Firestore, Play Store CTA)
  ├── !termsAccepted  → LanguageSelectionView (initial setup)
  │       └── WelcomeView → MapView
  └── returning user  → MapView
                            ├── FavoritesDrawer
                            └── SettingsView
                                ├── ThemeView
                                ├── LanguageSelectionView
                                ├── SchedulerView
                                ├── SmartShieldView
                                ├── PhotoGpsView
                                ├── SetupGuideView → MapControlsView
                                └── AboutView  (legal documents)
```

---

## Data Persistence

| Data | Storage | Strategy |
|---|---|---|
| Favorites, routes, schedules | **Drift** (SQLite) | Repository pattern, reactive `watch()` streams |
| Theme, language, map settings | **SharedPreferences** | Accessed via service locator |
| Remote feature flags & version | **Firebase Remote Config** | Fetched on splash, cached locally |

---

## Simulation Modes

| Mode | Cubit | Key Behavior |
|---|---|---|
| Fixed Location | `BasicMockCubit` | Instant pin from map tap, search, coordinates, or favorites |
| Joystick | `JoystickCubit` | Real-time directional offset at configurable speed |
| Route | `RouteCubit` | A→B interpolation at fixed 50 km/h; pause / resume |
| Pro Route | `ProRouteCubit` | Up to 10 waypoints; per-waypoint speed, altitude, wait, signal loss; loop & queue modes |
| Human Simulation | `HumanSimulationCubit` | Random walk within radius; behavior profiles (walk / run / cycle / drive / idle) |

---

## Map Tools Module

`MapToolsCubit` acts as a **state machine** for the Tools mode, managing active tool transitions and accumulated drawing state:

| Tool | Function |
|---|---|
| Ruler | Multi-point cumulative distance |
| Area | Polygon area calculation (m² / km²) |
| Circle | Center + radius; computes circumference & area |
| Cooldown | 2-point distance → safe travel time (Pokémon GO) |
| Compass | Live magnetometer heading; calibration accuracy warnings |
| Sun | Sunrise / sunset / day length for any map point (NOAA algorithm) |
| Sun Path *(Pro)* | Day-scrubber slider; animated solar azimuth ray |

---

## Firebase Infrastructure

The app runs three Firebase services in production, each serving a distinct purpose in the application lifecycle:

### Firebase Crashlytics — Error Monitoring

`firebase_crashlytics` is active in **all production builds**. Uncaught exceptions and Flutter framework errors are automatically captured and reported:

```dart
// main.dart
FlutterError.onError = FirebaseCrashlytics.instance.recordFlutterFatalError;

PlatformDispatcher.instance.onError = (error, stack) {
  FirebaseCrashlytics.instance.recordError(error, stack, fatal: true);
  return true;
};
```

- Fatal and non-fatal errors are tracked separately
- Custom keys are attached to crashes (e.g. active simulation mode, OS version)
- Native Kotlin crashes are captured automatically via the Crashlytics Android SDK
- Enables rapid identification and resolution of production issues

### Firebase Remote Config — Feature Flags & Version Gate

`firebase_remote_config` drives the **Splash screen version check** and controls feature rollout without an app update:

| Key | Purpose |
|---|---|
| `min_version` | Minimum required app version — triggers forced update screen |
| `latest_version` | Current latest version shown in update prompt |
| `changelog_{languageCode}` | Per-language release notes fetched from Remote Config |

`SplashCubit` fetches and caches Remote Config values on every cold start. If `min_version > current_version`, the user is routed to `UpdateRequiredView` and blocked from proceeding until they update via Play Store.

### Cloud Firestore — Dynamic Content

`cloud_firestore` is used to serve **app changelog content** dynamically:

- `FirestoreService.getAppChangelog(languageCode)` fetches release notes in the user's active language
- Displayed on the `UpdateRequiredView` with copy and Google Translate actions
- Content can be updated server-side without a new app release

---

## Flutter Dependencies

> App version: **2.1.0+59** · SDK: `^3.10.7`

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
| **Localization** | easy_localization | ^3.0.7 |
| **Storage** | shared_preferences | ^2.5.4 |
| | drift | ^2.28.0 |
| | sqlite3_flutter_libs | 0.5.32 |
| | path_provider | ^2.1.5 |
| **Theming** | dynamic_color | ^1.7.0 |
| **Connectivity** | connectivity_plus | ^6.1.0 |
| **Photo GPS Editor** | image_picker | ^1.1.2 |
| | native_exif | ^0.6.2 |
| | gal | ^2.3.0 |
| **File Import (GPX/KML/TCX)** | file_picker | ^8.1.2 |
| | xml | ^6.5.0 |
| **Smart Shield** | installed_apps | ^1.6.0 |
| **UI** | lottie | ^3.3.1 |
| | font_awesome_flutter | ^10.12.0 |
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
| **Code Generation** | build_runner | ^2.7.0 |
| | freezed | ^3.0.0 |
| | json_serializable | ^6.9.4 |
| | drift_dev | >=2.28.0 <2.32.0 |
| **Build** | flutter_launcher_icons | ^0.14.4 |
| | flutter_native_splash | ^2.4.7 |
