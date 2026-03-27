# Architecture

## Stack

```
Flutter (Dart)
├── BLoC / Cubit     — state management
│   └── HydratedBloc — automatic persistence (favorites, routes, schedules)
├── GoRouter          — navigation (8 routes)
├── Freezed           — immutable state models
├── EasyLocalization  — 14-language support
└── Google Maps       — map rendering, markers, polylines

Native (Kotlin)
├── MockLocationService      — ForegroundService, dual-provider GPS injection
├── SchedulerAlarmReceiver   — BroadcastReceiver + BOOT_COMPLETED handling
└── MainActivity             — MethodChannel bridge to Flutter
```

---

## Simulation Mode Coordination

A central `MapCoordinatorCubit` orchestrates all four modes with mutual exclusion — only one mode can be active at a time.

```
MapCoordinatorCubit
├── BasicMockCubit        (tap-to-mock)
├── JoystickCubit         (real-time joystick)
├── RouteCubit            (multi-waypoint route)
└── HumanSimulationCubit  (natural movement)
```

---

## How Mock Location Works

The native service injects the fake location into both Android location stacks simultaneously:

- **FusedLocationProviderClient** — used by Google Maps and modern apps
- **LocationManager** (GPS + Network providers) — used by legacy apps

This ensures compatibility across virtually all apps.

---

## Scheduler Architecture

```
Flutter (Dart)                    Native (Kotlin)
SchedulerChannel  ─────────────→  AlarmManager.setExactAndAllowWhileIdle
                                       │
                                       ▼ (alarm fires)
                                  SchedulerAlarmReceiver (BroadcastReceiver)
                                       │
                                       ▼
                                  MockLocationService (ForegroundService)
```

- Android 12+: uses `setExactAndAllowWhileIdle` (FGS start exemption)
- Falls back to `setAndAllowWhileIdle` if exact alarm permission is denied
- `BOOT_COMPLETED` reschedules all persisted alarms after reboot

---

## Navigation Flow

```
Splash (version check)
  ├── Update required → Play Store
  ├── First launch → Language Selection → Welcome → Setup Guide → Map
  └── Returning user → Map
                          └── Settings
                              ├── Language Selection
                              ├── Setup Guide → Map Controls
                              └── About
```

---

## Flutter Dependencies

| Category | Package | Version |
|----------|---------|---------|
| Maps | google_maps_flutter | ^2.14.0 |
| Location | geolocator | ^14.0.2 |
| Location | permission_handler | ^12.0.1 |
| State | flutter_bloc | ^9.1.0 |
| State | hydrated_bloc | ^10.1.1 |
| Navigation | go_router | ^17.1.0 |
| Models | freezed_annotation | ^3.0.0 |
| Firebase | firebase_core | ^4.4.0 |
| Firebase | firebase_remote_config | ^6.1.4 |
| Localization | easy_localization | ^3.0.7 |
| Storage | shared_preferences | ^2.5.4 |
| UI | lottie | ^3.3.1 |
| UI | font_awesome_flutter | ^10.12.0 |
| Utils | url_launcher | ^6.3.1 |
| Utils | share_plus | ^12.0.1 |
| Utils | uuid | ^4.5.3 |
