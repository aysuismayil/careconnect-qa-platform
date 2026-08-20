# Accessibility Testing

I ran manual accessibility checks against WCAG 2.1 AA as a baseline on all customer-facing flows (search, profile, booking, checkout), since these are the flows every user — including users relying on assistive technology — must be able to complete independently.

## Tools used

| Tool | Purpose |
|---|---|
| axe DevTools (Chrome extension) | Automated scan for common WCAG violations (contrast, missing labels, ARIA misuse) as a first pass |
| Keyboard only (no mouse) | Manual tab-order, focus visibility, and operability check |
| VoiceOver (iOS/macOS) | Screen reader spot checks on native app and Safari |
| TalkBack (Android) | Screen reader spot checks on native app and Chrome |
| Browser zoom to 200% | Reflow/readability check without loss of content or function |
| Chrome DevTools color contrast checker | Manual contrast ratio verification on custom components not caught by axe |

I treat automated tools (axe) as a **first pass, not the full check** — axe catches maybe 30-40% of real accessibility issues; the manual keyboard and screen reader passes are where I find the issues that actually block a real user.

## Test cases

| ID | Title | Steps | Expected Result | Priority | Result |
|---|---|---|---|---|---|
| TC-A11Y-001 | Full booking flow completable using keyboard only | Tab through search → profile → booking → checkout using only Tab/Shift+Tab/Enter/Space, no mouse | Every interactive element is reachable and operable via keyboard; a booking can be completed start to finish | P0 | Pass |
| TC-A11Y-002 | Visible focus indicator on all interactive elements | Tab through the app, observe focus ring on each element | Every focused element has a clearly visible focus outline (not `outline: none` with nothing replacing it) | P0 | Pass — one component (date picker) initially missing visible focus, flagged and fixed |
| TC-A11Y-003 | Logical tab order matches visual layout | Tab through the search results page and checkout form | Tab order follows a sensible reading order (left-to-right, top-to-bottom), not jumping unpredictably | P1 | Pass |
| TC-A11Y-004 | Form inputs have properly associated labels | Inspect checkout form fields (name, card entry iframe, address) with screen reader | Each field announces a meaningful label when focused (e.g., "Cardholder name, edit text"), not just a placeholder that disappears on input | P0 | Fail — two fields relied on placeholder text only; filed as accessibility bug, not detailed as one of this repo's 6 featured bugs but tracked in my working notes |
| TC-A11Y-005 | Images have meaningful alt text | Inspect caregiver profile photos and icons with screen reader | Profile photos announce something meaningful (e.g., "Photo of [caregiver name]"); purely decorative icons are marked `aria-hidden` and skipped, not read as "image" | P1 | Pass, with one icon-only button (favorite/heart icon) missing an accessible name — flagged |
| TC-A11Y-006 | Color contrast meets WCAG AA (4.5:1 for normal text, 3:1 for large text/UI components) | Check text/background contrast on price labels, buttons, and status badges (e.g., "Confirmed" green badge) | All checked combinations meet or exceed the required ratio | P1 | Fail — the light green "Confirmed" badge text on white background measured ~3.2:1, below the 4.5:1 requirement for normal-size text; flagged |
| TC-A11Y-007 | Screen reader announces booking status changes | With VoiceOver/TalkBack running, complete a booking and observe announcements | Status changes (e.g., "Booking confirmed") are announced via an ARIA live region, not silently updated on screen only | P1 | Fail — confirmation message appeared visually but was not announced by the screen reader (missing `aria-live` region); flagged |
| TC-A11Y-008 | Error messages are announced and programmatically associated with their field | Trigger a validation error (e.g., invalid email) with screen reader running | Error is announced immediately and is associated with the field via `aria-describedby` so it's read when the field is revisited | P1 | Pass |
| TC-A11Y-009 | Page content reflows correctly at 200% browser zoom | Zoom browser to 200% on search results and checkout pages | No content is cut off, no horizontal scrolling required to read primary content, all functionality remains usable | P2 | Pass |
| TC-A11Y-010 | Time-based interactions (10-minute slot hold) don't silently disadvantage assistive tech users | Start checkout with a screen reader, take longer than average to complete due to navigating with AT | Countdown/expiry is announced with enough advance warning (not just a silent visual timer) so a slower AT user isn't unfairly cut off mid-checkout | P2 | Fail — hold expiry has no announced warning; flagged as a fairness/accessibility concern, not just a technical one |
| TC-A11Y-011 | Modal dialogs trap focus correctly and are dismissible via keyboard | Open the "Cancel booking" confirmation modal via keyboard | Focus moves into the modal, Tab cycles only within the modal (focus trap), and Escape closes it, returning focus to the triggering element | P1 | Pass |
| TC-A11Y-012 | Native app supports OS-level dynamic text sizing | Increase iOS/Android system font size to largest setting, open the app | Text scales and layout reflows without truncation (same intent as [TC-MOB-009](../test-cases/06-mobile-test-cases.md)) | P1 | Pass |

## Summary of findings from this pass

Several real, user-impacting issues were found and reported to the team (missing form labels relying only on placeholders, insufficient color contrast on the "Confirmed" status badge, and unannounced status/error changes for screen reader users). These were tracked as accessibility bugs in the team's backlog alongside the functional bugs in [`bug-reports/`](../bug-reports/) — I treat accessibility defects with the same rigor and the same bug report structure as functional defects, since they block real users from completing the same flows just as effectively as a functional bug would.
