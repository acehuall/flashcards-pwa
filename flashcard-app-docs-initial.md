# Flashcard App

## Product Documentation

Version 1.0  |  March 2026

## Overview

A Progressive Web App (PWA) for creating and reviewing flashcard decks. Inspired by the structure of NotebookLM datasets, with improved mechanics, a clean dark UI, and offline-first capability.

Cards are organised in a three-level hierarchy:
- Packs — top-level grouping (e.g. "Medical School", "Coding")
- Sets — chapter or topic within a Pack
- Cards — individual question/answer pairs within a Set

## 1. Card Creation

Two methods for adding cards to a set:

### a) CSV Import

- Upload a .csv file with two columns: question and answer
- Column headers are required and must match exactly
- Rows with missing or empty values are skipped silently; a post-import summary reports skipped rows
- Maximum 500 cards per import
- Export is also supported — any set can be exported back to the same two-column CSV format

Example CSV format:
```
question,answer
What is the powerhouse of the cell?,The mitochondria
What year did WW2 end?,1945
```

### b) Manual Creation

- Create cards one at a time via a form with two text fields: Question and Answer
- Cards can be edited or deleted at any time from the set view
- No limit on card length, but overly long content will scroll within the card during review

## 2. Styling & Design

### Colour Palette

| Token          | Hex     | Usage                                |
| -------------- | ------- | ------------------------------------ |
| Background     | #1A1C1E | App background                       |
| Background Alt | #141414 | Alternate / deeper background        |
| Question Card  | #303030 | Question card surface                |
| Answer Card    | #1E2124 | Answer card surface                  |
| Primary Text   | #FFFFFF | All main text                        |
| Secondary Text | #909090 | Subtitles, labels, borders           |
| Correct        | #4CAF50 | Correct button / result indicator    |
| Incorrect      | #F44336 | Incorrect button / result indicator  |
| Flag           | #FFC107 | Flag button / flagged card indicator |
| Navigation     | #3F51B5 | Nav elements, primary actions        |

### Typography

- Font: Roboto (via Google Fonts) — Inter is an acceptable fallback

### Component Styling

- Cards: Rounded corners (border-radius 16px)
- Question card: No border
- Answer card: 1px solid #909090 (Secondary Text colour)
- Navigation and review action buttons: Pill / circle style
- Swipe gesture targets: Full card area

## 3. Screen Map

| Route               | Screen         | Description                                           |
| ------------------- | -------------- | ----------------------------------------------------- |
| /                   | Home           | Pack browser — lists all Packs with colour indicators |
| /pack/:id           | Pack Detail    | Lists all Sets within a Pack                          |
| /set/:id            | Set Detail     | Lists all Cards in a Set; import/export/edit actions  |
| /review/:setId      | Review Session | Full-screen card review mode                          |
| /results/:sessionId | Results        | End-of-session score screen                           |
| /create/pack        | Create Pack    | Form to name and colour a new Pack                    |
| /create/set/:packId | Create Set     | Form to name a new Set within a Pack                  |
| /create/card/:setId | Create Card    | Form to manually add a card to a Set                  |
| /settings           | Settings       | App-level preferences                                 |

## 4. Mechanics

### Hierarchy

Pack  (e.g. "Medical School")

 └── Set  (e.g. "Chapter 4 — Cardiology")

      └── Card  (question / answer pair)

### Review Session Flow

- User selects a Set and taps Start Review
- Cards are displayed one at a time, full-screen
- Tapping the card flips it to reveal the answer (horizontal flip animation)
- User selects one of three outcomes:
- Correct — card marked correct for this session
- Incorrect — card marked incorrect; re-queued later in the session
- Flag — card flagged for follow-up; does not affect score
- A progress indicator at the top shows remaining cards with coloured markers for flagged/incorrect cards
- Left/right navigation arrows allow manual card movement without scoring

### Swipe Gestures (mobile/touch)

Swipes are a first-class interaction, not an optional extra:
- Swipe right → Correct
- Swipe left → Incorrect
- Swipe up → Flag
- Button UI remains visible as a fallback on all screen sizes

### End of Session — Results Screen

- Score shown as a doughnut chart (correct / incorrect / flagged breakdown)
- Summary list of all cards with their outcome
- Four options presented:
- Review Flagged — step through flagged cards only
- Retake Full Set — restart from the beginning, reshuffled
- Retake Weak Cards — restart with only incorrect cards
- Exit — return to Set Detail

## 5. Data Schema (Dexie.js / IndexedDB)

| Store    | Fields                                                                               | Purpose                            |
| -------- | ------------------------------------------------------------------------------------ | ---------------------------------- |
| packs    | ++id, name, color, createdAt                                                         | Top-level grouping (e.g. "Coding") |
| sets     | ++id, packId, title, description, createdAt                                          | Chapter/topic, linked to a Pack    |
| cards    | ++id, setId, question, answer, createdAt                                             | Individual card, linked to a Set   |
| sessions | ++id, setId, startedAt, completedAt, score                                           | One review attempt on a Set        |
| results  | ++id, sessionId, cardId, outcome, timestamp                                          | Per-card outcome within a session  |
| stats    | ++id, cardId, correctCount, incorrectCount, flaggedCount, lastResult, lastReviewedAt | Aggregate card-level history       |

#### Key Design Decisions

- results is the source of truth for a session — stats is derived and updated at session end
- Keeping stats as a separate table avoids scanning all results rows for card-level summaries
- sessions records are kept so future features (e.g. historical graphs) have the data they need
- outcome field accepts: 'correct' | 'incorrect' | 'flagged'

## 6. Review Algorithm

The app uses a lightweight re-queue system (not full SM-2 in v1):
- Correct cards are removed from the active queue for the current session
- Incorrect cards are re-inserted into the queue at position current + 3 (seen again soon, not immediately)
- Flagged cards are removed from the active queue but tracked separately; they surface in Review Flagged at end of session
- At session end, stats.correctCount, stats.incorrectCount, and stats.flaggedCount are incremented for each card

ℹ  Future: SM-2 spaced repetition (intervals, ease factor, due dates) can be layered on top once the base loop is stable. The stats table is designed to support it.

## 7. Offline Behaviour

The app is offline-first. All core features work without a network connection:

| Feature               | Offline?            |
| --------------------- | ------------------- |
| Browse Packs and Sets | Yes                 |
| Review any Set        | Yes                 |
| Create cards manually | Yes                 |
| Import CSV            | Yes (file is local) |
| Export CSV            | Yes                 |
| View results / stats  | Yes                 |

The Service Worker caches the full app shell (HTML, JS, CSS, fonts) on first load. All data is stored in IndexedDB via Dexie.js — there is no backend, so there is nothing to sync.

ℹ  Network is only needed for: initial app load (first visit) and any future auth/sync features if added later.

## 8. Error States & Edge Cases

| Scenario                              | Behaviour                                                                         |
| ------------------------------------- | --------------------------------------------------------------------------------- |
| Review started on empty Set           | Button disabled; tooltip: "Add at least one card to start"                        |
| CSV import — no valid rows            | Toast error: "No valid rows found. Check column headers are question and answer." |
| CSV import — some rows skipped        | Post-import summary: "Imported 47 cards. 3 rows skipped (missing answer)."        |
| Card question or answer is blank      | Form validation prevents save; field highlighted in red                           |
| Mid-session app close                 | Session state persisted to IndexedDB; user offered to resume on next open         |
| Very long card text                   | Text scrolls within the card; card height does not expand beyond viewport         |
| Pack or Set deleted with cards inside | Confirmation modal warns of cascade delete; all child records removed             |

## 9. Settings & User Preferences

Accessible from /settings:

| Setting                      | Default | Options              |
| ---------------------------- | ------- | -------------------- |
| Shuffle cards on review      | On      | On / Off             |
| Card flip animation          | On      | On / Off             |
| Show answer automatically    | Off     | Off / 3s / 5s / 10s  |
| Swipe gestures               | On      | On / Off             |
| Study reminder notifications | Off     | Off / Daily / Custom |

ℹ  Settings are stored in a settings key in LocalStorage (not Dexie — no relational need).

## 10. Tooling & PWA Setup

### Core Stack

- Framework: Vite + React
- Hosting: Vercel (auto-deploy from main branch)
- Local storage: Dexie.js (IndexedDB wrapper)
- PWA plugin: vite-plugin-pwa — handles manifest and Service Worker generation

### PWA Requirements

#### a) Web Manifest

Defines app name, icons, theme colour, display mode (standalone). Generated automatically by vite-plugin-pwa.

#### b) Service Worker

Intercepts network requests and serves cached assets offline. Strategy: cache-first for app shell, network-first for any future API calls.

#### c) HTTPS

Required for Service Worker registration. Vercel provides HTTPS automatically on all deployments.

#### d) Local Storage (IndexedDB via Dexie.js)

All card data, session history, and stats stored client-side. No backend database in v1.

### Auth Decision

Deferred. No auth in v1 — all data is local to the device. If multi-device sync becomes a priority later, the most likely approach is Supabase (Postgres + Auth + Row Level Security) with a local-first sync strategy. The data schema is designed to be portable.

### Deployment

- Push to main → Vercel auto-deploys to production
- PRs generate preview deployments automatically
- Environment variables (if any added later) managed via Vercel dashboard

## 11. Accessibility

- Colour contrast for all text must meet WCAG AA minimum (4.5:1 for normal text)
- All interactive elements reachable by keyboard (Tab / Enter / Space)
- Card flip triggerable via Enter key as well as click/tap
- Swipe gestures always have a button equivalent — never gesture-only interactions
- aria-live region announces card flip and score updates to screen readers

## 12. Future Ideas

- Images in cards — attach an image to either the question or answer side
- Game modes — timed mode, multiple choice, match-the-pairs
- Tags / categorisation — tag cards within a set to allow focused sub-reviews
- SM-2 spaced repetition — full interval scheduling based on review history
- Multi-device sync — optional account + Supabase backend
- Shared decks — publish a Set with a shareable link
