# Engineering Challenges

Seven problems that stand out in the production code. Outcomes describe observable behaviour in the code; no metrics are claimed.

---

## 1. Local-first data that also follows the user

**Problem**
The app has to work instantly and offline for guests, yet keep a signed-in user's data consistent across devices, including a second device that signs in later.

**Engineering approach**
- Typed Hive boxes are the working store for everyone.
- On sign-in, a sync service downloads the cloud copy and overwrites the local one for the initial sync. If the cloud is empty it uploads the local data instead, so a guest's entries are not lost when they sign in.
- Live listeners then merge remote changes into Hive without rewriting everything, and the UI refreshes through an app-wide refresh signal.
- Default categories are re-asserted after every sync so a sync can never leave the app without them.
- Sign-out wipes local data so a shared device starts clean.

**Trade-offs**
The initial sync favours the cloud when both sides have data. That is simple and predictable, but it is not a field-level merge.

**Outcome**
Guests and signed-in users run the same code path, and changes made elsewhere appear without a restart.

---

## 2. Two stores, two billing models

**Problem**
Google Play returns one product entry per offer, and each entry's price is only its first pricing phase, which is "Free" for a trial offer. Naively displayed, the paywall shows "Free" instead of the real price, or hides the trial. iOS is a different API entirely.

**Engineering approach**
- A platform-agnostic billing service routes to a native Swift StoreKit bridge on iOS (method and event channels) and the Play billing plugin on Android. The Android plugin is only touched on Android so it never registers on iOS.
- A plan model groups the base-plan entry and the free-trial entry, extracts the true renewal price and keeps the offer tokens needed to buy with or without the trial.
- Product details are refreshed before the paywall opens, and a diagnostic dump lists every offer and pricing phase Play returned.
- A purchase result type carries restore and error outcomes to the UI.

**Trade-offs**
Maintaining a native iOS billing bridge costs more than using the plugin everywhere, in exchange for direct control of the purchase lifecycle. iOS currently sells one plan while Android sells two.

**Outcome**
The paywall shows the renewal price and trial status correctly, from one normalised model.

---

## 3. A tracking prompt that never appears

**Problem**
On a real cold launch, requesting App Tracking Transparency from Dart can arrive before iOS has an established key window. iOS then resolves the request as "not determined" without showing the dialog, which is what App Store review sees even when local testing looks fine.

**Engineering approach**
A native trigger requests authorisation when the app becomes active, independent of the Flutter engine or method channel being ready. Requests are de-duplicated and callbacks queued, so the trigger is safe on every foreground and returns the cached status afterwards.

**Trade-offs**
Two entry points (native and Dart) must share one source of truth, which is why the request is de-duplicated natively.

**Outcome**
The system prompt appears reliably on first launch, and later calls are no-ops.

---

## 4. Remote ad toggles that did nothing until restart

**Problem**
Ad placements are switched from a remote document, but flipping a toggle had no effect until the app restarted.

**Engineering approach**
Banner slots were placed as `const` widgets, which Flutter canonicalises and therefore never rebuilds when an ancestor does. Reading the config inside `build` was not enough. Every inline ad widget now listens to a merged notifier that fires when the kill switch changes, a placement flag changes, the tutorial starts or ends, or premium status flips.

**Trade-offs**
Each ad widget must remember to subscribe; the rule is documented at the notifier because it is easy to forget.

**Outcome**
Remote toggles, entering premium and starting the tour hide or restore ads immediately.

---

## 5. Analytics events that are silently dropped

**Problem**
GA4 truncates or drops events and parameters that break its limits, and silently accepts reserved names that then do not behave as expected. None of this is visible where the event is logged.

**Engineering approach**
All events and user properties pass through one sanitiser, wired into the two funnels every event uses. It renames or fixes violating names, cleans parameters and returns the violations so a breadcrumb can be logged. Unit tests cover the sanitiser and the shared analytics context.

**Trade-offs**
A central funnel is a single point of failure for analytics, so it is kept pure and tested.

**Outcome**
Bad event names and parameters are caught in one place instead of corrupting reports.

---

## 6. Reminders that survive restarts and failures

**Problem**
A logging reminder is only useful if it fires on schedule across restarts, and a platform-channel failure while scheduling should not take down unrelated screens.

**Engineering approach**
- Android uses native WorkManager workers for daily, weekly and monthly reminders, with notification channels created at app start and a native request for the notification permission.
- iOS uses scheduled local notifications, with timezone-aware times.
- One wrapper is the only call site for scheduling and cancelling, used by settings, onboarding and language changes; failures are caught and reported to crash reporting.
- Permission state can be read without prompting, and a pre-prompt sheet explains the reason before the system dialog.

**Trade-offs**
Two scheduling implementations must stay behaviourally aligned.

**Outcome**
Reminders are scheduled natively, and scheduling failures are visible in crash reports instead of invisible.

---

## 7. Online is not the same as connected

**Problem**
A network interface being active does not mean the internet is reachable (Wi-Fi without internet, captive portals).

**Engineering approach**
A connectivity service combines interface state with a lightweight DNS lookup, exposing a notifier and a stream so widgets and sync can distinguish "connected to Wi-Fi with no internet" from "actually online".

**Trade-offs**
An active probe costs a little extra work compared with trusting interface state.

**Outcome**
Offline banners and ad logic react to real reachability.
