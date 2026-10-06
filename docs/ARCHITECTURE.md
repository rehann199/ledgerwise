# Architecture

All diagrams are original and conceptual. They describe responsibilities, not production class or file names.

## 1. Layers

```mermaid
flowchart TD
    UI[Screens and widgets]
    AS[App state<br/>refresh signal on sync / auth change]
    ST[Storage service<br/>typed Hive boxes]
    SY[Sync service<br/>Firestore listeners]
    SV[Services<br/>billing, premium, ads, reminders,<br/>analytics, auth, ATT, updates]
    NT[Native bridges<br/>Swift + Kotlin]
    FB[Firebase]
    UI --> AS
    UI --> ST
    ST <--> SY
    SY <--> FB
    UI --> SV
    SV --> NT
    SV --> FB
```

The UI reads and writes through the storage service. Everything else (sync, billing, ads, reminders) sits beside it as a service so a screen never talks to an SDK directly.

## 2. Data model

```mermaid
erDiagram
    EXPENSE {
        string id
        string itemName
        string category
        double price
        int quantity
        datetime dateTime
        string note
        datetime createdAt
    }
    CATEGORY {
        string id
        string name
        string icon
        bool isCustom
        string imagePath
        int sortOrder
    }
    SETTINGS {
        string currency
        double monthlyGoal
        bool hasCompletedOnboarding
        bool isPremium
        string userId
    }
    CATEGORY ||--o{ EXPENSE : "referenced by name"
```

An expense's total is price multiplied by quantity. Categories are referenced from expenses by category value, and defaults are guaranteed to exist after any sync.

## 3. Sync

```mermaid
sequenceDiagram
    participant U as User
    participant A as Auth
    participant S as Sync service
    participant F as Firestore
    participant H as Hive
    U->>A: Sign in (Google / Apple)
    A->>S: Signed-in user
    S->>F: Fetch expenses, categories, settings
    alt cloud has data
        F-->>S: Cloud data
        S->>H: Overwrite local with cloud (initial)
    else cloud empty
        S->>F: Push local data
    end
    S->>F: Start live listeners
    F-->>S: Remote change
    S->>H: Merge change (no full overwrite)
    S-->>U: Refresh UI
    U->>A: Sign out
    S->>H: Wipe local data
```

Firestore is the source of truth while signed in; Hive is a local cache that also serves guests. Security rules scope all reads and writes to documents under the signed-in user's own id.

## 4. Startup

```mermaid
flowchart LR
    N[Native splash] --> FS[Flutter splash<br/>intro clip]
    FS --> PR[Restore premium + billing]
    PR --> AO[App-open ad<br/>bounded by timeout]
    AO --> H[Onboarding or Dashboard]
    PR -. overlaps minimum splash time .-> FS
```

The ad is bounded by a timeout and every failure path falls through to the reveal, so access to the app is never blocked behind an ad.

## 5. Billing

```mermaid
flowchart TD
    PAY[Paywall] --> BS[Billing service]
    BS -->|iOS| SK[Swift StoreKit bridge<br/>method + event channels]
    BS -->|Android| PP[Play billing plugin]
    PP --> PLAN[Plan model:<br/>base plan + trial offer collapsed]
    SK --> PS[Premium service]
    PLAN --> PS
    PS --> ST[(Local premium flag)]
    PS --> FS[(Cloud profile)]
    ST --> ADS[Ads and export gates]
```

Premium state is stored locally and mirrored to the cloud profile. A change notifies listeners, so ads and upgrade prompts disappear immediately.

## 6. Reminders

On Android, daily, weekly and monthly reminders run as native WorkManager workers with their own notification channels and a native permission request. On iOS they are scheduled local notifications. One wrapper is the single call site for scheduling and cancelling, so a platform failure is caught and reported there.
