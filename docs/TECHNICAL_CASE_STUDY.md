# Technical Case Study: LedgerWise

An engineering write-up of a Flutter expense tracker for iOS and Android. It is based on read-only inspection of the production codebase. No production source is reproduced; descriptions are deliberately high level.

## 1. Product context

LedgerWise is a small, fast expense tracker: record a purchase, watch a monthly goal, look at spending by period and category, export a report. It is free with ads and has a Premium subscription that removes ads and unlocks exports. Data lives on the device first; sign-in is optional and adds cross-device sync.

The engineering interest is in the seams around a simple core: syncing a local store with a cloud source of truth, two different billing stacks, native reminders, ad policy, and analytics hygiene.

## 2. Requirements that shaped the design

| Area | Requirement |
|---|---|
| Speed | Add an expense in a few taps; the app must open to something useful immediately |
| Offline | Everything works without a network, for guests too |
| Sync | Optional sign-in; changes follow the user across devices |
| Insights | Period summaries and charts without a charting dependency |
| Export | CSV and PDF reports for an arbitrary date range |
| Reach | 15 languages, any currency |
| Monetisation | Subscription with trial, plus ads that can be turned off remotely and never block app access |
| Platform | iOS and Android with different store, notification and tracking rules |

## 3. Architecture

Screens → storage service (Hive) and services (sync, billing, ads, reminders, analytics, auth) → native bridges and Firebase. State is `provider` plus a small app-wide refresh signal that screens listen to after sync and auth changes. See [ARCHITECTURE.md](ARCHITECTURE.md).

- **Startup.** Native splash, then a Flutter splash that plays a short intro clip, overlapping premium and billing restore with the minimum splash time, then a bounded app-open ad, then onboarding or the dashboard. Every failure path falls through to the reveal.
- **Navigation.** Drawer-based navigation between Dashboard, Analytics, Categories, Export, Notifications, Settings and More Apps.
- **Guided tour.** A coach-mark controller with a mixin that, if the user taps something before the tour starts (opens the drawer, pushes a screen), restores the expected screen first. Starting the tour hides inline ads.

## 4. Data and storage

- **Models.** Expense (id, item name, category, price, quantity, date, note, created time), Category (id, name, icon or emoji, custom flag, optional image, sort order) and Settings (currency, monthly goal, onboarding flag, premium flag, user id), each a Hive type with a generated adapter.
- **Totals.** An expense's total is price times quantity. Money is stored as double values.
- **Defaults.** Default categories are localised and re-asserted after sync.
- **Preferences.** Reminder settings and similar small values live in shared preferences.
- **Reset and delete.** Sign-out wipes the local store; reset and account deletion are explicit flows.

## 5. Sync

Firestore is the source of truth while signed in, Hive is the working cache. Initial sync downloads (or uploads when the cloud is empty), then live listeners merge changes. Security rules restrict every read and write to the signed-in user's own document tree. See the sequence diagram in the architecture doc.

## 6. Analytics screens and export

- **Dashboard.** Day, week and month periods; entries grouped by day; monthly goal progress.
- **Analytics.** Daily, weekly, monthly and yearly periods with summary cards, top categories and a chart that toggles between by-date and by-category views. Charts are drawn with custom painters rather than a charting package.
- **CSV export.** A UTF-8 file with a byte-order mark so spreadsheet apps detect the encoding, named from the selected date range, shared through the OS share sheet.
- **PDF export.** An A4 document built with the `pdf` package, with a branded header, summary block and a bordered transaction table.
- **Gating and telemetry.** Both exports are Premium; started, completed and failed events are logged and failures are recorded as non-fatal errors.

## 7. Platform integration

| Platform | Capability |
|---|---|
| iOS (Swift) | StoreKit billing over method and event channels; App Tracking Transparency with a native cold-launch trigger; push registration |
| Android (Kotlin) | WorkManager workers for daily, weekly and monthly reminders; notification channels; native notification permission request; splash activity |
| Both | Local notifications, in-app update prompts, share sheet, image picker |

A vendored fork of the Apple sign-in package carries an iPad presentation-anchor fix, with a documented plan to return to upstream.

## 8. Monetisation and policy

- **Premium.** Local flag mirrored to the cloud profile, with a change notifier so ads and prompts react immediately to purchase, restore and sign-out.
- **Ads.** A facade over a mediation SDK applies two gates: a remote kill switch and premium entitlement. Per-placement toggles live in a remote document, and ads load on demand.
- **Consent.** App Tracking Transparency on iOS; ad consent is collected by the mediation SDK.
- **Policy guard rails.** The startup ad is bounded by a timeout and never blocks access to the app.

## 9. Analytics and diagnostics

A sanitiser sits in front of every analytics event and user property to enforce GA4's name and parameter rules; a crash-context helper attaches non-fatal errors with context; Firebase Performance is wired in. A connectivity service combines interface state with a DNS lookup to detect real internet reachability.

## 10. Performance

- Hive gives synchronous-feeling local reads for the lists the dashboard needs.
- Entries are grouped by day once per build for the selected period.
- Charts are custom painters.
- Native workers handle reminders without the app running.
- Shimmer placeholders cover ad loading.
- Release builds use R8 minification and resource shrinking.

## 11. Testing and release

Unit tests cover the ad placement catalogue and config service, the analytics sanitiser and context, and the tour controller, plus a default widget test. Sync, billing, export and reminders rely on manual testing; a release checklist and a verification script exist for physical-device smoke tests.

## 12. Trade-offs and maintainability

- **Strengths:** a clear storage/sync/service split, platform differences handled in dedicated bridges, defensive handling of non-core failures, and documented reasons in comments.
- **Weaknesses:** a few screens are very large (analytics is around two thousand lines, the dashboard over a thousand); sync is cloud-wins rather than field-level merge; money is stored as floating-point values.

## 13. Possible next steps

- Store money as integer minor units and format at the edge.
- Split the largest screens and extract chart painters into their own files.
- Field-level or timestamp-based conflict resolution for offline edits made on two devices.
- Automated tests around sync merging and export content.
- Offer the yearly plan on iOS as on Android.
