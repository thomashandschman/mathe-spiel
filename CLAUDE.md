# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

"Mathe-Held" — a German-language math game for kids, built as a single
self-contained HTML file (inline CSS + vanilla JS, no build step, no
dependencies, no package manager). It is designed primarily as an iOS
add-to-home-screen PWA, so much of the CSS deals with safe areas, notches,
and per-device viewport tuning.

Content is aligned to the Austrian curriculum and uses standard primary-math
didactics (Fünferstruktur / "Kraft der Fünf", structured dot fields). There
are two age profiles:

- **Theo (1. Klasse Volksschule):** Zahlenraum 10/20, Plus/Minus, Zehnerübergang,
  Zahlzerlegung & verliebte Zahlen (Partner zu 10), Verdoppeln/Halbieren — every
  task is backed by a Zehner-/Zwanzigerfeld visual.
- **Lara (Kindergarten):** pre-numeric, picture-based — Zählen, Würfelbilder
  (subitizing), Mengenvergleich (mehr/weniger/gleich), Ziffer↔Menge.

## Running

Open the HTML file directly in a browser, or serve the folder statically:

```
python3 -m http.server 8000   # then open http://localhost:8000/
```

There are no builds, linters, or tests to run.

## The two-file sync constraint (important)

`index.html` and `mathe-spiel.html` are **byte-identical and must be kept that
way**. `mathe-spiel.html` is the original; `index.html` is the copy GitHub
Pages serves from the repo root. Any change to the game must be applied to
**both** files in the same commit. Verify with `diff index.html
mathe-spiel.html` before committing.

## Architecture

Everything lives in one file. Layout:

- **CSS** (`<style>`): CSS-variable theme in `:root`, `clamp()`-based fluid
  sizing throughout, and a stack of `@media` blocks at the end that retune
  spacing for specific device classes (iPhone Pro Max, SE, landscape). Touch
  behavior is locked down globally (`user-select: none`, tap-highlight off).
- **`state` object**: the single source of truth for a play session
  (`profile`, `range`, `types`, `level`, `xp`/`xpNeeded`, `stars`, `streak`,
  `correct`/`total`, `questionNum`, `current`, `answerEls`). `resetState()`
  reinitializes counters for a new round; `profile`/`range`/`types` are set
  from the setup screens.
- **Screen model**: `.screen` divs (`profile` → `theo`/`lara` setup → `game`
  → `summary`) plus a fixed level-up `.overlay`. Only one screen has the
  `.active` class at a time; `showScreen(name)` toggles it. There is no
  router — this class swap *is* the navigation.
- **Game loop**: `startGame()` → `nextQuestion()` → `handleAnswer()` →
  (after a ~1700ms delay) back to `nextQuestion()`, until `questionsPerRound`
  is reached, then `showSummary()`.
- **Question generation**: `generateQuestion()` picks a random enabled type
  for the active profile and dispatches to a per-type `build*()` function
  (`buildPlusMinus`, `buildZehner`, `buildZerlegung`, `buildDoppeln`,
  `buildZaehlen`, `buildWuerfel`, `buildVergleich`, `buildZuordnen`). Each
  returns a question object: `{ kind:'choice'|'compare', question, sub,
  visual (HTML string), options/left/right, answer, answerText }`. `nextQuestion`
  renders it; multiple-choice options go in `#answers`, `compare` questions use
  the tappable `#compareArea` (left card / `=` / right card). `handleAnswer`
  compares the chosen value (number or `'left'|'right'|'equal'`) to `q.answer`,
  using `state.answerEls` to highlight the correct choice on a miss.
- **Visual renderers** return HTML strings: `tenFrame`/`twentyFrame`/`dotRows`
  (built on `rowsHTML`) render structured dot fields with cell classes
  `on-a`/`on-b`/`empty`/`gone`/`want` and a `gap5` class for the Fünfer-gap;
  `diceHTML` draws dice pips; `objectsHTML` lays out emoji to count;
  `miniMenge` draws small quantity pictures for option buttons.
- **Learning animations** (CSS, with `prefers-reduced-motion` opt-out): filled
  dots/objects/pips animate in *staggered in fill order* (renderers emit inline
  `animation-delay`) so "adding" is visible; `want` cells pulse to mark the
  unknown. On a correct answer the visual cheers, and `q.revealType` drives a
  concept reveal — `'want'` fills the missing dots (`revealWant`, Zerlegung /
  verliebte Zahlen), `'gone'` fades the subtracted dots (`revealGone`).
- **Navigation chrome**: two fixed corner buttons outside `.game-container` —
  `#muteBtn` (always shown) and `#backBtn` (shown only on the game screen via
  `showScreen`), which returns to the active child's setup screen.

- **Persistence** (`localStorage`, key `matheheld_v1`): a single `store` object
  holds the global `muted` flag and per-profile stats (`totalStars`, `bestLevel`,
  `bestStreak`, `games`, `bestAccuracy`, plus `lastRange`/`lastTypes`).
  `commitProgress()` runs once per finished round (from `showSummary`) to
  accumulate totals and detect new records; `renderProfileStats()` shows each
  child's totals on the profile screen; `applySetup()` restores the last-used
  range/task-types when a profile is opened.
- **Sound** (Web Audio, no audio files): `tone()` synthesises notes; `sfxCorrect`
  / `sfxWrong` / `sfxLevelUp` / `sfxTap` are the cues. `ensureAudio()` lazily
  creates/resumes the `AudioContext` and must be triggered from a user gesture
  (done on profile selection and on each answer) — important for iOS, which
  blocks audio until then. All sfx no-op when `state.muted`; `#muteBtn` toggles
  and persists it.

When changing scoring/leveling, edit `handleAnswer`. To add or tune an exercise
type, edit the relevant `build*()` function (and register it in
`generateQuestion` + a chip in the setup screen). `makeOptions(answer, lo, hi)`
generates the distractors (3 distinct, in-range, non-negative).

A quick non-browser sanity check on the builders: stub `document`, concatenate
the inline `<script>` with assertions over many `generateQuestion()` runs, and
run it through `node` (verifies every answer is in its options, no dupes/negatives,
arithmetic is correct).

## Conventions

- UI strings are German; keep new user-facing text German and in the existing
  encouraging, kid-friendly tone (see the `PRAISE` / `ENCOURAGE` / `LEVEL_MSGS`
  arrays).
- DOM access goes through the `$ = id => document.getElementById(id)` helper.
- Keep it dependency-free — no frameworks, bundlers, or external scripts. The
  only external resource is the Nunito web font.
