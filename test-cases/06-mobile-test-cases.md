# Test Cases — Mobile (iOS / Android / Responsive Web)

Covers mobile-specific behavior: native app parity, responsive breakpoints, device features, and platform-specific gestures. See [template](../templates/test-case-template.md).

## Device / OS coverage matrix used

| Platform | Devices/OS versions covered |
|---|---|
| iOS (native app) | iPhone SE (small screen), iPhone 14, iPhone 15 Pro Max — iOS current + previous major version |
| Android (native app) | Pixel 6, Samsung Galaxy S22, a budget/low-end device — Android current + previous major version |
| Mobile Web | Safari on iOS, Chrome on Android, responsive breakpoints at 375px, 390px, 428px widths |
| Coverage tool | BrowserStack for device/OS combinations not physically on hand |

## Test cases

| ID | Title | Preconditions | Test Data | Steps | Expected Result | Priority | Type | Platform |
|---|---|---|---|---|---|---|---|---|
| TC-MOB-001 | Booking flow completes end-to-end on native iOS app | iOS app installed, logged in | Test card | Complete full search → book → pay flow | Booking confirmed, matches behavior/data seen on web (parity) | P0 | Positive | iOS |
| TC-MOB-002 | Booking flow completes end-to-end on native Android app | Android app installed, logged in | Test card | Complete full search → book → pay flow | Booking confirmed, matches web behavior/data (parity) | P0 | Positive | Android |
| TC-MOB-003 | Push notification received for booking confirmation | Push permission granted | — | Complete a booking with app backgrounded | Push notification received promptly with correct booking details; tapping it deep-links into the correct booking screen | P1 | Positive | iOS/Android |
| TC-MOB-004 | App requests location permission with clear rationale before using device GPS for search | Fresh install, permission not yet granted | — | Open search screen for the first time | OS permission prompt appears with a clear reason; denying permission does not crash the app and still allows manual location entry | P1 | Positive/Negative | iOS/Android |
| TC-MOB-005 | Denying location permission falls back gracefully to manual search | Location permission denied | — | Deny location permission, attempt search | User can still manually type a location and search normally; no broken state | P1 | Negative | iOS/Android |
| TC-MOB-006 | App handles interrupted network gracefully mid-booking (e.g., subway/elevator dead zone) | Mid-checkout | Simulated network drop | Start checkout, disable network mid-request, re-enable after a delay | No duplicate charge on reconnect (see [TC-PAY-004](05-payments-test-cases.md)); user sees a retry/offline state, not a crash or silent hang | P0 | Negative/Edge | iOS/Android |
| TC-MOB-007 | App state persists correctly when backgrounded mid-flow | Mid-checkout | — | Background the app mid-checkout, return after 2 minutes (within hold window) | User returns to the same checkout step with the slot still held (if within hold window) or a clear "session expired, please restart" message (if hold expired) | P1 | Positive | iOS/Android |
| TC-MOB-008 | Responsive layout does not break at small screen widths | Mobile web, 375px viewport | — | Load search results, profile, and checkout pages at 375px width | No horizontal scroll, no overlapping/cut-off text or buttons, tap targets remain usable | P1 | Positive | Mobile Web |
| TC-MOB-009 | Text scales correctly with OS-level accessibility font size settings | Device font size set to largest supported | — | Increase system font size to max, open the app | Text scales and layout reflows without truncation or overlapping elements | P1 | Positive/Accessibility | iOS/Android |
| TC-MOB-010 | Camera/photo picker works for profile photo upload on native app | Caregiver logged in on native app | — | Tap "Upload photo," choose "Take Photo" and "Choose from Library" | Both paths work; permission prompts appear appropriately if not yet granted; photo uploads successfully | P2 | Positive | iOS/Android |
| TC-MOB-011 | Deep link opens the correct in-app screen | App installed | Deep link to a specific caregiver profile | Tap a shared deep link (e.g., from SMS/email) | App opens directly to the correct caregiver profile screen, not just the home screen | P2 | Positive | iOS/Android |
| TC-MOB-012 | App behaves correctly on orientation change (portrait/landscape) | Mid-flow on any screen | — | Rotate device during search results and during checkout | Layout adapts without losing form input or crashing | P2 | Positive | iOS/Android |
| TC-MOB-013 | Biometric login (Face ID/Touch ID/fingerprint) works and falls back to password | Biometric enabled on device, account linked | — | Attempt login via biometric; then test fallback | Biometric login succeeds; if biometric fails/unavailable, password fallback is offered, not a dead end | P2 | Positive | iOS/Android |
| TC-MOB-014 | App update banner/force-update screen appears for deprecated app versions | Simulated old app version against current backend | Old build number | Launch an outdated app build against staging | If backend enforces a minimum version, user sees a clear "please update" screen rather than broken/undefined behavior | P2 | Negative | iOS/Android |
| TC-MOB-015 | Native app and mobile web show identical pricing/booking data for the same account | Same account logged in on native app and mobile Safari/Chrome | Same booking | View the same booking's details on both native app and mobile web | Price, time, and status are identical on both — no platform-specific data drift | P1 | Positive | iOS/Android/Mobile Web |
