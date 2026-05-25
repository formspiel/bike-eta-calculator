# ETA Calculator

A mobile-first web app for calculating estimated arrival time on a bike ride. Enter the average speed from your bike computer and how far you still have to go — the app shows your ETA, a comparison table across nearby speeds, and a shareable message you can send to let people know when you'll arrive. Works reliably even with a poor signal, and fully offline once loaded.

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
- Text size follows your device's text size preference — increase it in iOS Settings → Accessibility → Larger Text, or Android Settings → Display → Font size. Note: the iOS Control Centre text size shortcut does not apply to web apps.
- Designed for accessibility, targeting WCAG 2.2 AA
- Works reliably even offline — a service worker caches the app so it loads even after force-quitting in flight mode

## Usage

Open `index.html` in any mobile browser. No installation, no account, no internet connection required after the file is loaded.

## Privacy

The app sends no data anywhere. Everything runs locally in your browser. Only your climbing penalty preference is stored locally if you change it from the default.
