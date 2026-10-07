# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Keep this file current

**Whenever you add a new feature, stage, data source, or change the deployment/data model, update this file in the same session** — add/edit the relevant section below before finishing. This is the only documentation future sessions get; if it goes stale, the next session will re-derive everything from scratch or make wrong assumptions (e.g. about Firestore rules, which are not in this repo).

## What this is

HPdle: a single-file Harry Potter daily guessing game, in three stages:
1. **Character guess** — Wordle-style daily character guess with colored feedback columns (gender/role/species/house/birth year/first book).
2. **Spell Challenge** — unlocked only after solving stage 1. Three multiple-choice questions, one per difficulty (Easy/Medium/Hard), about spells.
3. **Quote Challenge** — unlocked only after finishing stage 2. One daily movie quote: guess who said it (free-text search over the character list), then — regardless of whether that guess was right — guess which of the 8 films it's from (same free-text search UX, over movie titles).

All three stages pick the same puzzle for every visitor on a given calendar day (deterministic daily seeding), and all three report anonymous aggregate stats (solve counts, accuracy) to a shared Firestore collection.

## Commands

There is no build step, package manager, linter, or test suite. The entire app is `index.html` (inline `<style>` + inline `<script type="module">`), plus `HPfont.TTF`, `hp_spells.csv`, and `hp_quotes.csv` (data sources, not loaded at runtime — see below).

- **Run it**: open `index.html` directly in a browser (double-click, or `file://` path). No server needed.
- **Deploy it**: `git push origin main` — the repo is hosted on **GitHub Pages** at `https://tomerw94.github.io/hpdle/`, serving directly from `main`. Pushing to `main` is the entire deploy process.
- **Ad hoc logic testing**: there's no test framework. To verify JS logic changes without a browser, extract the `<script type="module">` body to a `.mjs` file, strip the `import` lines, and run it under Node with small stub objects for `document`, `localStorage`, and the Firebase functions (`initializeApp`, `getFirestore`, `doc`, `getDoc`, `setDoc`, `increment`). This catches reference errors and lets you assert on the daily-question logic, option generation, etc., without touching Firestore or a real DOM.

## Architecture

Everything lives in `index.html`:
- `<style>`: all CSS, dark/gold Harry-Potter theme, mobile breakpoint at the bottom.
- `<script type="module">`: Firebase imports, then data, then daily-selection logic, then stage 1 state/rendering, then stage 2 state/rendering, then stage 3 state/rendering, then event wiring, ending with `restoreProgress()` which kicks off restoring all three stages on load (it chains into `restoreStage2Progress()`, which chains into `restoreStage3Progress()`).

### Daily determinism

All three stages reuse the same pattern: `seededRNG(seed)` (a small xorshift-style PRNG) + `buildShuffledOrder(n, seed)` (seeded Fisher-Yates) produce a fixed shuffle of indices; `dayNumber()` (days since a fixed `EPOCH` constant, using the browser's **local** date via `getLocalToday()`/`todayKey()`, not UTC) picks today's index via `day % length`. This guarantees the same character, the same 3 spell questions, and the same quote for every visitor on the same calendar day, without any server-side coordination.

- Stage 1: `DAILY_ORDER` + `getDailyTarget()` → `TARGET` (from `CHARACTERS`).
- Stage 2: `STAGE2_SEEDS` (one seed per difficulty) + `pickDailySpell()` → the correct answer per difficulty; `buildStage2Question()` then seeds a second RNG (seed XOR'd with the day number) to pick 3 decoy options from the **same difficulty only** and shuffle the 4 options into display order — also deterministic per day. `STAGE2_QUESTIONS` (built once at load) holds all 3 questions.
- Stage 3: `QUOTE_ORDER` + `pickDailyQuote()` → `DAILY_QUOTE` (from `QUOTES`), the exact same single-array daily-index pattern as Stage 1's `TARGET` (there's no decoy generation to seed, since both sub-answers are free-text guesses, not multiple choice).

### Stage 2 question shape

Each `STAGE2_STEPS` entry has `promptField`/`answerField` pointing at keys on a spell object (`name` or `pronunciation`). Easy shows `pronunciation` and asks for `name`; Medium/Hard show `name` and ask for `pronunciation`. This is driven entirely by data shape, not separate per-difficulty UI code.

**The `clue` field (anti-giveaway override):** some spells have a name and pronunciation similar enough that showing one makes the other trivial to guess (e.g. "Impervius Charm" / "Impervius", or "Shield Charm" / "Protego" — the English root gives it away). For those spells only, `SPELLS` entries carry an extra hand-written `clue` field (a lowercase verb-phrase fragment, e.g. `'makes an object float and drift through the air'`). `renderStage2Question()` checks `q.target.clue` first: if present, it builds the prompt as `` `Which spell ${clue}?` `` (when asking for `name`) or `` `What is the pronunciation of the spell that ${clue}?` `` (when asking for `pronunciation`), bypassing the normal name/pronunciation prompt entirely. `clue` is **not** part of the CSV pipeline (see below) — it's authored directly in `index.html` per spell, specifically worded to avoid sharing a root with that spell's `pronunciation` (so don't reuse the stock `description` text verbatim; several of those also leak the root, e.g. "Erects..." for Erecto). If you re-sync `SPELLS` from a fresh CSV parse, you must manually re-add `clue` to the affected spells or the anti-giveaway fix is silently lost.

Spells where name ≈ pronunciation with **no reasonable clue fix** (e.g. literally identical strings, or minor variants not worth the authoring effort) have been removed entirely from both `SPELLS` and `hp_spells.csv` rather than patched — see the CSV pipeline note below.

### Stage 3 question shape

Unlike Stage 2 (3 multiple-choice questions across difficulties), Stage 3 is a single daily quote (`DAILY_QUOTE`, from `QUOTES`) with two sequential free-text sub-answers, each a single attempt (right or wrong, it locks and moves on — no retries):
1. **Speaker** — `renderStage3Question()` shows the quote text and a character-name search input (reusing the `CHARACTERS` name list as the autocomplete pool). `submitStage3SpeakerGuess()` checks the guess against `DAILY_QUOTE.speaker`, reveals the result, then **always** reveals the movie sub-step regardless of correctness.
2. **Movie** — a second search input over the fixed 8-title `MOVIES` array. `submitStage3MovieGuess()` checks against `DAILY_QUOTE.movie` and reveals the result (plus `DAILY_QUOTE.situation` as bonus trivia) alongside a `stage3-finish-btn` ("See Final Results"). It does **not** auto-advance to the summary — the player must see the movie result and click through, same as Stage 2's result-then-"Next Level"/"Finish"-button pattern (see "Per-stage summary dialogs" below). Clicking the button calls `finishStage3()`.

Both inputs are driven by a small shared `setupAutocomplete({ inputEl, listEl, buttonEl, options, onSubmit })` helper — a parameterized version of Stage 1's hardcoded search-input logic (substring match via `.includes()`, 8-result cap, arrow-key nav, blur-delay-for-click), used twice (once per sub-step) since Stage 1's original autocomplete code is hardcoded against its own module-level singletons and isn't reusable as-is. Both this helper and Stage 1's own `showDropdown()` match anywhere in the name/title, not just the start — e.g. typing "dumb" finds "Albus Dumbledore" (this was changed from a `.startsWith()` prefix match after it shipped; keep both in sync if you touch one).

`QUOTES[].speaker` must always equal a `CHARACTERS[].name` string exactly, and `QUOTES[].movie` must always equal one of the 8 `MOVIES` strings exactly — this is what the CSV curation step (see the quote data pipeline below) guarantees.

### Per-stage summary dialogs / continue pattern

Every stage ends on a result the player can actually see and dismiss on their own terms — never an instant auto-transition. This is a deliberate, repeat-this-for-any-new-stage rule:
- **Mid-stage results** (e.g. each Stage 2 question, each Stage 3 sub-answer) reveal a correct/incorrect result and pause there; the player clicks an explicit button (`stage2-next-btn`, `stage3-finish-btn`) to move on. Never call the next stage/step's render function directly from inside a submit handler without that click in between — the first Stage 3 implementation got this wrong for the movie sub-answer (it called `finishStage3()` synchronously right after rendering the result, so the result DOM update and the summary's hide-this-container update both happened in the same tick and the browser never painted the in-between state — the player never saw whether their last guess was right). If a future stage needs a similar pause, give it its own named button and wire the "proceed" call only to that button's click handler.
- **End-of-stage summary dialogs** (`#win-banner`, `#stage2-summary`, `#stage3-summary`) show the player's result for *that* stage, then a "Continue to `<next stage>`" button if there's a next stage. The button only appears once the current stage is actually finished (`stage2-start-btn`/`stage3-start-btn` are hidden by default and revealed via JS).
- The **final stage's summary is the combined final-results screen**: `#stage3-summary` (currently the last stage) recaps **all three** stages — character found in N guesses, Spell Challenge score, Quote Challenge score — followed by all three global stats lines (`displayStage1StatsForFinalSummary()`, `displayStage2StatsForFinalSummary()`, `displayStage3Stats()`), then the "come back tomorrow" message. If you add a Stage 4, move this combined recap + "come back tomorrow" message to the new final stage's summary, and turn `#stage3-summary` back into a normal mid-game summary with its own "Continue to Stage 4" button.
- `displayStage1StatsForFinalSummary()`/`displayStage2StatsForFinalSummary()` are intentionally near-duplicates of `displayDailyStats()`/`displayStage1StatsForSummary()` and `displayStage2Stats()` (same Firestore reads, different target element ids) rather than a single parameterized function — consistent with how this codebase already duplicates small per-screen display functions instead of abstracting early; only generalize if a 4th near-identical call site shows up.

### Character hint columns (`CHARACTERS` fields → `compare()` → `renderRow()`)

Each `CHARACTERS` entry has `gender`, `roles` (array), `species` (array), `firstAppearance`, `house`, and `birthYear`. `compare(guess)` derives a result object with one key per column (`gender`, `roles`, `species`, `house`, `birthYear`, `book`), and `renderRow()` renders them in that same column order (see the `<thead>` in `index.html` for the matching header order: Character/Gender/Role/Species/House/Birth Year/First Appearance).

- `gender` and `house` are simple equality checks → green/red, no arrow.
- `roles` and `species` are array comparisons → green (exact set match), orange (overlap), red (no overlap), via `setsEqual`/`setsOverlap`.
- `firstAppearance` (the `book` key in the result) is ordinal: green on exact match, else red with an `↑`/`↓` arrow pointing toward the target, via index position in `BOOK_ORDER`.
- `birthYear` is a **range**, either `[minYear, maxYear]` (a single known year is just `[Y, Y]`) or the literal string `"Unknown"`. `compareBirthYear()`: green if the guess's range is identical to the target's; orange if the ranges overlap but aren't identical (covers both partial overlap and one range containing the other); red with an `↑`/`↓` arrow if the ranges don't overlap at all (arrow points toward the target, based on whether the guess's range is entirely before or after it); red with no arrow if either side is `"Unknown"` (direction/overlap can't be determined). `formatBirthYear()` renders `[Y, Y]` as `"Y"` and `[min, max]` as `"min–max"` for display.
- `house` is one of `Gryffindor`/`Slytherin`/`Hufflepuff`/`Ravenclaw`/`"-"` (character never had a Hogwarts house — non-human, attended a different school, or no house ever confirmed in canon).

**Data provenance / accuracy caveat:** `birthYear`/`house` were populated via web research (Harry Potter Wiki) rather than from a source file, since no CSV/dataset existed for this character data. One entry, **Corban Yaxley**, was added specifically to support Stage 3 (his line "Magic is might." needed a valid `CHARACTERS` match) — same research provenance, species/role/house/birthYear filled in by the same web-research convention as the rest of the roster (Death Eater, `"Unknown"` birth year, `"-"` house — his house is never confirmed in canon). Exact single-year birthdates (stored as `[Y, Y]`) are only canon-solid for major characters with a stated birthday (e.g. Harry `[1980,1980]`, Hermione `[1979,1979]`). Many secondary characters only have a Hogwarts-school-year window in canon (e.g. "born between Sept 1979 and Aug 1980") — those are stored as the real `[1979, 1980]` two-year range rather than a guessed single year, which is what feeds the orange/overlap comparison. Characters with **no** usable range at all in canon (not even a school-year window — true for most non-students, creatures, ghosts, founders, and several background humans) are left `"Unknown"`; this intentionally includes a handful of characters that only have a one-sided bound in canon (e.g. "born before 1961" with no lower limit) — a one-sided range was judged too wide to give a meaningful orange/red signal, so those stay `"Unknown"` rather than being encoded with a fabricated far-end bound. `house` is `"-"` both for characters who truly never had one (non-Hogwarts, non-human) and for characters who likely attended Hogwarts but whose house is never confirmed in canon (e.g. Barty Crouch Sr./Jr., Cornelius Fudge, Rita Skeeter). If you have better-sourced info for any `"Unknown"`/`"-"` entry, it's safe to just edit that character's object in place.

### Stage gating / UI visibility

Stage 2's `<div id="stage2">` starts hidden. `showStage2()` reveals it **and hides the entire `.game-area` div** (stage 1's search box, table, win banner) — the intent is the player only ever sees their current stage, not a growing page of finished stages. Stage 3 follows the identical pattern one level up: `showStage3()` reveals `<div id="stage3">` and hides `#stage2` (including its `#stage2-summary`). The `stage2-start-btn` lives in `#win-banner`; the analogous `stage3-start-btn` lives inside `#stage2-summary` and is only revealed once Stage 2 finishes (via `restoreStage3Progress()`, called at the end of `renderStage2Summary()` — see Persistence below). If you add a stage 4, follow the same pattern: hide the previous stage's container when the next one opens, and put the "continue" button inside the previous stage's summary/completion view.

### Persistence (per calendar day, per browser)

`localStorage` keys are namespaced by `todayKey()` so progress resets naturally each day:
- `hpdle_progress_<date>` — stage 1 (`guesses`, `won`, `statsRecorded`).
- `hpdle_stage2_<date>` — stage 2 (`step`, `results`, `currentAnswered`, `currentChosenName`, `finished`, `statsRecorded`).
- `hpdle_stage3_<date>` — stage 3 (`speakerChosen`, `speakerCorrect`, `speakerAnswered`, `movieChosen`, `movieCorrect`, `movieAnswered`, `finished`, `statsRecorded`).

Old dated keys are never cleaned up (harmless, grows slowly over months).

Restore chains linearly: `restoreProgress()` (stage 1) → `restoreStage2Progress()` (only reached if stage 1 was won) → `restoreStage3Progress()` (called from inside `renderStage2Summary()`, so it runs both on a fresh Stage 2 finish and on every reload once Stage 2 is done). `restoreStage3Progress()` either reveals the `stage3-start-btn` (no saved Stage 3 data yet), resumes mid-quote (speaker answered but not movie), or re-renders the finished summary — mirroring `restoreStage2Progress()`'s three-way branch exactly.

### Firestore (shared aggregate stats only — no auth, no per-user data)

Single collection `dailyStats`, one doc per day (`doc(db, 'dailyStats', todayKey())`), written via `setDoc(..., { merge: true })` with `increment()`. Fields: `solvedCount`/`totalGuesses` (stage 1, written by `recordWinStats`), `stage2FinishedCount`/`stage2CorrectAnswers`/`stage2TotalAnswers` (stage 2, written by `finishStage2`), and `stage3FinishedCount`/`stage3CorrectAnswers`/`stage3TotalAnswers` (stage 3, written by `finishStage3` — `stage3CorrectAnswers` is 0–2 per finish, `stage3TotalAnswers` is always +2, since each finish is exactly 2 sub-answers: speaker + movie). Each write is guarded by a `statsRecorded`/`stage2StatsRecorded`/`stage3StatsRecorded` flag persisted in `localStorage` so a solve/finish is only ever counted once per browser per day, and retried on next load if the write failed.

**Firestore security rules are not in this repo** — they're set directly in the Firebase console (Firestore Database → Rules) and must be manually kept in sync with the fields the client writes. They use a strict `request.resource.data.keys().hasOnly([...])` whitelist on the *entire resulting document*, validated per-field-group via `diff(resource.data).affectedKeys()` (so a stage-1-only write isn't forced to satisfy stage-2/stage-3 field constraints and vice versa). **Adding any new field written to `dailyStats` requires updating these console-side rules in the same change, or the write will silently fail client-side** (caught and logged to console, but otherwise invisible — this has already caused a real bug once). **Confirmed still outstanding: the `stage3FinishedCount`/`stage3CorrectAnswers`/`stage3TotalAnswers` fields have NOT been added to the console-side rules** — a live test against the production project reproduced `finishStage3`'s write throwing `FirebaseError: Missing or insufficient permissions` (`permission-denied`) every time. The client degrades correctly (catches it, logs it, leaves the Quote Challenge stats line blank — no crash), but **do this console update before relying on Stage 3 stats in production.** Firebase project config (`firebaseConfig` in the script) is not secret; it's gated entirely by these rules.

**Concurrent get+set on the same `dailyStats` doc can return a stale/partial read — always re-read after your own write resolves.** Each `finishStageN()` fires its stage's own summary render (`renderStageNSummary()`) *before* awaiting its own `setDoc`. That render kicks off `getDoc()` calls for every stats line shown on that screen, including ones for *other* stages reading the same physical doc (e.g. `displayStage1StatsForSummary()` inside `renderStage2Summary()`). Those reads start concurrently with the pending `setDoc` for the current stage and can resolve with a snapshot that's missing fields from other, already-committed writes (confirmed live: a `getDoc()` fired this way returned `{stage2FinishedCount, stage2CorrectAnswers, stage2TotalAnswers}` with `solvedCount`/`totalGuesses` entirely absent, even though they'd already been written and displayed successfully on the Stage 1 win banner moments earlier in the same page session) — and since those two fields never get a second read, the "X people have solved..." line stayed blank forever until a full page reload re-ran everything from scratch. The fix, applied in both `finishStage2()` and `finishStage3()`: **re-call every stats-display function for that screen again after your own `setDoc` resolves** (not just the one for the stage you're finishing) — see the retry calls right after `saveStage2Progress()`/`saveStage3Progress()` in each function. If you add a stage 4, give `finishStage4()` the same treatment: call all of that screen's `displayStageNStats*()` functions once eagerly (for a snappy initial paint) and once more after its own write settles.

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

### Quote data pipeline (`hp_quotes.csv`)

**`hp_quotes.csv` is a source file, not loaded at runtime** — same pipeline convention as `hp_spells.csv`: parsed by hand into the inline `QUOTES` array in `index.html`. **Editing the CSV does nothing to the live game until someone re-parses it and replaces the `QUOTES` array block.**

Columns: `Quote` → `quote`, `Speaker` → `speaker`, `Movie name` → `movie`, `Situation` → `situation` (shown as bonus flavor text after the movie sub-answer is revealed, not used for scoring).

**Curation requirement:** every `speaker` value must exactly match a `CHARACTERS[].name` string, since Stage 3's speaker guess is checked against this list and rendered via the same autocomplete pool. When re-syncing from a fresh CSV export, re-validate every speaker against `CHARACTERS` before merging — three issues were found and fixed in the original 96-row export:
- **"Trolley Witch"** — not a playable character and not worth adding → the quote was **dropped entirely** (95 quotes remain).
- **"Lord Voldemort / Quirrell"** (dual-credited line, Quirrell's body / Voldemort's will) → normalized to **"Lord Voldemort"**.
- **"Hagrid"** (missing "Rubeus", inconsistent with every other Hagrid line in the set) → normalized to **"Rubeus Hagrid"**.

One quote ("Magic is might.") was kept by adding its speaker, **Corban Yaxley**, to `CHARACTERS` instead of dropping the line (see the Data provenance note above) — prefer adding a missing-but-valid character over dropping a good quote, unless the character is clearly not a real guessable entity (like the Trolley Witch).

### `MOVIES`

A fixed 8-entry array of the canonical film titles exactly as they appear in `hp_quotes.csv`'s `Movie name` column (`"Harry Potter and the Philosopher's Stone"` through `"Harry Potter and the Deathly Hallows: Part 2"`). Used both as the Stage 3 movie-guess autocomplete pool and for exact-match scoring against `QUOTES[].movie`.

## Data files

- `index.html` — the entire app.
- `hp_spells.csv` — spell reference data; source-of-truth for `SPELLS`, but must be manually re-synced into `index.html` after edits (see above).
- `hp_quotes.csv` — quote reference data; source-of-truth for `QUOTES`, but must be manually re-synced into `index.html` after edits (see above).
- `HPfont.TTF` — custom display font, used only for the `<h1>` title via `@font-face`.
