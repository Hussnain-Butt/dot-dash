# Implementation Plan

Tasks are written in build order. Each task is small enough to finish and verify on
its own, and the unit tests for a module live in the same task as that module.
Property-based tests (P1–P12) are a dedicated task because they exercise the whole
pure stack together. The only file outside `src/morse/` defined by this spec is the
DOM-touching input adapter in `src/ui/`; everything in `src/morse/` stays pure (no
DOM, no UI framework, no `localStorage`, no network) per `structure.md`.

## 1. Project setup — Vite, TypeScript, Vitest, fast-check

- [ ] Initialize the Vite + TypeScript project in the workspace root: `package.json`,
  `index.html`, `tsconfig.json`, `vite.config.ts`, and a minimal `src/main.ts`
  placeholder so `vite build` runs.
- [ ] Add dev dependencies: `vitest`, `fast-check`, and `jsdom`; configure Vitest so
  `tests/morse/**` runs in the default node environment (no DOM) and `tests/ui/**`
  runs in the jsdom environment.
- [ ] Create the directory skeleton per `structure.md`: `src/morse/`, `src/ui/`,
  `tests/morse/`, `tests/ui/`.
- [ ] Add an `npm run build` script (`vite build`) and an `npm test` script
  (`vitest --run`); add a placeholder `tests/morse/smoke.test.ts` asserting the
  toolchain works.
- [ ] Verify `npm run build` and `npm test` both pass from a clean checkout.

Requirements covered: NFR (Tech conformance — TypeScript/Vite/Vitest/fast-check,
file naming, tests mirroring `src/` paths), `structure.md` purity and layout, Req 7.1
(default-unit bootstrap context).

## 2. Morse table — `src/morse/morse-table.ts`

- [ ] Define `Symbol = 'dot' | 'dash'`, `UNKNOWN_PLACEHOLDER = '\u25A1'` (□, marked
  non-spoken), and `ERASE_PROSIGN = '........'` (eight dots, a control prosign, not a
  printable table entry).
- [ ] Add the full International Morse lookup (dot/dash string → character): letters
  A–Z, digits 0–9, and the punctuation set from `morse-timing.md` (`. , ? ' ! / ( ) & : ;
  = + - _ " @`).
- [ ] Add a pure `decodeSequence(seq: string): string | null` returning the char or
  `null` for an unknown sequence (the caller maps `null` → `□`), and an
  `encodeChar(ch: string): string | null` reverse lookup.
- [ ] Keep the module pure data + pure functions: no imports, no side effects.
- [ ] Write `tests/morse/morse-table.test.ts`: every entry round-trips
  (`decodeSequence(encodeChar(c)) === c`); unknown codes are not in the table;
  `........` is excluded from printable entries (handled as a control, not a char).

Requirements covered: Req 5.1, 5.2, 5.3, 5.4, 5.7.

## 3. Serializable profile — `src/morse/profile.ts`

- [ ] Define the `TimingProfile` interface per design §2/§7: `version: 1`, the four
  learned EWMA averages (`dotAvg`, `dashAvg`, `withinLetterGapAvg`,
  `betweenLetterGapAvg`), the four sample counts, `minPressFraction`,
  `letterGapFloor`, `wordGapFloor`.
- [ ] Implement `defaultProfile()` returning all defaults with zero samples
  (dot=240 ms, dash=720 ms, within-letter gap=240 ms, between-letter gap=720 ms,
  `minPressFraction=0.25`, floors=0).
- [ ] Implement `serializeProfile(p): string` (`JSON.stringify`) and
  `deserializeProfile(json): TimingProfile | null` returning `null` for bad JSON,
  unknown version, or NaN/negative numeric fields (replacing invalid numbers with
  defaults where the shape is otherwise valid).
- [ ] Write `tests/morse/profile.test.ts`: `deserializeProfile(serializeProfile(p))`
  deep-equals valid generated profiles; bad JSON, wrong version, and NaN/negative
  fields yield `null`; `defaultProfile()` matches the §4.1 default values exactly.

Requirements covered: Req 6.6, 6.7 (reset to defaults), 7.1.

## 4. Adaptive timing engine — `src/morse/adaptive-timing.ts`

- [ ] Define the named constants from design §4.2: `UNIT_MS_DEFAULT=240`,
  `DEFAULT_DOT`, `DEFAULT_DASH`, `DEFAULT_WITHIN_LETTER_GAP`,
  `DEFAULT_BETWEEN_LETTER_GAP`, `DOT_DASH_EWMA_ALPHA=0.3`, `GAP_EWMA_ALPHA=0.3`,
  `CALIBRATION_TARGET=5`, `MIN_PRESS_FRACTION=0.25`, `MIN_PRESS_FLOOR_MS=40`.
- [ ] Implement the `AdaptiveTiming` state (learned EWMAs, sample counts, floors)
  plus pure update functions `recordDot`, `recordDash`, `recordWithinLetterGap`,
  `recordBetweenLetterGap` using `avg ← α·sample + (1−α)·avg` (first sample seeds
  the average).
- [ ] Implement the default-blend fractions
  `f = clamp(min(dotSamples, dashSamples)/5, 0, 1)` and
  `g = clamp(min(withinLetterGapSamples, betweenLetterGapSamples)/5, 0, 1)`, and the
  effective averages `effDotAvg`, `effDashAvg`, `effWithinLetterGap`,
  `effBetweenLetterGap` that blend from defaults to learned values.
- [ ] Derive thresholds: `dotDashThreshold = (effDotAvg + effDashAvg)/2`;
  `letterGapThreshold = max(letterGapFloor, (effWithinLetterGap + effBetweenLetterGap)/2)`;
  `wordGapThreshold = max(wordGapFloor, (5/3)·effBetweenLetterGap)`.
- [ ] Implement `classifyPress(duration): 'dot' | 'dash' | 'ignored'` with the
  narrow ambiguous band (`zoneHalfWidth = 0.15·span`, `span = |effDashAvg − effDotAvg|`):
  below the band → dot, above → dash, inside → majority bias (dotCount vs dashCount,
  tie → threshold side); and `minPressDuration = max(MIN_PRESS_FLOOR_MS,
  MIN_PRESS_FRACTION·effDotAvg)` so sub-threshold presses are `'ignored'`.
- [ ] Add pure getters for the derived thresholds, `minPressDuration`, and a
  `TimingProfile` snapshot/restore (no `localStorage` — only serializable data).
- [ ] Write `tests/morse/adaptive-timing.test.ts`: defaults reproduce the §4.1
  threshold table exactly (dot/dash=480, letter=480, word=1200); EWMA updates match
  the formula by hand for a few steps; `f`/`g` reach 1 at 5/5 samples; pause floors
  clamp thresholds and never lower below the floor; the tremor filter ignores
  sub-`minPressDuration` presses; between-word gaps are excluded from gap learning.

Requirements covered: Req 2.1, 2.2, 2.3, 3.3, 4.2, 4.4, 6.1, 6.2, 6.3, 6.4, 6.5, 6.6,
7.1, 7.2, 7.3, 10.1, 10.2, 11.1, 11.2.

## 5. Decoder orchestrator — `src/morse/morse-decoder.ts` (+ `index.ts`)

- [ ] Define `DecoderState` (text, currentLetter, pressed, lastReleaseTs,
  lastPressTs, dotCount, dashCount, consecutiveErase, timing) and the `DecoderEvent`
  union (`symbol`, `letter`, `space`, `erase_symbol`, `erase_letter`, `erase_word`,
  `ignored`) per design §2.
- [ ] Implement `MorseDecoder` holding the state, constructed from an optional
  `TimingProfile`, exposing `press(ts)`, `release(ts)`, `tick(ts)`, `flush()`,
  `undoSymbol()`, `getText()`, `getProfile()`, each returning `DecoderEvent[]`.
- [ ] `press(ts)`: measure `gap = ts − lastReleaseTs` (first press has no gap);
  within-letter (`gap < letterGapThreshold`) continues the current letter and learns
  the within-letter gap; between-letter (`letterGapThreshold ≤ gap < wordGapThreshold`)
  commits the current letter and learns the between-letter gap; between-word
  (`gap ≥ wordGapThreshold`) commits the letter, inserts exactly one space, and learns
  nothing (Req 4.4); set `pressed=true`, `lastPressTs=ts`.
- [ ] `release(ts)`: `duration = ts − lastPressTs`; if below `minPressDuration` emit
  `{ignored}` and learn nothing; otherwise classify dot/dash, update the dot/dash
  EWMA + counts, push the symbol into `currentLetter`, emit `{symbol}`.
- [ ] `commitCurrentLetter()` with the **empty-letter guard (review rule)**: if
  `currentLetter` is empty, do nothing and emit nothing — committing an empty current
  letter never produces a `□`. Consequently a `tick` that fires after a letter was
  already committed is a no-op, and a press that follows a tick commit never adds a
  white square.
- [ ] On commit of a non-empty letter: if the sequence equals `ERASE_PROSIGN`
  (`........`) apply the erase prosign — first occurrence → `undoLetter()` +
  `{erase_letter}` and set `consecutiveErase=true`; a second `........` in a row →
  `undoWord()` (strip trailing spaces, then remove the last word back to and including
  the space before it) + `{erase_word}`, and reset `consecutiveErase=false`; otherwise
  decode via the table, appending the char or `□` (non-spoken) to `text`, resetting
  `consecutiveErase=false`; always clear `currentLetter` after a real commit.
- [ ] `tick(ts)`: if pressed, or `lastReleaseTs == null`, or `currentLetter` is empty →
  no-op; else if `gap ≥ letterGapThreshold` commit the current letter (no space) and
  emit `{letter}` so the letter appears promptly within one letter-threshold of its
  release.
- [ ] `flush()`: commit `currentLetter` if non-empty (no trailing space) emitting
  `{letter}`; otherwise nothing.
- [ ] Implement `undoSymbol()` (pop the last dot/dash from a non-empty partial letter,
  else remove the last committed char), `undoLetter()`, and `undoWord()` with the
  edge cases from design §6 (no-op on empty text+letter; clears all text when there is
  no space). Never require a second simultaneous input.
- [ ] Add totality/error guards: stray release/press ignored, non-monotonic or
  negative timestamps skipped for learning, zero-length press ignored as tremor;
  no input sequence ever throws or leaves invalid state.
- [ ] Add `src/morse/index.ts` re-exporting the public surface (`MorseDecoder`,
  `TimingProfile`, table constants, profile helpers, event/symbol types).
- [ ] Write `tests/morse/morse-decoder.test.ts`: a press→release→tick→flush round-trip
  decodes a word; `tick` commits a partial letter promptly; the empty-letter commit
  guard holds (no `□` after a tick commit; a press following a tick commit never adds
  a white square); `undoSymbol` pops partial-then-committed; a single `........`
  deletes one letter; two in a row delete a word; edge cases (undo past empty, no
  space clears all); a stray release is ignored; an unknown sequence yields `□` and
  never `?`.

Requirements covered: Req 1.5, 2.1, 2.3, 3.1, 3.2, 3.3, 4.1, 4.2, 4.3, 4.4, 5.1, 5.4,
5.5, 5.6, 5.7, 6.3, 6.6, 7.1, 8.1, 8.2, 8.4, 8.5, 8.6, 9.1, 9.2, 10.1, 11.1, 11.2,
NFRs (purity, totality/error handling, real-time latency).

## 6. Property-based tests P1–P12 — `tests/morse/properties.test.ts`

- [ ] Add a fast-check helper that encodes a supported text into an ideal-timing
  press/release stream (dot=1u, dash=3u, within-letter gap=1u, between-letter gap=3u,
  between-word gap=7u) using the decoder's learned unit, plus a calibration runner
  that feeds 5 dots then 5 dashes.
- [ ] P1 — Encode/decode round-trip: for any text from the supported character set,
  `decode(encode(text)) === text` (Req 5).
- [ ] P2 — Scale invariance once learned: after calibration, scaling all durations by
  `k` (bounded so every scaled press stays above the 40 ms tremor floor, with no pause
  floors set) leaves the decoded text unchanged (Req 6.4, 6.5).
- [ ] P3 — Never crashes on random presses: arbitrary/non-monotonic/negative
  timestamps never throw and `getText()` is always a string (error handling).
- [ ] P4 — Dot/dash classification consistency: below the band → dot, above → dash,
  inside → majority bias (tie → threshold side); outside the band classification is
  solely by threshold side (Req 2).
- [ ] P5 — Segmentation correctness: gap < letter threshold never ends a letter;
  `[letter, word)` ends exactly one letter; `≥ word` inserts exactly one space and is
  excluded from gap learning (Req 3, 4.4).
- [ ] P6 — Tremor filter: any press below `minPressDuration` emits `{ignored}` and
  updates no average (Req 10).
- [ ] P7 — Pause floors bound thresholds: after setting floors, the effective letter
  and word thresholds are always ≥ their floors (Req 11.2).
- [ ] P8 — Undo invariants: `undoSymbol` shrinks a non-empty partial letter, else
  removes a committed char; one `........` removes exactly one letter; two remove the
  last word; text length never goes negative (Req 8).
- [ ] P9 — Placeholder behaviour: an unsupported sequence yields exactly `□`, marked
  non-spoken and undoable; the decoder never emits `?` (Req 5.4–5.7).
- [ ] P10 — Blend monotonicity: as dot/dash samples grow 0→5 each, `f` rises 0→1 and
  effective averages move from defaults toward the learned EWMAs; at 5/5 the effective
  averages equal the learned EWMAs (Req 7.3).
- [ ] P11 — No fixed unit after learning: after learning from a user whose unit ≠
  240 ms, the classification thresholds differ from the hard-coded defaults (Req 2.2,
  6.2).
- [ ] P12 — Profile round-trip: `deserializeProfile(serializeProfile(p))` deep-equals
  `p` for every valid generated profile; corrupted/unknown-version data returns `null`
  (Req 6.6).
- [ ] Run `vitest --run` and confirm all twelve properties pass.

Requirements covered: Req 2, 3, 4.4, 5, 6.2, 6.4, 6.5, 6.6, 7.3, 8, 10, 11.2, NFR
(totality/error handling).

## 7. Input capture adapter — `src/ui/input-capture.ts`

- [ ] Create `src/ui/input-capture.ts` as the only DOM-touching adapter: listen to
  space-bar `keydown`/`keyup` (ignoring `repeat` events from a held key, per 1.5),
  `pointerdown`/`pointerup` on the on-screen Morse button (≥44×44 px, visible focus
  indicator, `aria-label`), and mouse/touch clicks, unifying all three into a single
  `{type:'press'|'release', timestamp}` stream via `performance.now()`.
- [ ] Forward the stream to `MorseDecoder.press(ts)`/`release(ts)`; schedule a short
  `tick(ts)` interval timer so the current letter commits promptly at the letter
  threshold; run an idle-flush timer that calls `decoder.flush()` as a fallback before
  the speak action.
- [ ] Wire Backspace and the on-screen undo button to `decoder.undoSymbol()`. Perform
  no decoding, classification, or timing math here — that all lives in `src/morse/`.
- [ ] Enforce the purity boundary: confirm no `src/morse/` file imports from `src/ui/`
  or touches the DOM.
- [ ] Write `tests/ui/input-capture.test.ts` (jsdom): space keydown/keyup produce a
  press then a release; key-repeat events on a held key are ignored; pointer and click
  events map to the same stream; the adapter forwards to a mock decoder and never
  decodes itself; the button meets the 44×44 px, focus-indicator, and `aria-label`
  contract.

Requirements covered: Req 1.1, 1.2, 1.3, 1.4, 1.5, 8.2, 8.3, 8.6, NFR accessibility
(targets, focus, labels), `structure.md` purity boundary.

## Notes

- The review rule in Task 5 (committing an empty current letter does nothing) is the
  one addition beyond the design's §3 event flow; it prevents a `tick`-driven or
  press-driven commit of an empty letter from emitting a spurious `□`.
- Out of scope per `requirements.md`: speech (`src/speech/`), the phrase board, word
  prediction (`src/words/`), Urdu mode, practice mode, carer view, two-switch mode, the
  service worker (`public/`), the telegraph aesthetic, and the `localStorage` module
  (`src/storage/`). This plan only builds the pure decoder, the adaptive timing
  engine, the Morse table, undo, the serializable profile, and the input-capture
  adapter contract.
