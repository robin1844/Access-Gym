# Access Gym — Project Briefing for Claude Code

## What this project is

A single-file HTML web app (`index.html`) that connects to a Bluetooth fitness bike and displays real-time stats (speed, cadence, power, distance, calories, time, heart rate). It is designed to be fully accessible with VoiceOver on iPhone.

The app is hosted on GitHub Pages at:
**https://shaunpreece.github.io/access-gym-iphone/**

It must be opened in the **WebBLE browser app** on iPhone (not plain Safari), because Safari does not support the Web Bluetooth API. WebBLE is a companion app that adds Web Bluetooth support to iOS.

## The bike

A **Toputure TEB1** spin bike. It broadcasts over **Bluetooth FTMS** (Fitness Machine Service, UUID `0x1826`), not the CSCS profile. The specific characteristic used for indoor bike data is UUID `0x2ad2` (Indoor Bike Data).

## Architecture

The entire app is a **single self-contained HTML file** with no backend, no build process, no dependencies, and no external scripts. CSS and JavaScript are all inline. It must stay this way — do not introduce npm, frameworks, bundlers, or external libraries.

## Key features

- **FTMS Bluetooth connection** — connects to the bike via Web Bluetooth, reads the Indoor Bike Data characteristic, and parses the binary data per the FTMS spec
- **Live stats display** — speed, cadence, power, distance, calories, elapsed time, heart rate (if broadcast by the bike)
- **Spoken summaries** — uses the browser's ARIA live region (`aria-live="assertive"`) to announce stats on a configurable timer (10s to 10min, or off)
- **Real-time band alerts** — announces cadence/speed/power/heart rate the moment they settle into a new band (cadence/HR to nearest 5, speed to nearest 5, power to nearest 25); a dwell timer of 1500ms prevents constant chatter
- **Per-kilometre/mile milestones** — announces "X kilometres reached" as you ride
- **Personal bests** — tracks session bests, compares against stored all-time bests, announces when beaten; stored in localStorage
- **Units** — km/h or mph, togglable; stored in localStorage
- **Announce preferences** — per-stat toggles for what gets included in spoken summaries; stored in localStorage
- **Dark/light mode** — via CSS `prefers-color-scheme`
- **Accessibility** — full keyboard and VoiceOver navigability; all interactive elements have proper labels; status updates via `role="status"` and `aria-live="polite"`

## Auto-reconnect (work in progress)

The user wants the app to reconnect to the bike automatically on page load without having to go through the Bluetooth picker every time.

**Current state:** A `tryAutoReconnect()` function has been added that:
1. Calls `navigator.bluetooth.getDevices()` on page load to check for a previously permitted device
2. If found, updates the status message to say "Last machine: [name]. Tap Connect to reconnect..."
3. Registers an `advertisementreceived` listener and calls `watchAdvertisements()` to attempt a fully automatic reconnect when the bike starts broadcasting

**Known issue:** `watchAdvertisements()` support in WebBLE on iOS is unconfirmed — it may or may not work. The `tryAutoReconnect()` call is wrapped in a try/catch so it fails silently if unsupported.

**Important:** An earlier attempt to modify `connect()` to skip the picker when a remembered device exists broke the connection flow entirely. The `connect()` function has been **restored to the original** and must not be modified to skip the picker — it always calls `requestDevice()`. Auto-reconnect must only be attempted via `tryAutoReconnect()` running in the background on page load.

## Bluetooth parsing

The `parseBike()` function parses the FTMS Indoor Bike Data characteristic binary format per the Bluetooth spec. The flags word (first 2 bytes, little-endian) indicates which fields are present. Fields parsed:

- Bit 0 clear → speed (u16, /100, km/h)
- Bit 2 → cadence (u16, /2, rpm)
- Bit 4 → distance (u24, metres)
- Bit 6 → power (s16, watts)
- Bit 8 → energy (u16, kcal)
- Bit 9 → heart rate (u8, bpm)

Distance and calories are also estimated by dead reckoning from speed/power if not broadcast directly by the bike.

## Accessibility requirements

- The primary user uses **VoiceOver on iPhone** with the **JAWS screen reader on Windows** (Windows is not the target platform for this app but informs sensitivity to AT issues)
- All spoken output goes through the `#spoken` ARIA live region — never `speechSynthesis` directly
- Status messages go through `#status` (`role="status"`, `aria-live="polite"`)
- No purely visual-only feedback — everything meaningful must be in text
- Buttons must always have visible, descriptive labels
- Focus management matters — don't move focus unexpectedly

## Files in this project

- `index.html` — the entire app (keep single-file)
- `CLAUDE.md` — this briefing document

## Deployment

Push `index.html` to the `main` branch of the GitHub repo `shaunpreece/access-gym-iphone` and GitHub Pages serves it automatically within a minute or two at the URL above.
