# Spec: morse-decoder

## Overview

Dot-Dash lets people who cannot speak and can barely move type with one switch in Morse code; the browser then speaks the message. This spec covers the **core decoder**: the one-switch input capture, the symbol and segmentation logic, the International Morse code table, and — most importantly — the **adaptive timing engine** that learns each user's own dot, dash, and pause thresholds and adjusts continuously so slow, uneven, or fatiguing presses are not misread.

This is the foundation everything else builds on. It must be correct and pure before speech, prediction, or the phrase board are layered on.

It follows:
- `product.md` — purpose, users, must-have one-switch input and adaptive timing.
- `accessibility.md` — single-action input, undo, live region, contrast, reduced-motion.
- `morse-timing.md` — standard ratios, character set, and the adaptive timing rule.

## Scope

In scope:
- One-switch input capture: press/release timestamps from space bar, on-screen button, or mouse/touch click, unified into a single event stream.
- Symbol classification: dot vs dash from press duration against an adaptive threshold.
- Segmentation: within-letter gap, between-letter gap (≈3 units), between-word gap (≈7 units) from gap duration against adaptive thresholds.
- International Morse code table: the full character set defined in `morse-timing.md`.
- Adaptive timing engine: learns the user's own dot/dash and gap thresholds, adjusts continuously, disambiguates toward the user's most common pattern.
- Decoder orchestrator: pure function/class that takes timestamped press/release data in and returns decoded text and incremental events out.
- Undo: undo last letter and undo last word, each in one action.
- Observability: an incremental decoded-text stream the UI can render and announce via a live region.
- Serialization of the learned timing profile so it can be saved/restored.

Out of scope (separate specs/work):
- Speech output via the Web Speech API (`src/speech/`).
- Quick-phrase board.
- Word prediction and the built-in word list (`src/words/`).
- Urdu Morse input and Urdu speech.
- Practice mode.
- Carer view.
- Two-switch mode (one switch for dots, one for dashes).
- Service worker / offline caching (`public/`).
- Telegraph visual aesthetic (CSS, layout, imagery).
- The `localStorage` read/write module itself (`src/storage/`). This spec makes the profile serializable; the storage wiring is separate.

## Background

Standard Morse timing is defined in relative units: dot = 1, dash = 3, gap inside a letter = 1, gap between letters = 3, gap between words = 7 (see `morse-timing.md`). A fixed unit length is useless for AAC users — their presses are slow, uneven, and change with fatigue. The decoder must learn each person's own timing in their own units and keep adjusting. When a press falls in an ambiguous zone, it must lean toward the user's most common pattern rather than toward an error.

The decoder lives in `src/morse/` and must stay pure: no DOM, no UI framework, no side effects, no `localStorage`. It takes timing data in and returns decoded text out. The UI layer calls it; it never calls the UI.

**Resolved decisions** (carried into the requirements below): the decoder starts at roughly 5 words per minute (one unit ≈ 240 ms) with thresholds placed halfway between the standard ratios; it learns dots and dashes as two separate weighted moving averages with the classification threshold between them; the profile is saved on this device only via `localStorage` with a "reset my timing" control; an unknown sequence shows a white square (Unicode U+25A1, □) that is never spoken and can be undone, never a question mark; an optional, skippable calibration of five dots and five dashes seeds thresholds faster; learning blends defaults into observed data gradually and is fully learned after at least five dots and five dashes; any pause longer than the word gap yields exactly one space and is excluded from learning.

## Requirements

### 1. One-switch input capture
As a user with very limited mobility, I want to type using a single switch — the space bar, a large on-screen button, or a mouse/touch click — so I do not need two hands or simultaneous inputs.

Acceptance criteria:
1.1 WHEN a space-bar `keydown`/`keyup`, an on-screen Morse button `pointerdown`/`pointerup`, or a mouse/touch click occurs, THE decoder SHALL consume all three through a single unified press/release event stream.
1.2 WHEN any input source is used, THE decoder SHALL require only a single action; no simultaneous-key combination shall ever be required.
1.3 WHEN on-screen and clickable targets are rendered, THE system SHALL make each at least 44 × 44 px.
1.4 WHEN an interactive control receives focus, THE system SHALL display a visible focus indicator and SHALL expose an `aria-label` or visible text.
1.5 WHILE the switch is held, THE decoder SHALL interpret it as one continuous press until release; WHEN key-repeat events fire from a held key, THE decoder SHALL ignore them.

### 2. Symbol classification (dot vs dash)
As a user whose press lengths vary, I want each press classified as a dot or dash based on my own learned timing, not a fixed constant, so slow or uneven presses are not misread.

Acceptance criteria:
2.1 WHEN a press is released, THE decoder SHALL classify it as a dot or dash from its duration relative to the adaptive dot/dash threshold.
2.2 THE decoder SHALL never classify a press using a hard-coded fixed unit length.
2.3 WHEN a press duration falls in the ambiguous zone between the learned dot average and the learned dash average, THE decoder SHALL classify it toward the user's most common symbol rather than toward an error or an empty result.

### 3. Letter segmentation
As a user, I want a pause to end the current letter so each dot/dash run becomes the right character.

Acceptance criteria:
3.1 WHEN the gap between two presses is shorter than the adaptive between-letter threshold (within-letter, ≈1 unit in the user's scale), THE decoder SHALL treat it as part of the same letter and SHALL NOT produce a letter boundary.
3.2 WHEN the gap reaches the adaptive between-letter threshold (≈3 units), THE decoder SHALL end the current letter and start a new one.
3.3 WHILE the user continues typing, THE decoder SHALL learn the between-letter threshold from the user's own observed gaps so it adapts continuously.

### 4. Word segmentation
As a user, I want a longer pause to end the current word and insert a space so words separate correctly.

Acceptance criteria:
4.1 WHEN the gap reaches the adaptive between-word threshold (≈7 units), THE decoder SHALL end the current word and insert a single space.
4.2 THE decoder SHALL keep the between-word threshold adaptive and distinct from the between-letter threshold.
4.3 WHEN a word boundary is produced, THE decoder SHALL represent it as a space character in the decoded text, never by colour or sound alone.
4.4 WHEN a pause is longer than the between-word threshold, THE decoder SHALL emit exactly one space and SHALL NOT use that gap duration for learning thresholds.

### 5. International Morse decoding
As a user, I want my dot/dash sequences decoded into the full International Morse character set so I can write ordinary messages.

Acceptance criteria:
5.1 WHEN a dot/dash sequence matches a letter A–Z, THE decoder SHALL decode it to that letter.
5.2 WHEN a dot/dash sequence matches a digit 0–9, THE decoder SHALL decode it to that digit.
5.3 WHEN a dot/dash sequence matches a punctuation mark from `morse-timing.md`, THE decoder SHALL decode it to that mark: period `.`, comma `,`, question `?`, apostrophe `'`, exclamation `!`, slash `/`, parentheses `(` `)`, ampersand `&`, colon `:`, semicolon `;`, equals `=`, plus `+`, minus/hyphen `-`, underscore `_`, quotation `"`, at `@`.
5.4 WHEN a dot/dash sequence does not match any known character, THE decoder SHALL emit a white square symbol (Unicode U+25A1, □) as the placeholder.
5.5 THE decoder SHALL mark the placeholder so it is never spoken by the speech layer.
5.6 THE decoder SHALL allow the placeholder to be undone like any other character.
5.7 THE decoder SHALL never use a question mark as the placeholder for an unknown sequence.

### 6. Adaptive timing engine
As a user whose timing drifts with fatigue, I want the decoder to learn my own dot, dash, and pause thresholds and keep adjusting, so it keeps reading me correctly over time.

Acceptance criteria:
6.1 THE decoder SHALL learn the user's own dot duration, dash duration, within-letter gap, between-letter gap, and between-word gap.
6.2 THE decoder SHALL derive thresholds from observed data and SHALL never assume a fixed unit length.
6.3 WHEN new presses are observed, THE decoder SHALL update thresholds continuously.
6.4 THE decoder SHALL tolerate slow and uneven presses without misreading.
6.5 THE decoder SHALL maintain two separate weighted moving averages — one for dots and one for dashes — that weight recent presses more than older ones so fatigue-driven drift is followed, and SHALL place the dot/dash classification threshold between the two averages.
6.6 THE decoder SHALL expose the full learned timing profile as serializable data (dot/dash threshold, letter-gap threshold, word-gap threshold, and a confidence/samples count) and SHALL restore from such data.
6.7 THE system SHALL save the learned profile on this device only, through `localStorage`, and SHALL provide a "reset my timing" control that clears the stored profile and returns to defaults.

### 7. Bootstrap before enough data
As a first-time user, I want the decoder to work from my very first press using sensible defaults, then improve as it learns me.

Acceptance criteria:
7.1 WHEN no learned data exists, THE decoder SHALL function from the first press using default thresholds starting at approximately 5 words per minute (one unit ≈ 240 ms), with the dot/dash threshold halfway between the dot (1) and dash (3) ratios, the letter-gap threshold halfway between the within-letter (1) and between-letter (3) ratios, and the word-gap threshold halfway between the between-letter (3) and between-word (7) ratios.
7.2 THE decoder SHALL keep default thresholds clearly separated enough to classify early presses without trivial misreads.
7.3 WHEN each new press is observed, THE decoder SHALL move gradually from the default thresholds toward the learned thresholds, and SHALL consider itself fully learned after at least five dots and five dashes have been observed.
7.4 THE system SHALL offer an optional, skippable calibration pass of five dots and five dashes to seed thresholds faster; WHEN the user skips calibration, THE decoder SHALL still learn online from normal typing.

### 8. Undo
As a user who makes mistakes, I want to undo my last letter and my last word, each with one action.

Acceptance criteria:
8.1 WHEN the user enters the Morse error prosign (eight dots, `........`), THE decoder SHALL delete the last emitted letter, so a one-switch user can undo with the switch alone.
8.2 WHEN the user presses Backspace, THE decoder SHALL delete the last emitted letter, for keyboard users and carers.
8.3 WHEN the user activates the on-screen undo button, THE decoder SHALL delete the last emitted letter, for carers and pointer users.
8.4 WHEN the user requests undo of the last word, THE decoder SHALL remove all letters back to and including the last space in a single action.
8.5 WHILE a letter is partly typed but not yet committed, THE decoder SHALL undo the last dot or dash in the current letter when undo is requested; WHEN a letter is already committed, THE decoder SHALL undo that committed letter.
8.6 THE decoder SHALL never require a second simultaneous input for any undo action.

### 9. Decoded-text stream for rendering and live region
As a screen-reader user, I want decoded text announced as it forms so I can follow what I am typing.

Acceptance criteria:
9.1 WHEN the decoder processes an event, THE decoder SHALL expose the current decoded text plus incremental updates (letter appended, letter committed, word completed, space inserted).
9.2 THE decoder SHALL provide data sufficient for the UI to render the buffer and to feed an `aria-live` region. (UI-side `aria-live` wiring is out of scope for this spec; the decoder only provides the stream.)

### 10. Tremor and switch bounce rejection
As a user with tremor or an unreliable switch, I want very short accidental presses ignored so stray bumps do not become dots or dashes.

Acceptance criteria:
10.1 WHEN a press duration is shorter than the adaptive minimum press duration, THE decoder SHALL ignore it and SHALL NOT classify it as a dot or dash.
10.2 THE decoder SHALL learn the minimum press duration from the user's own press patterns so it adapts to their tremor and switch characteristics.

### 11. Minimum pause floors
As a user who pauses to think or rest, I want to set slower letter and word pauses so the decoder does not misread my thinking time as letter or word boundaries.

Acceptance criteria:
11.1 WHEN the user or a carer sets a minimum letter-pause floor and a minimum word-pause floor, THE decoder SHALL treat those floors as the lower bound for the respective gap thresholds.
11.2 THE adaptive engine SHALL never lower a gap threshold below its floor, so a user who stops to think is not punished with spurious letter or word boundaries.

## Non-functional requirements

- **Purity:** `src/morse/` SHALL have no DOM access, no UI framework imports, no side effects, and no `localStorage` access; it SHALL be callable from tests and from the UI with no browser environment required.
- **Offline:** THE decoder SHALL make no runtime network calls and SHALL run with no connection.
- **Real-time latency:** WHEN an event is processed, THE decoder SHALL complete classification, segmentation, and decoding well within a frame so input feels instantaneous under typical typing speeds.
- **Accessibility (WCAG 2.2 AA):**
  - THE system SHALL make every action achievable with one key or one switch; no simultaneous input shall be required.
  - THE system SHALL make touch/click targets at least 44 × 44 px.
  - THE system SHALL show a visible focus indicator on all interactive elements.
  - THE system SHALL keep text contrast at least 4.5:1; UI components, focus indicators, and the large Morse button SHALL be at least 3:1 against adjacent areas (WCAG 1.4.11).
  - THE system SHALL never convey information by colour alone; dots, dashes, and states SHALL also differ by shape or text (WCAG 1.4.1).
  - THE system SHALL provide screen-reader labels on all controls.
  - THE system SHALL respect `prefers-reduced-motion` with no non-essential animation.
  - THE system SHALL impose no time limit that punishes slow users; the adaptive timer SHALL adjust to the user and SHALL never force a fixed speed; the user or carer MAY set minimum pause floors (Requirement 11) that the engine never goes below.
  - THE system SHALL always provide undo of the last letter and the last word.
  - WHEN an unknown sequence is decoded, THE system SHALL show a white square (U+25A1, □) that is never spoken and can be undone, never a question mark.
- **Standard timing reference:** THE decoder SHALL follow the 1:3:3:7 dot:dash:letter-gap:word-gap ratio from `morse-timing.md` as the reference model, adapted to the user's own unit scale.
- **Persistence:** THE system SHALL save the learned profile on this device only via `localStorage` and SHALL provide a "reset my timing" control; the decoder exposes the profile as serializable data (Requirement 6) while the storage module itself is out of scope.
- **Tech conformance:** THE system SHALL be built in TypeScript with Vite; tests SHALL use Vitest unit tests and fast-check property-based tests under `tests/` mirroring `src/` paths; files SHALL be kebab-case, types PascalCase, functions camelCase per `structure.md`.
- **Language:** THE system SHALL keep all code comments, this spec, and UI text in English (Urdu mode is a separate, optional should-have).
