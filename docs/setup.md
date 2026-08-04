# Setup Guide

Map Tools: Fake GPS & Tracker uses Android's mock location API. A one-time setup
is required.

---

## Steps

### 1. Enable Developer Options

1. Open **Settings** on your Android device
2. Go to **About phone**
3. Tap **Build number** 7 times
4. Developer Options is now unlocked

### 2. Enable Mock Location

1. Open **Settings → Developer Options**
2. Find **Select mock location app**
3. Choose **Map Tools: Fake GPS & Tracker**

### 3. Grant the location permission

Open the app and allow location access when asked. It is used to centre the map,
verify your setup, and — only while you are recording a walk yourself — sample
your real GPS track.

### 4. You're ready

Pick a simulation mode. The in-app setup guide walks through each step with
real-time status checks and can open the right Android settings screen for you.

---

## Notes

- No root required
- Works system-wide — all apps will see the simulated location
- To stop mocking, press the stop button inside the app, or use the home screen
  widget / floating widget if the app is closed
- Recording a walk is blocked while a mock location is active, so a simulated
  track can never be saved as a real one
