# 1) the core learning loop

## Phase 1 — App shell and architecture

Build the React/Vite app skeleton, routing, shared layout, design tokens, and Dexie schema first. Do not start with gestures, charts, PWA caching, or polish. Start with the foundation: packs, sets, cards, sessions, results, stats, and the route map. If this layer is weak, everything else becomes rework.

**Deliverables:**

- App bootstrapped with Vite + React
- Router set up for Home, Pack, Set, Review, Results, Create, Settings
- Dexie tables created
- Basic reusable UI primitives: page shell, buttons, modals, forms, card component

## Phase 2 — Content management

Next, make the hierarchy usable end to end: create pack, create set, create card, edit card, delete card, list packs, list sets, list cards. This is the first place a developer can validate that the information architecture is sound. Until this works, there is nothing to review.

**Deliverables:**

- CRUD for Packs / Sets / Cards
- Cascade delete rules
- Empty states
- Validation for blank question/answer
- Set detail page with visible card list

## Phase 3 — Review session engine

This is the heart of the product. Build it before import/export, PWA behaviour, or settings. The review session is the main value proposition: flip card, mark correct/incorrect/flagged, re-queue incorrect cards, and finish the session with persisted results.

**Deliverables:**

- Start review from set
- One-card-at-a-time session flow
- Flip interaction
- Correct / Incorrect / Flag actions
- Re-queue logic for incorrect cards
- Session persistence to IndexedDB
- Resume interrupted session

# 2) viability

“Viable” means someone could realistically choose to use it instead of abandoning it after one session.

## Phase 4 — Results, stats, and repeat-study loops

Once review works, add the reason to come back: results screen, weak-card retake, flagged-card review, and stats updates. Without this, the app feels like a demo rather than a study tool. Your spec already defines results as a source-of-truth-driven flow, so this should follow naturally from the session model.

**Deliverables:**

- Results screen
- Correct / incorrect / flagged breakdown
- Review flagged
- Retake weak cards
- Retake full set
- Stats table updates at session end

## Phase 5 — Card ingestion and portability

Now add CSV import/export. Manual creation proves the model works, but CSV is what makes the product usable at scale. A lot of users will not hand-enter 100+ cards. This is one of the biggest “viability” features in your whole spec.

**Deliverables:**

- CSV parser with exact header validation
- Import summary with skipped row reporting
- Export set to CSV
- 100-card import cap
- Helpful import error messages

## Phase 6 — Reliability and usability layer

This is where the app starts to feel trustworthy. Add the edge cases and guardrails from the spec rather than treating them as later clean-up. These are not “nice to have” because they stop broken study sessions.

**Deliverables:**

- Disable review on empty sets
- Long-text scrolling inside cards
- Confirmation modals on destructive deletes
- Toasts and error states
- Keyboard support
- Accessible announcements for flip/score updates

## Phase 7 — Offline/PWA and installability

Only once the core app works in browser should you add service worker, manifest, installability, and offline app-shell caching. Your spec positions offline-first as important, but from an implementation standpoint it should be added after the local data and main flows are stable. Otherwise debugging gets messier too early.

**Deliverables:**

- PWA manifest
- Service worker via vite-plugin-pwa
- Offline app shell
- Install prompt / install testing
- Recovery testing after refresh / reopen

# 3) deferals/Enhancement

These are good features, but they should not block v1.

### Defer to v1.1 or later

- Swipe gestures
- Doughnut chart polish
- Study reminders
- Animations beyond basic flip
- Advanced settings
- Deep visual polish passes

### Defer to v2

- SM-2 spaced repetition
- Tags / categorisation
- Images in cards
- Shared decks
- Multi-device sync / auth / Supabase

Your own doc supports deferring auth and sync, and explicitly frames SM-2 as a future layer once the base loop is stable. That is the right call.

## Recommended roadmap

If this were a solo capable frontend dev, I’d frame it like this:

### Weeks 1–3: MVP

- App shell, routes, Dexie schema
- Packs / sets / cards CRUD
- Review session engine

### Weeks 4–6: Viable v1

- Results, weak-card / flagged loops, stats
- CSV import/export
- Error states, accessibility, resume flow
- PWA/offline, deploy to Vercel, device QA

### Weeks 7–10: Product polish

- Swipe gestures
- Better mobile interactions
- Settings polish
- Visual refinement and onboarding
- Usage analytics / instrumentation

### After that: strategic expansion

- Tags
- SM-2
- Accounts + sync
- Sharing / published decks

## What absolutely must come first

If I had to force the priority order, it would be:
- Data model and route structure
- Packs / sets / cards CRUD
- Review engine
- Session persistence + results
- CSV import/export
- Edge cases + accessibility
- PWA/offline packaging

That is the order because each step unlocks the next one. By contrast, swipes, charts, reminders, auth, sync, SM-2, tags, and sharing do not unlock the product — they enhance it.

## What makes it “viable”

To me, this app becomes viable when a user can:
- create or import a real deck,
- review it smoothly,
- stop and resume safely,
- see results,
- retake weak cards,
- use it offline,
- trust that they won’t lose data.

That is the bar. Everything beyond that improves retention or differentiation, but does not define whether the product works.

I can also turn this into a developer-ready roadmap table with columns for phase, tasks, dependencies, owner, estimate, and acceptance criteria.
