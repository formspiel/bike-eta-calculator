# ETA Calculator

A mobile-first web app for calculating estimated arrival time on a bike ride. Enter your current average speed and remaining distance — the app shows your ETA, a comparison table across nearby speeds, and a ready-to-send message you can share directly from your phone.

## Features

- Live clock based on device time
- ETA calculated from average speed and distance remaining
- Optional ascent (m) with a configurable climbing penalty added to ETA
- Optional planned stop time (min) added to ETA
- Comparison table: ETA at your current speed and ±3 steps of 0.3 km/h, with one tap to adopt any row's speed
- Pre-written summary text with ascent and stop details, shareable via the native OS share sheet (Messages, WhatsApp, etc.) or copy button on desktop
- Settings card to adjust the climbing penalty (default: 9 min per 100 m), saved between sessions
- German and English UI, auto-detected from browser/OS language, switchable manually; comma or period as decimal separator
- Dark and light mode, follows OS setting
- Responds to the OS text size setting (Dynamic Type on iOS, browser font size on Android)
- Fully accessible (WCAG 2.2 AA)
- Works offline after first load, reliably — a service worker caches the app so it loads even after force-quitting in flight mode

## Usage

Open `index.html` in any mobile browser. No installation, no account, no internet connection required after the file is loaded.

## Privacy

The app sends no data anywhere. Everything runs locally in your browser. Only your climbing penalty preference is stored locally if you change it from the default.
