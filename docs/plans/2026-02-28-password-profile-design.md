# Password Profile & Progress Persistence Design

**Date:** 2026-02-28
**Status:** Approved

## Overview

Add a password-based profile system to `vocab-flashcards.html` so progress persists across sessions on the same device. No backend required — all data lives in `localStorage`.

## What Gets Saved

| Data | Trigger |
|------|---------|
| Selected sections + direction preference | On every toggle/change |
| Learn mode SR data (per Spanish word) | After every card rating |
| Quiz history (date, score, total) | After each quiz completes |
| Match best score (lowest misses) | After each match completes |

Flashcard mode has no persistent state.

## Storage Structure

Key: `vocab_profile_<password>`

```json
{
  "direction": "es-en",
  "selectedSections": ["Lectura: Latinx", "..."],
  "srData": {
    "Abarcar": { "interval": 4, "easeFactor": 2.5, "reps": 3, "graduated": true }
  },
  "quizHistory": [
    { "date": "2026-02-28", "score": 17, "total": 20 }
  ],
  "matchBest": 3
}
```

SR data is keyed per Spanish word so changing which sections are selected doesn't lose mastery history.

## UX Flow

1. Page loads → Password screen (input + Enter button)
2. Enter password → load existing profile or create fresh one → go to menu
3. Menu shows saved stats: "Last quiz: 17/20 · Best match: 3 misses"
4. "Sign out" link in menu returns to password screen
5. All saves are automatic (no manual save button)

## Architecture

- New `PasswordScreen` component rendered before `VocabApp` when no profile is loaded
- Profile state lives at top level, passed into `VocabApp` as props
- `saveProfile(password, profileData)` and `loadProfile(password)` utility functions wrap localStorage calls
- `VocabApp` calls save after: section toggle, direction change, learn card rating, quiz complete, match complete
- On `startLearn`, initialize deck from selected sections but merge in any saved `srData` per word
