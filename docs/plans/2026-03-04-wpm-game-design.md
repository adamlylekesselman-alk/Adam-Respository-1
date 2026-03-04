# WPM Speed Reading Recall Game — Design

## Overview

A single-page HTML game where words from a sentence flash one-at-a-time in a fixed center spot (RSVP). After the sentence finishes, the player types it back. Correct recall advances to the next sentence at a higher WPM. Wrong → game over. Score = highest WPM reached.

## Game Flow

1. **Start screen** — title, high score, settings panel, Start button
2. **Countdown** — 3-2-1 Go
3. **Flash phase** — words appear one-at-a-time in center, progress bar shows position
4. **Recall phase** — input field appears, player types the sentence
5. **Result** — correct: show next sentence at higher speed; wrong: show game over with score
6. **Game over** — score display, high score update, Play Again button

## Settings (collapsible panel)

- Starting WPM: 100–500 (default 100)
- WPM step size: 50–200 (default 100)
- Words per round: 1–100 (default ~8, driven by sentence length)
- Mode: Real sentences (default) | Random words

## Scoring

- Score = highest WPM at which the player accurately recalled the sentence
- Stored in localStorage as high score

## Architecture

- Single `wpm-game.html` file — vanilla JS, no dependencies
- Dark theme matching other games in the project (pong.html)
- Sentences: ~30 pre-written sentences of varying length
- Random word mode: pool of ~200 common English words

## Visual Design

- Black background, white centered word, large font (4–6rem)
- Thin progress bar at top showing word index / total
- After flash: input centered below, auto-focused
- Speed label (e.g., "300 WPM") shown in corner during flash
