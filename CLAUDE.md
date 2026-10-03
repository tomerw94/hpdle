# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Keep this file current

**Whenever you add a new feature, stage, data source, or change the deployment/data model, update this file in the same session** — add/edit the relevant section below before finishing. This is the only documentation future sessions get; if it goes stale, the next session will re-derive everything from scratch or make wrong assumptions (e.g. about Firestore rules, which are not in this repo).

## What this is

HPdle: a single-file Harry Potter daily guessing game, in two stages:
1. **Character guess** — Wordle-style daily character guess with colored feedback columns (gender/role/species/first book).
2. **Spell Challenge** — unlocked only after solving stage 1. Three multiple-choice questions, one per difficulty (Easy/Medium/Hard), about spells.

Both stages pick the same puzzle for every visitor on a given calendar day (deterministic daily seeding), and both report anonymous aggregate stats (solve counts, accuracy) to a shared Firestore collection.

## Commands

There is no build step, package manager, linter, or test suite. The entire app is `index.html` (inline `<style>` + inline `<script type="module">`), plus `HPfont.TTF` and `hp_spells.csv` (data source, not loaded at runtime — see below).

- **Run it**: open `index.html` directly in a browser (double-click, or `file://` path). No server needed.
- **Deploy it**: `git push origin main` — the repo is hosted on **GitHub Pages** at `https://tomerw94.github.io/hpdle/`, serving directly from `main`. Pushing to `main` is the entire deploy process.
- **Ad hoc logic testing**: there's no test framework. To verify JS logic changes without a browser, extract the `<script type="module">` body to a `.mjs` file, strip the `import` lines, and run it under Node with small stub objects for `document`, `localStorage`, and the Firebase functions (`initializeApp`, `getFirestore`, `doc`, `getDoc`, `setDoc`, `increment`). This catches reference errors and lets you assert on the daily-question logic, option generation, etc., without touching Firestore or a real DOM.

## Architecture

Everything lives in `index.html`:
- `<style>`: all CSS, dark/gold Harry-Potter theme, mobile breakpoint at the bottom.
- `<script type="module">`: Firebase imports, then data, then daily-selection logic, then stage 1 state/rendering, then stage 2 state/rendering, then event wiring, ending with `restoreProgress()` which kicks off restoring both stages on load.

### Daily determinism

Both stages reuse the same pattern: `seededRNG(seed)` (a small xorshift-style PRNG) + `buildShuffledOrder(n, seed)` (seeded Fisher-Yates) produce a fixed shuffle of indices; `dayNumber()` (days since a fixed `EPOCH` constant, using the browser's **local** date via `getLocalToday()`/`todayKey()`, not UTC) picks today's index via `day % length`. This guarantees the same character and the same 3 spell questions for every visitor on the same calendar day, without any server-side coordination.

- Stage 1: `DAILY_ORDER` + `getDailyTarget()` → `TARGET` (from `CHARACTERS`).
- Stage 2: `STAGE2_SEEDS` (one seed per difficulty) + `pickDailySpell()` → the correct answer per difficulty; `buildStage2Question()` then seeds a second RNG (seed XOR'd with the day number) to pick 3 decoy options from the **same difficulty only** and shuffle the 4 options into display order — also deterministic per day. `STAGE2_QUESTIONS` (built once at load) holds all 3 questions.

### Stage 2 question shape

Each `STAGE2_STEPS` entry has `promptField`/`answerField` pointing at keys on a spell object (`name` or `pronunciation`). Easy shows `pronunciation` and asks for `name`; Medium/Hard show `name` and ask for `pronunciation`. This is driven entirely by data shape, not separate per-difficulty UI code.

**The `clue` field (anti-giveaway override):** some spells have a name and pronunciation similar enough that showing one makes the other trivial to guess (e.g. "Impervius Charm" / "Impervius", or "Shield Charm" / "Protego" — the English root gives it away). For those spells only, `SPELLS` entries carry an extra hand-written `clue` field (a lowercase verb-phrase fragment, e.g. `'makes an object float and drift through the air'`). `renderStage2Question()` checks `q.target.clue` first: if present, it builds the prompt as `` `Which spell ${clue}?` `` (when asking for `name`) or `` `What is the pronunciation of the spell that ${clue}?` `` (when asking for `pronunciation`), bypassing the normal name/pronunciation prompt entirely. `clue` is **not** part of the CSV pipeline (see below) — it's authored directly in `index.html` per spell, specifically worded to avoid sharing a root with that spell's `pronunciation` (so don't reuse the stock `description` text verbatim; several of those also leak the root, e.g. "Erects..." for Erecto). If you re-sync `SPELLS` from a fresh CSV parse, you must manually re-add `clue` to the affected spells or the anti-giveaway fix is silently lost.

Spells where name ≈ pronunciation with **no reasonable clue fix** (e.g. literally identical strings, or minor variants not worth the authoring effort) have been removed entirely from both `SPELLS` and `hp_spells.csv` rather than patched — see the CSV pipeline note below.

### Stage gating / UI visibility

Stage 2's `<div id="stage2">` starts hidden. `showStage2()` reveals it **and hides the entire `.game-area` div** (stage 1's search box, table, win banner) — the intent is the player only ever sees their current stage, not a growing page of finished stages. If you add a stage 3, follow the same pattern: hide the previous stage's container when the next one opens.

### Persistence (per calendar day, per browser)

`localStorage` keys are namespaced by `todayKey()` so progress resets naturally each day:
- `hpdle_progress_<date>` — stage 1 (`guesses`, `won`, `statsRecorded`).
- `hpdle_stage2_<date>` — stage 2 (`step`, `results`, `currentAnswered`, `currentChosenName`, `finished`, `statsRecorded`).

Old dated keys are never cleaned up (harmless, grows slowly over months).

### Firestore (shared aggregate stats only — no auth, no per-user data)

Single collection `dailyStats`, one doc per day (`doc(db, 'dailyStats', todayKey())`), written via `setDoc(..., { merge: true })` with `increment()`. Fields: `solvedCount`/`totalGuesses` (stage 1, written by `recordWinStats`) and `stage2FinishedCount`/`stage2CorrectAnswers`/`stage2TotalAnswers` (stage 2, written by `finishStage2`). Each write is guarded by a `statsRecorded`/`stage2StatsRecorded` flag persisted in `localStorage` so a solve/finish is only ever counted once per browser per day, and retried on next load if the write failed.

**Firestore security rules are not in this repo** — they're set directly in the Firebase console (Firestore Database → Rules) and must be manually kept in sync with the fields the client writes. They use a strict `request.resource.data.keys().hasOnly([...])` whitelist on the *entire resulting document*, validated per-field-group via `diff(resource.data).affectedKeys()` (so a stage-1-only write isn't forced to satisfy stage-2 field constraints and vice versa). **Adding any new field written to `dailyStats` requires updating these console-side rules in the same change, or the write will silently fail client-side** (caught and logged to console, but otherwise invisible — this has already caused a real bug once). Firebase project config (`firebaseConfig` in the script) is not secret; it's gated entirely by these rules.

### Spell data pipeline (`hp_spells.csv`)

**`hp_spells.csv` is a source file, not loaded at runtime.** It's parsed by hand (a one-off Node script, not checked into the repo) into the inline `SPELLS` array in `index.html`, the same way `CHARACTERS` is a hand-maintained inline array. **Editing the CSV does nothing to the live game until someone re-parses it and replaces the `SPELLS` array block in `index.html`.**

Current column mapping (the CSV header names these explicitly):
1. `#` — ignored.
2. `Name (Spell / Description Title)` → `name` (a plain-English title, e.g. "Killing Curse" — NOT the Latin incantation).
3. `Incantation (Pronunciation Answer)` → `pronunciation` (the actual spoken word, e.g. "Avada Kedavra" — this is what Stage 2 treats as "the pronunciation", despite not being a phonetic spelling).
4. `Description` → `description`.
5. `Seen / Mentioned` → `seen`.
6. `Difficulty` → `difficulty` (`Easy`/`Medium`/`Hard`; `SPELLS_BY_DIFFICULTY` groups by this at load).

Watch for stray Windows-1252 bytes in the CSV (e.g. a raw `0x96` en-dash) — read it as `latin1` and remap known bytes rather than assuming clean UTF-8, or you'll get mangled characters in descriptions.

**Curated removals:** 9 spells were deliberately dropped from both `hp_spells.csv` and `SPELLS` (currently 98 spells, 28/37/33 by Easy/Medium/Hard) because their name and pronunciation were too similar to fix with a `clue` (see above) — either literally identical strings (Age Line, Anti-Cheating Spell, Bat-Bogey Hex, Caterwauling Charm, Cheering Charm, Fiendfyre) or niche variants not worth custom-authoring a clue for (Protective Black-Fire Ring / Protego Diabolica, Dark Shield Charm / Protego Horribilis, Boundary Shield Charm / Protego Totalum — all "Protego X" variants whose base spell, Shield Charm / Protego, was kept with a clue instead). If re-adding spells from a different CSV export, re-check new entries for this same name/pronunciation-similarity problem before merging.

## Data files

- `index.html` — the entire app.
- `hp_spells.csv` — spell reference data; source-of-truth for `SPELLS`, but must be manually re-synced into `index.html` after edits (see above).
- `HPfont.TTF` — custom display font, used only for the `<h1>` title via `@font-face`.
