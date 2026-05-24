# ETA Calculator

A mobile-first web app for calculating your estimated arrival time while driving. Enter your current average speed and remaining distance — the app shows your ETA, a comparison table across nearby speeds, and a ready-to-send message you can share directly from your phone.

## Features

- Live clock based on device time
- ETA calculated from average speed and distance remaining
- Comparison table: ETA at your current speed and ±3 steps of 0.3 km/h, with one tap to adopt any row's speed
- Pre-written summary text, shareable via the native OS share sheet (Messages, WhatsApp, etc.)
- German and English UI, with comma or period as decimal separator
- Dark and light mode, follows OS setting
- Responds to the OS text size setting (Dynamic Type on iOS, browser font size on Android)
- Fully accessible (WCAG 2.1 AA)

## Usage

Open `eta.html` in any mobile browser. No installation, no account, no internet connection required after the file is loaded.

## Privacy

The app sends no data anywhere. Everything runs locally in your browser. Nothing is stored.

## Tech

Single HTML file. No framework, no dependencies, no build step.
