# Flashcard App — Technical Architecture Specification

**Version:** 1.0  
**Date:** March 2026  
**Status:** Draft for implementation  
**Scope:** Desktop-first web application architecture for v1, followed by PWA/mobile additions

---

## 1. Purpose

This document defines the technical architecture for the Flashcard App, with a specific focus on:

1. the **initial pre-mobile implementation** — a desktop-first, browser-based application with a stable app shell, routing, local persistence, CRUD flows, and review engine
2. the **second-phase PWA/mobile additions** — installability, offline app-shell caching, service worker behaviour, touch-first adaptations, and notification support

This is intended to act as a build-ready engineering handoff for the first implementation phase.

---

## 2. Product Summary

The Flashcard App is a local-first study application for creating and reviewing flashcards. Content is organised in a three-level hierarchy:

- **Packs** — top-level grouping
- **Sets** — topics/chapters within a pack
- **Cards** — question/answer pairs within a set

Version 1 is intentionally simple:

- no user accounts
- no backend
- no cloud sync
- all primary data stored on-device
- fully usable for core flows without a network connection once loaded

The app should feel fast, quiet, focused, and reliable.

---

## 3. Delivery Phases

## Phase A — Pre-mobile foundation

This phase delivers the complete functional web app without PWA/mobile-specific behaviours. It includes:

- desktop-first app shell
- routing
- IndexedDB persistence via Dexie
- CRUD for packs, sets, and cards
- CSV import/export
- review session engine
- session results and card stats
- settings
- keyboard accessibility
- error handling and empty states

This phase establishes the codebase, domain model, data contracts, and UX patterns.

## Phase B — PWA/mobile layer

This phase adds:

- installable PWA packaging
- web manifest
- service worker and caching strategy
- offline shell loading after first visit
- mobile layout refinements
- swipe gestures
- study reminder notifications
- standalone app experience

The PWA layer must be additive. It should not require rewriting the Phase A domain or UI architecture.

---

## 4. Goals and Non-Goals

## Goals

- Deliver a stable app shell and maintainable frontend architecture
- Keep all core data local-first and durable
- Make the review flow reliable and deterministic
- Make the app straightforward to extend later with spaced repetition and sync
- Separate transient UI state from persisted domain state
- Ensure the PWA layer can be introduced cleanly after the desktop-first build is stable

## Non-goals for v1

- authentication
- cloud sync
- collaborative or shared decks
- image cards
- spaced repetition algorithms beyond the lightweight re-queue loop
- server APIs
- multi-device merge logic

---

## 5. Architecture Principles

The implementation should follow these principles:

### 5.1 Local-first
Core data is stored on the client in IndexedDB. The app should not depend on network calls for normal v1 usage.

### 5.2 Feature-based frontend structure
The codebase should be organised around features and domain boundaries, not only page files.

### 5.3 Clear separation of responsibilities
The application should separate:

- **routing and app shell**
- **feature UI**
- **domain logic**
- **persistence**
- **shared UI primitives**

### 5.4 Deterministic review engine
Review behaviour must be implemented as explicit state transitions and not spread across ad hoc UI handlers.

### 5.5 PWA as an enhancement layer
PWA/mobile behaviour should plug into the existing architecture rather than forcing redesign of data or navigation.

---

## 6. Recommended Stack

## Core

- **Framework:** React
- **Build tool:** Vite
- **Language:** TypeScript
- **Routing:** React Router
- **Database:** Dexie.js over IndexedDB
- **Forms:** React Hook Form or lightweight native controlled forms
- **CSV parsing/export:** Papa Parse for import; native Blob download for export
- **Charts:** Recharts for results donut chart
- **Styling:** Tailwind CSS or CSS Modules
- **Utility class merging:** clsx / class-variance-authority if using Tailwind

## PWA layer

- **PWA tooling:** vite-plugin-pwa
- **Service worker mode:** generated via Workbox through vite-plugin-pwa
- **Notifications:** Web Notifications API only where supported

## Why this stack

This combination keeps the v1 build simple, client-only, and deployable on Vercel without backend infrastructure. It also keeps the later move toward sync or Supabase viable because the domain model remains independent from transport.

---

## 7. High-Level System Overview

The app consists of five main layers:

### 7.1 App shell layer
Provides routing, top-level layout, global providers, and route error handling.

### 7.2 Feature layer
Implements user-facing areas:

- packs
- sets
- cards
- review
- results
- settings
- import/export

### 7.3 Domain layer
Contains pure types, transformation logic, review-session logic, validation rules, and derived calculations.

### 7.4 Persistence layer
Wraps Dexie access, schema definitions, repository functions, and transaction logic.

### 7.5 PWA/runtime layer
Adds manifest, service worker, install prompts, notification support, and connection/offline events.

---

## 8. Application Layout and App Shell

## 8.1 Shell objectives
The shell must provide a consistent frame for all non-review screens and a distraction-free layout for review.

## 8.2 Layout modes

The app uses two top-level layout modes:

### StandardShell
Used for:

- Home
- Pack Detail
- Set Detail
- Create Pack
- Create Set
- Create Card
- Settings
- Results

Contains:

- left navigation rail or compact top nav depending on width
- top app header
- page container with max width
- global toast area
- modal host

### ReviewShell
Used for:

- Review Session

Contains:

- minimal chrome
- top progress bar / progress summary
- central card stage
- review action controls
- optional exit button

Review should intentionally bypass the denser app shell so it feels focused and full-screen.

## 8.3 Global shell responsibilities

The shell owns:

- theme tokens and typography bootstrapping
- provider composition
- route rendering
- layout switching by route
- global toasts
- confirmation dialogs
- not-found screen
- database ready/loading state

---

## 9. Route Architecture

## 9.1 Route table

| Route | Screen | Layout | Purpose |
|---|---|---|---|
| `/` | Home | StandardShell | Browse packs |
| `/pack/:packId` | Pack Detail | StandardShell | View sets in a pack |
| `/set/:setId` | Set Detail | StandardShell | View cards, import/export, start review |
| `/review/:setId` | Review Session | ReviewShell | Run a session |
| `/results/:sessionId` | Results | StandardShell | Review session outcomes |
| `/create/pack` | Create Pack | StandardShell | Create a pack |
| `/create/set/:packId` | Create Set | StandardShell | Create a set |
| `/create/card/:setId` | Create Card | StandardShell | Create a card |
| `/edit/card/:cardId` | Edit Card | StandardShell | Edit an existing card |
| `/settings` | Settings | StandardShell | Manage preferences |
| `*` | Not Found | StandardShell | Unknown route |

## 9.2 Route behaviour requirements

Each route must handle four states where relevant:

- loading
- success
- empty
- error/not found

Examples:

- invalid `packId` -> not-found state with return action
- valid pack with no sets -> empty state with “Create Set” CTA
- valid set with no cards -> empty state with disabled review button
- invalid `sessionId` -> results not found state

## 9.3 Route data loading

Initial implementation can load route data inside route components using feature hooks. If React Router data APIs are introduced, they should still delegate to the same feature services.

---

## 10. Recommended Frontend Project Structure

```text
src/
  app/
    providers/
    router/
    layout/
    styles/
  components/
    ui/
    feedback/
    charts/
  features/
    packs/
      components/
      hooks/
      pages/
      services/
      types.ts
    sets/
      components/
      hooks/
      pages/
      services/
      types.ts
    cards/
      components/
      hooks/
      pages/
      services/
      types.ts
    review/
      components/
      hooks/
      engine/
      pages/
      services/
      types.ts
    results/
      components/
      pages/
      services/
    settings/
      components/
      pages/
      services/
    import-export/
      components/
      services/
      validators/
  db/
    dexie.ts
    schema.ts
    migrations.ts
    repositories/
  domain/
    review/
    stats/
    validation/
    ids/
    dates/
  hooks/
  lib/
  types/
  pwa/
    install/
    notifications/
    offline/
```

## 10.1 Structure rules

- **pages** contain route-level composition only
- **components** contain reusable feature UI
- **services** coordinate persistence and domain logic
- **engine/domain** contains pure logic independent from React
- **db/repositories** handle direct Dexie reads/writes
- **ui** contains generic primitives such as Button, Modal, Card, Input, EmptyState, ConfirmDialog

---

## 11. Global Providers

The root app should compose providers in roughly this order:

1. Router provider
2. Theme/token provider if needed
3. Query/data provider only if introduced later
4. Toast provider
5. Dialog/modal host
6. Settings provider
7. PWA/install/offline provider in Phase B

For Phase A, keep global providers minimal. Avoid introducing heavy global state libraries unless needed.

---

## 12. State Management Strategy

## 12.1 Persisted state
Persisted state lives in IndexedDB or localStorage.

### IndexedDB via Dexie

- packs
- sets
- cards
- sessions
- results
- stats
- resumable review session snapshot

### localStorage

- settings
- lightweight UI preferences not needing relational queries

## 12.2 Transient in-memory state

Use component state or feature hooks for:

- current modal open/closed status
- temporary form input state
- unsaved validation errors
- hover/focus UI state
- currently flipped review card face
- temporary import preview info

## 12.3 Review session runtime state

The active review loop should be managed by a dedicated review engine hook backed by a reducer. This state should include:

- current queue
- current index or current card pointer
- per-card outcomes for current session
- flagged cards list
- whether card face is flipped
- whether session was resumed
- whether session is complete

The reducer should be pure and testable.

---

## 13. Domain Model

## 13.1 Core entities

```ts
interface Pack {
  id: number
  name: string
  color: string
  createdAt: string
  updatedAt?: string
}

interface SetEntity {
  id: number
  packId: number
  title: string
  description?: string
  createdAt: string
  updatedAt?: string
}

interface CardEntity {
  id: number
  setId: number
  question: string
  answer: string
  createdAt: string
  updatedAt?: string
}

interface Session {
  id: number
  setId: number
  startedAt: string
  completedAt?: string
  mode: 'full' | 'flagged' | 'incorrect-only'
  score?: number
}

interface Result {
  id: number
  sessionId: number
  cardId: number
  outcome: 'correct' | 'incorrect' | 'flagged'
  timestamp: string
}

interface CardStats {
  id: number
  cardId: number
  correctCount: number
  incorrectCount: number
  flaggedCount: number
  lastResult?: 'correct' | 'incorrect' | 'flagged'
  lastReviewedAt?: string
}

interface Settings {
  shuffleCards: boolean
  flipAnimation: boolean
  autoShowAnswer: 'off' | '3s' | '5s' | '10s'
  swipeGestures: boolean
  studyReminderNotifications: 'off' | 'daily' | 'custom'
}
```

## 13.2 Session snapshot entity

To support resume, store a dedicated in-progress snapshot:

```ts
interface ActiveSessionSnapshot {
  sessionId: number
  setId: number
  createdAt: string
  updatedAt: string
  queueCardIds: number[]
  currentCardId: number | null
  outcomes: Array<{ cardId: number; outcome: 'correct' | 'incorrect' | 'flagged' }>
  flaggedCardIds: number[]
  incorrectCardIds: number[]
  isFlipped: boolean
  mode: 'full' | 'flagged' | 'incorrect-only'
}
```

This should be stored in Dexie so resume works after browser close.

---

## 14. Database Architecture (Dexie)

## 14.1 Store design

Recommended stores:

- `packs`
- `sets`
- `cards`
- `sessions`
- `results`
- `stats`
- `activeSessions`

## 14.2 Schema definition example

```ts
packs: '++id, name, createdAt'
sets: '++id, packId, title, createdAt'
cards: '++id, setId, createdAt'
sessions: '++id, setId, startedAt, completedAt, mode'
results: '++id, sessionId, cardId, outcome, timestamp, [sessionId+cardId]'
stats: '++id, cardId, lastReviewedAt'
activeSessions: 'sessionId, setId, updatedAt'
```

## 14.3 Persistence rules

- `results` is the source of truth for session outcomes
- `stats` is a derived performance table updated at session completion
- `activeSessions` only stores incomplete sessions
- deleting a pack cascades to sets, cards, sessions, results, stats references, and active session snapshots tied to descendant sets
- deleting a set cascades to cards, sessions, results, stats references, and active session snapshots

## 14.4 Transaction boundaries

Transactions should be used for:

- pack delete cascade
- set delete cascade
- session completion write
- bulk CSV import
- active session create/update/clear

## 14.5 Migration strategy

Schema versions must be explicitly managed through Dexie versioning. All schema changes should:

- add a new version block
- preserve existing user data where possible
- include upgrade logic for renamed fields or new derived fields

---

## 15. Repository and Service Layer

Repositories should be thin wrappers around Dexie queries. Services should combine repository and domain logic.

### Example responsibility split

#### Repositories
- `packRepository.getAll()`
- `setRepository.getByPackId(packId)`
- `cardRepository.getBySetId(setId)`
- `sessionRepository.create()`
- `resultRepository.bulkInsert()`

#### Services
- `createPackService`
- `deletePackCascadeService`
- `importCardsFromCsvService`
- `startReviewSessionService`
- `completeReviewSessionService`
- `resumeReviewSessionService`
- `exportSetToCsvService`

This keeps database concerns separate from workflow concerns.

---

## 16. Feature Architecture

## 16.1 Packs feature

Responsibilities:

- list packs on home
- create pack
- delete pack
- navigate to pack detail
- show pack color indicator and set count

Key components:

- `PackList`
- `PackCard`
- `CreatePackForm`
- `DeletePackDialog`

## 16.2 Sets feature

Responsibilities:

- list sets within a pack
- create set
- delete set
- show set metadata and card count
- enter review

Key components:

- `SetList`
- `SetRow`
- `CreateSetForm`
- `SetHeader`
- `StartReviewButton`

## 16.3 Cards feature

Responsibilities:

- list cards in a set
- create/edit/delete a card
- import/export cards
- validate question/answer data

Key components:

- `CardTable`
- `CardEditorForm`
- `ImportCsvDialog`
- `ExportCsvButton`
- `DeleteCardDialog`

## 16.4 Review feature

Responsibilities:

- initialise queue
- manage flip and scoring actions
- re-queue incorrect cards
- track flagged cards
- persist resumable state
- complete session

Key components:

- `ReviewProgressBar`
- `ReviewCardStage`
- `ReviewCard`
- `ReviewActions`
- `ReviewExitDialog`

## 16.5 Results feature

Responsibilities:

- load completed session
- display score and breakdown
- display per-card outcomes
- support retake variants

Key components:

- `ResultsSummary`
- `ResultsDonutChart`
- `OutcomeList`
- `RetakeActions`

## 16.6 Settings feature

Responsibilities:

- manage local app preferences
- provide toggles/select controls
- expose preferences to review UI

Key components:

- `SettingsForm`
- `SettingsToggle`
- `SettingsSelect`

---

## 17. Review Engine Specification

This is the most important behavioural subsystem in v1.

## 17.1 Review session modes

- `full` — all cards in set
- `flagged` — only previously flagged cards from a completed session
- `incorrect-only` — only incorrect cards from a completed session

## 17.2 Queue initialisation

When a session starts:

1. fetch cards in the target set
2. reject start if no cards exist
3. build initial queue as ordered card IDs
4. apply shuffle if setting enabled
5. create session record
6. persist active session snapshot
7. navigate into review screen

## 17.3 Card lifecycle

Each current card moves through these states:

- `question-visible`
- `answer-visible`
- `resolved`

## 17.4 Allowed events

- `FLIP_CARD`
- `MARK_CORRECT`
- `MARK_INCORRECT`
- `MARK_FLAGGED`
- `NEXT_CARD`
- `PREVIOUS_CARD` (visual navigation only where supported)
- `EXIT_SESSION`
- `RESUME_SESSION`
- `COMPLETE_SESSION`

## 17.5 Outcome rules

### Correct
- record current card outcome as `correct`
- remove from active queue
- do not reinsert
- move to next available card

### Incorrect
- record outcome as `incorrect`
- remove current occurrence from queue
- reinsert at `currentPosition + 3` or queue end if shorter
- move to next available card

### Flagged
- record outcome as `flagged`
- add card to flagged list if not already present
- remove from active queue
- do not count as correct or incorrect
- move to next available card

## 17.6 Completion rules

When queue is empty:

1. finalise session in `sessions`
2. write all `results`
3. update `stats`
4. delete `activeSessions` snapshot
5. navigate to results route

## 17.7 Resume rules

On app load or set-detail screen open:

- if an active session exists for the set, offer resume
- if active session data is corrupt or points to deleted cards, discard snapshot and show safe fallback

## 17.8 Reducer requirement

The review engine should be implemented using a reducer plus pure helper functions, not scattered event handlers. This is required for testability.

---

## 18. CRUD Flows

## 18.1 Create Pack

Inputs:

- name
- colour

Validation:

- name required
- trim leading/trailing whitespace
- colour required from approved palette or validated hex value

On success:

- create record
- navigate to new pack detail

## 18.2 Create Set

Inputs:

- packId
- title
- optional description

Validation:

- parent pack must exist
- title required

On success:

- create record
- navigate to set detail

## 18.3 Create/Edit Card

Inputs:

- question
- answer

Validation:

- both required
- whitespace-only values invalid

On success:

- persist card
- return to set detail or remain in editor based on UX choice

## 18.4 Delete cascade

### Delete pack
Must remove:

- pack
- descendant sets
- descendant cards
- sessions linked to descendant sets
- results linked to those sessions
- stats linked to deleted cards
- active session snapshots linked to deleted sets

### Delete set
Must remove:

- set
- descendant cards
- sessions for set
- results for those sessions
- stats for deleted cards
- active session snapshots for set

All destructive actions must use confirmation dialogs.

---

## 19. CSV Import/Export Specification

## 19.1 Import

Accepted format:

- `.csv`
- required headers: `question`, `answer`
- max 500 rows per import

Validation rules:

- missing required headers -> hard fail
- empty question or answer row -> skipped
- whitespace-only value -> skipped
- extra columns ignored

Import flow:

1. user chooses CSV file
2. file parsed client-side
3. header validation runs
4. row validation runs
5. valid rows inserted in a single transaction
6. post-import summary shown

Result message example:

- `Imported 47 cards. 3 rows skipped.`

## 19.2 Export

Export the selected set to a `.csv` with exact columns:

- `question`
- `answer`

Export must preserve plain text values but does not need to preserve original row order if the app uses current set ordering.

---

## 20. Settings Architecture

Settings are application-level preferences and do not require relational queries.

## 20.1 Storage

Store settings under a single localStorage key, for example:

- `flashcard-app:settings`

## 20.2 Supported settings

- shuffle cards on review
- card flip animation
- show answer automatically
- swipe gestures
- study reminder notifications

## 20.3 Access pattern

Use a `SettingsProvider` or `useSettings()` hook that:

- loads settings at app startup
- supplies defaults when absent
- writes updates immediately
- exposes a stable typed API to feature code

---

## 21. UI and Component Standards

## 21.1 Shared UI primitives

The app should define reusable components for:

- Button
- IconButton
- CardSurface
- TextInput
- TextArea
- Select
- Toggle
- PageHeader
- EmptyState
- ConfirmDialog
- Toast
- ProgressBar
- Badge

## 21.2 Styling rules

- keep spacing and radii tokenised
- maintain strong contrast for dark theme
- avoid one-off component styling where a shared primitive can be used
- preserve visual distinction between question and answer card surfaces

## 21.3 Responsive policy in Phase A

The initial implementation is desktop-first but should still avoid breaking on smaller widths. This means:

- no desktop-only fixed widths that overflow narrow screens
- tables/list rows collapse gracefully
- forms stack vertically when needed
- the review screen remains usable on tablet widths

This is not yet full mobile optimisation, but it prevents costly rewrites later.

---

## 22. Error Handling and Edge States

Every feature must explicitly define expected failures.

## 22.1 Route and entity errors

- pack not found
- set not found
- card not found
- session not found
- malformed route parameter

## 22.2 Validation errors

- blank question/answer
- invalid CSV headers
- no valid CSV rows
- over-limit import file

## 22.3 Runtime data integrity issues

- active session references deleted card
- results exist for deleted session
- local storage settings parse failure
- IndexedDB unavailable or blocked

## 22.4 UX behaviour

- user-safe message shown
- log detailed error in development console
- allow recovery where possible
- never leave user on a broken blank screen

---

## 23. Accessibility Requirements

Accessibility is a release requirement, not a later enhancement.

### Required behaviours

- all interactive controls keyboard reachable
- visible focus states
- buttons have text or accessible labels
- review card flippable by keyboard
- swipe-only actions must always have button equivalents
- colour is not the only meaning carrier for outcomes
- `aria-live` announcements for review progress and score updates
- modals trap focus and restore focus on close

---

## 24. Performance Requirements

## 24.1 Targets

- route transitions should feel instant under local-only conditions
- review actions should not visibly lag
- importing 500 cards should complete without freezing the UI excessively

## 24.2 Strategies

- keep Dexie queries indexed appropriately
- use bulk inserts for CSV import
- memoise expensive derived calculations where useful
- avoid unnecessary re-renders in the review screen
- store review queue as IDs, not full card objects where possible

---

## 25. Testing Strategy

## 25.1 Unit tests

Required for:

- review queue logic
- incorrect-card reinsertion logic
- stats derivation logic
- CSV row validation
- settings parsing/defaulting

## 25.2 Integration tests

Required for:

- create pack -> create set -> create card -> review flow
- CSV import into set
- session completion and results rendering
- resume interrupted session
- pack/set delete cascade

## 25.3 Manual QA checklist

- invalid route handling
- keyboard-only review flow
- long-card text behaviour
- empty-state screens
- corrupted local settings fallback
- browser refresh during active session

---

## 26. Deployment and Runtime

## 26.1 Hosting

- deploy on Vercel
- production from `main`
- preview deployments from pull requests

## 26.2 Environment needs

Phase A requires no runtime secrets.

## 26.3 Browser support

Target modern evergreen browsers with IndexedDB support.

---

## 27. Phase A Acceptance Criteria

The pre-mobile foundation is complete when the following are true:

### App shell and routing
- all core routes exist
- standard and review layouts are both implemented
- not-found and route error states exist

### Data and persistence
- packs, sets, cards, sessions, results, stats persist locally
- settings persist in localStorage
- delete cascades work correctly

### CRUD and import/export
- packs, sets, and cards can be created, edited, and deleted
- CSV import validates headers and rows
- CSV export works from set detail

### Review engine
- session starts only on non-empty sets
- cards flip correctly
- correct/incorrect/flagged outcomes work as specified
- incorrect cards re-queue at position `+3`
- active session resumes after reload/close
- results screen shows completed session summary

### Accessibility and quality
- keyboard navigation works
- obvious empty states exist
- long text is handled safely
- major happy-path tests pass

---

## 28. Phase B — PWA and Mobile Additions

This section describes the changes added after the Phase A foundation is stable.

## 28.1 Architectural intent

The PWA layer should add runtime capabilities, not replace the underlying feature architecture. Domain logic, repositories, route structure, and primary screen composition should remain intact.

## 28.2 New responsibilities introduced in Phase B

- app installability
- offline shell boot after first load
- standalone display mode support
- connection status awareness
- touch gesture support in review
- notification permission management
- reminder scheduling hooks where browser support allows

---

## 29. PWA Additions to the Stack

Add:

- `vite-plugin-pwa`
- manifest configuration
- service worker registration helper
- offline/install hooks

No change is required to Dexie, React Router, or the feature service layer.

---

## 30. PWA Runtime Architecture

## 30.1 Manifest

Provide a web manifest including:

- app name
- short name
- theme colour
- background colour
- display mode: `standalone`
- icon set
- start URL

## 30.2 Service worker responsibilities

The service worker should:

- cache app shell assets after first successful load
- enable offline reopen after initial install/use
- version caches cleanly between releases
- avoid caching mutable database content because IndexedDB already handles user data

## 30.3 Caching strategy

Recommended initial strategy:

- **cache-first** for app shell assets (HTML shell, JS bundles, CSS, fonts, icons)
- **network-first** only for future remote APIs if later introduced
- **stale cache cleanup** on deploy version changes

## 30.4 Install flow

Expose an install prompt only when the browser supports it and the app meets installability criteria. This should be handled via a lightweight install service/hook.

---

## 31. Mobile and Touch Additions

## 31.1 Layout changes

When the PWA layer is added, layouts should adapt for narrow screens:

- navigation rail collapses into top bar / menu
- page containers use tighter padding
- card tables become stacked lists on small widths
- review actions become thumb-reachable bottom controls

## 31.2 Review interaction changes

Add gesture support to review:

- swipe right -> correct
- swipe left -> incorrect
- swipe up -> flag

Rules:

- gestures must be optional via settings
- button actions remain present at all times
- gesture thresholds must prevent accidental scoring
- gesture feedback should be visual before commit

## 31.3 Standalone mode adjustments

When launched as an installed app:

- top spacing should account for standalone/mobile browser chrome differences
- avoid assuming browser back button availability
- ensure in-app navigation remains obvious

---

## 32. Offline Behaviour After PWA Enablement

## 32.1 Before first successful load
A network connection is still needed to load the application initially.

## 32.2 After first successful load
The user should be able to:

- reopen the app offline
- browse existing packs and sets
- create/edit/delete local content
- review sets
- view results and stats
- import local CSV files
- export local CSV files

## 32.3 Offline indicators

The PWA layer should provide:

- passive offline indicator when network is unavailable
- no blocking banner for local-only actions
- user messaging that data remains stored on device

---

## 33. Notification Additions

## 33.1 Scope

Study reminders are part of settings, but only become meaningfully implementable in Phase B.

## 33.2 Requirements

- permission must be explicitly requested
- app must still work fully if permission is denied
- reminder settings UI should clearly explain browser/device limitations

## 33.3 Practical limitation

Notification scheduling support varies by browser and OS. For this reason, reminder notifications should be treated as a best-effort enhancement, not a guaranteed cross-platform feature.

---

## 34. Phase B Codebase Additions

Recommended additional folders:

```text
src/
  pwa/
    manifest/
    sw/
    install/
    offline/
    notifications/
```

Suggested modules:

- `useInstallPrompt()`
- `useOnlineStatus()`
- `registerServiceWorker()`
- `notificationService.ts`
- `installBanner.tsx`

---

## 35. Phase B Acceptance Criteria

The PWA/mobile layer is complete when:

### Installability
- manifest is valid
- install prompt can appear on supported browsers
- installed app opens in standalone mode

### Offline
- app shell is available offline after first successful load
- local data still works offline
- offline indicator behaves correctly

### Mobile
- review flow is usable on mobile widths
- swipe gestures work when enabled
- button fallback always remains available

### Notifications
- permission request is handled safely
- reminder setting degrades gracefully where unsupported

---

## 36. Future Compatibility Notes

This architecture is designed to support future additions without major restructuring:

- spaced repetition algorithms can be added inside the review/domain layer
- Supabase or another backend can be introduced via a sync layer without discarding Dexie
- accounts and auth can sit above the current local-first model
- shared decks can be added as a publishing/import layer

The key is that domain types, review logic, and repository boundaries remain stable.

---

## 37. Implementation Order Recommendation

## Phase A sequence

1. project bootstrap and shared UI primitives
2. app shell and route scaffolding
3. Dexie schema and repositories
4. packs/sets/cards CRUD
5. CSV import/export
6. review engine and active session resume
7. results and stats
8. settings
9. testing and polish

## Phase B sequence

1. manifest and service worker setup
2. offline shell validation
3. install prompt flow
4. mobile layout pass
5. swipe gestures
6. notification support
7. standalone-mode polish and QA

---

## 38. Final Engineering Position

The Flashcard App should be built as a desktop-first, local-first React application with a stable feature-based architecture. The first release must focus on correctness, persistence, and a clean review engine. The PWA/mobile layer should be introduced as a controlled second phase that enhances installability, offline boot, and touch interactions without disturbing the underlying domain and storage architecture.

This separation is deliberate: Phase A makes the app reliable; Phase B makes it portable and app-like.
