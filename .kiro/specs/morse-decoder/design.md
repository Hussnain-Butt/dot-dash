# Spec: morse-decoder — Design

This design implements the requirements in `requirements.md`. It defines the module
boundaries, the data types, the event flow, the adaptive timing algorithm with
concrete formulas, the Morse table, the undo semantics, the serializable profile, the
error-handling posture, and the property-based testing strategy.

No code or tasks are produced by this document. It is the design against which the
implementation tasks will be written.

It follows `structure.md` (pure logic in `src/morse/`, input capture in `src/ui/`),
`morse-timing.md` (1:3:3:7 reference model and character set), `product.md`, and
`accessibility.md`.

## 1. Module map and boundaries

The decoder is split between **pure logic** in `src/morse/` and a **DOM-touching
input adapter** in `src/ui/`. Everything in `src/morse/` has no DOM access, no UI
framework imports, no `localStorage`, and no network access; it is callable from
Vitest with no browser present. Everything that touches the DOM lives in `src/ui/`.

### `src/morse/` — pure logic (no side effects)

| File | Responsibility | Purity |
|------|----------------|--------|
| `morse-table.ts` | The International Morse code table (dot/dash string → character), the `........` erase prosign, and the `□` (U+25A1) unknown-sequence placeholder marked as non-spoken. Pure data + pure lookup. | Pure |
| `adaptive-timing.ts` | The adaptive timing engine: the two dot/dash exponentially weighted moving averages (EWMAs), the gap EWMAs, the default-blend logic, threshold derivation, the minimum-press filter, and the pause floors. Exposes a pure `AdaptiveTiming` value/record plus pure update functions. | Pure |
| `morse-decoder.ts` | The `MorseDecoder` orchestrator. Takes press/release/tick events, drives `AdaptiveTiming` for classification and segmentation, consults `morse-table.ts` for decoding, applies undo, and emits the incremental event stream and current text. Holds the only mutable state and mutates only its own fields. | Pure (stateful, but no I/O) |
| `profile.ts` | The serializable `TimingProfile` type, version number, `serializeProfile` / `deserializeProfile`, and the "return to defaults" reset. Pure; no `localStorage`. | Pure |
| `index.ts` | Re-exports the public surface of `src/morse/`. | Pure |

The dependency direction is strictly inward: `morse-decoder.ts` imports
`adaptive-timing.ts`, `morse-table.ts`, and `profile.ts`; none of those import the
decoder or each other circularly. No `src/morse/` file imports from `src/ui/`,
`src/speech/`, `src/storage/`, or `src/words/`. The purity rule from `structure.md`
is enforced: `src/morse/` takes timing data in and returns decoded text and events
out; it never calls the UI.

### `src/ui/input-capture.ts` — the DOM boundary (the only adapter this spec defines)

`input-capture.ts` is the single place that listens to the DOM and unifies the three
input sources named in Requirement 1 into one press/release event stream. It owns:

- `keydown`/`keyup` on the space bar (ignoring key-repeat on a held key, per 1.5).
- `pointerdown`/`pointerup` on the large on-screen Morse button (44 × 44 px minimum,
  per 1.3; visible focus indicator and `aria-label`, per 1.4).
- Mouse/touch click on any clickable target, mapped to the same pointer events.

It converts each into `{ type: 'press' | 'release', timestamp }` using
`performance.now()` (monotonic) and forwards the stream to `MorseDecoder`. It does
**no** decoding, classification, or timing math — that all lives in `src/morse/`.
It also schedules periodic `decoder.tick(ts)` calls (a short interval timer, so the
current letter commits promptly once the gap reaches the letter threshold; see §3),
and runs the idle-flush timer that calls `decoder.flush()` as a fallback before the
speak action; both wirings are part of the adapter, not the decoder.

Everything else in `src/ui/` (rendering the buffer, the `aria-live` live region,
the undo button, the calibration prompt, the "reset my timing" control) is out of
scope for this spec; this spec only fixes the contract the adapter must satisfy and
the event stream it must produce. Speech (`src/speech/`), storage
(`src/storage/`), words (`src/words/`), and the service worker (`public/`) are out
of scope per `requirements.md`.

### Contract between the adapter and the decoder

```
src/ui/input-capture.ts                src/morse/morse-decoder.ts
─────────────────────────              ──────────────────────────
keydown/pointerdown ──► press(ts)  ──► MorseDecoder.press(ts) ──► DecoderEvent[]
keyup/pointerup    ──► release(ts)──► MorseDecoder.release(ts)──► DecoderEvent[]
tick timer         ──► tick(ts)    ──► MorseDecoder.tick(ts)    ──► DecoderEvent[]
idle timeout       ──► flush()     ──► MorseDecoder.flush()      ──► DecoderEvent[]
Backspace / button ──► undoSymbol()──► MorseDecoder.undoSymbol() ──► DecoderEvent[]
                                        MorseDecoder.getText()   ──► string
                                        MorseDecoder.getProfile()──► TimingProfile
```

## 2. Data types

All types are PascalCase, files kebab-case, functions camelCase, per `structure.md`.

### Input events (produced by `input-capture.ts`, consumed by the decoder)

```ts
type PressEvent   = { type: 'press';   timestamp: number }; // ms, monotonic
type ReleaseEvent = { type: 'release'; timestamp: number };
type InputEvent = PressEvent | ReleaseEvent;
```

`timestamp` comes from `performance.now()` in the adapter. The decoder never calls
`performance.now()` itself, keeping it deterministic and testable: tests pass
explicit timestamps.

### Symbols and characters

```ts
type Symbol = 'dot' | 'dash';
const UNKNOWN_PLACEHOLDER = '\u25A1'; // □, never spoken, undoable (Req 5.4–5.6)
const ERASE_PROSIGN = '........';      // eight dots → control, not a printed char (Req 8.1)
```

### The learned profile (serializable; see §7)

```ts
interface TimingProfile {
  version: 1;                     // profile format version
  dotAvg: number;                 // EWMA of classified dot durations (ms)
  dashAvg: number;                // EWMA of classified dash durations (ms)
  withinLetterGapAvg: number;     // EWMA of within-letter gaps (ms)
  betweenLetterGapAvg: number;    // EWMA of between-letter gaps (ms)
  dotSamples: number;
  dashSamples: number;
  withinLetterGapSamples: number;
  betweenLetterGapSamples: number;
  minPressFraction: number;      // tremor filter as a fraction of dotAvg (default 0.25)
  letterGapFloor: number;         // user/carer-set minimum letter-gap threshold (ms, default 0)
  wordGapFloor: number;           // user/carer-set minimum word-gap threshold (ms, default 0)
}
```

The thresholds themselves are **derived** at runtime from the averages above plus the
hard-coded defaults (see §5); they are not stored. A fresh/empty profile therefore
blends back to defaults automatically. The "reset my timing" control clears the
stored profile so the decoder restarts from defaults (Req 6.7).

### Decoder state (internal to `MorseDecoder`)

```ts
interface DecoderState {
  text: string;              // committed decoded text (letters, digits, spaces, □)
  currentLetter: Symbol[];  // uncommitted dots/dashes of the letter being typed
  pressed: boolean;          // switch currently held
  lastReleaseTs: number | null; // for measuring the next gap
  lastPressTs: number | null;
  dotCount: number;          // cumulative dots (for majority bias, Req 2.3)
  dashCount: number;          // cumulative dashes
  consecutiveErase: boolean;  // detects two ERASE_PROSIGN in a row (Req 8.1/8.4)
  timing: AdaptiveTiming;   // the adaptive engine state (profile + derived thresholds)
}
```

### Output events (the incremental stream the UI renders and announces; Req 9)

```ts
type DecoderEvent =
  | { type: 'symbol'; symbol: Symbol }   // dot/dash added to the current letter
  | { type: 'letter'; char: string }     // a letter committed (char may be □)
  | { type: 'space' }                    // word boundary inserted
  | { type: 'erase_symbol' }             // last dot/dash removed from partial letter
  | { type: 'erase_letter' }             // last committed letter removed
  | { type: 'erase_word' }               // letters back to/including last space removed
  | { type: 'ignored' };                 // press filtered out as tremor/bounce (Req 10)
```

The UI uses this stream to render the buffer and to feed the `aria-live` region.
The decoder only produces the stream; the `aria-live` wiring itself is out of scope
(Req 9.2). No state is ever conveyed by colour alone — symbols are `dot`/`dash`
text, the placeholder is the `□` glyph, and gaps are spaces (Req 1.4.1 / NFR).

## 3. Event flow — from timestamps to decoded text

A "press cycle" is one press followed by one release. The gap is measured between
the **previous release** and the **next press**, so segmentation (where the previous
letter ends) is decided at press time, before the new press's duration is known.

```
press(ts)   ──► measure gap = ts - lastReleaseTs (if any)
                ├─ gap < letterThreshold        ─► within-letter: continue current letter; learn within-letter gap
                ├─ letterThreshold ≤ gap < wordThreshold ─► between-letter: commit current letter (decode); learn between-letter gap
                └─ gap ≥ wordThreshold           ─► between-word: commit current letter + insert ONE space; do NOT learn (Req 4.4)
                record lastPressTs = ts; pressed = true

release(ts) ──► duration = ts - lastPressTs
                ├─ duration < minPressDuration   ─► ignored (tremor/bounce); emit {ignored}; do not classify or learn
                └─ else classify dot/dash (§5.3), update dot/dash EWMA + counts,
                    push symbol into currentLetter, emit {symbol}

flush()     ──► commit currentLetter if non-empty (no trailing space); emit {letter} or nothing
                (called by the adapter's idle timer and before "speak")

tick(ts)    ──► if pressed or lastReleaseTs == null or currentLetter is empty: no-op
                gap = ts - lastReleaseTs
                if gap ≥ letterThreshold:
                    commitCurrentLetter()        // commit the letter, NO space
                    emit {letter, char}           // the letter appears promptly
                // the space (if any) is still decided at the next press, not here

commitCurrentLetter():
                seq = currentLetter as dot/dash string
                ├─ seq === '........'  ─► ERASE_PROSIGN: apply undo (§6); emit {erase_letter} or {erase_word}; no char
                ├─ table has seq      ─► char = table[seq]; append to text; emit {letter, char}
                └─ else               ─► char = '□'; append to text; emit {letter, char:'□'} (non-spoken, undoable)
                currentLetter = []
```

The first press has no preceding gap, so no segmentation runs and no gap is learned
on the very first cycle. The current letter is committed **promptly** by the pure
`tick(ts)` function: once the gap since the last release reaches the letter
threshold, `tick` commits the partial letter without adding a space, so the letter
appears within one letter-threshold of its release instead of waiting for the next
press or the idle flush — after typing `WATER` the `R` shows promptly, not seconds
later. The space that may follow is still decided at the next press (a later gap at
or above the word threshold inserts exactly one space). The adapter in `src/ui`
schedules the `tick` calls (a short interval timer). `flush()` remains as a
fallback, called by the adapter's idle timer (no press for longer than a few times
the word-gap threshold) and immediately before the speak action, to commit any
partial letter `tick` has not yet reached. Both `tick` and `flush` commit the
partial letter without a trailing space, because no following word boundary has
been observed.

## 4. Adaptive timing algorithm

### 4.1 Reference model and defaults

The reference timing is the 1:3:3:7 ratio from `morse-timing.md` (dot:dash :
within-letter-gap : between-letter-gap : between-word-gap). Defaults start at
approximately 5 words per minute: **one unit = 240 ms** (Req 7.1).

| Quantity | Ratio (units) | Default (ms, at 240 ms/unit) |
|----------|---------------|------------------------------|
| Dot duration | 1 | 240 |
| Dash duration | 3 | 720 |
| Within-letter gap | 1 | 240 |
| Between-letter gap | 3 | 720 |
| Between-word gap | 7 | 1680 |
| **dot/dash threshold** | (1+3)/2 = 2 | **480** |
| **letter-gap threshold** | (1+3)/2 = 2 | **480** |
| **word-gap threshold** | (3+7)/2 = 5 | **1200** |

Thresholds are placed halfway between the adjacent standard ratios (Req 7.1), which
keeps early classifications clearly separated (Req 7.2). The decoder never assumes
a fixed unit length (Req 2.2, 6.2); 240 ms/unit is only the bootstrap default.

### 4.2 Constants

```ts
const UNIT_MS_DEFAULT          = 240;  // ms per unit at ~5 WPM
const DEFAULT_DOT              = 1 * UNIT_MS_DEFAULT;   // 240
const DEFAULT_DASH             = 3 * UNIT_MS_DEFAULT;   // 720
const DEFAULT_WITHIN_LETTER_GAP= 1 * UNIT_MS_DEFAULT;   // 240
const DEFAULT_BETWEEN_LETTER_GAP=3 * UNIT_MS_DEFAULT;   // 720
const DOT_DASH_EWMA_ALPHA      = 0.3;  // recent presses weighted more (Req 6.5)
const GAP_EWMA_ALPHA           = 0.3;
const CALIBRATION_TARGET       = 5;    // fully learned after ≥5 dots and ≥5 dashes (Req 7.3)
const MIN_PRESS_FRACTION       = 0.25; // tremor filter = fraction of dotAvg (Req 10)
const MIN_PRESS_FLOOR_MS       = 40;   // hard floor so a tiny learned dotAvg never starves the filter
```

`DOT_DASH_EWMA_ALPHA`, `GAP_EWMA_ALPHA`, `MIN_PRESS_FRACTION`, and
`CALIBRATION_TARGET` are named, tunable constants; they are not user-facing.

### 4.3 The two dot/dash weighted moving averages (Req 6.5)

The decoder keeps **two separate** EWMAs — one for dots, one for dashes — using the
exponentially weighted moving average, the canonical weighted moving average that
weights recent samples more than older ones:

```
On the first classified sample of a kind:  avg ← sample
On subsequent samples of that kind:         avg ← α·sample + (1 − α)·avg
```

with `α = DOT_DASH_EWMA_ALPHA = 0.3`. So `learnedDotAvg` and `learnedDashAvg` track
the user's own dot and dash durations and follow fatigue-driven drift because each
new sample carries 30 % of the weight.

The dot/dash classification threshold is placed **between the two averages**:

```
dotDashThreshold = (effDotAvg + effDashAvg) / 2
```

where `effDotAvg`/`effDashAvg` are the default-blended values from §4.4.

### 4.4 Gradual blend from defaults (Req 7.3)

Early on there is not enough data to trust the learned averages, so the effective
averages blend from the defaults toward the learned values as samples accrue:

```
f = clamp(min(dotSamples, dashSamples) / CALIBRATION_TARGET, 0, 1)   // 0 → 1

effDotAvg  = (1 − f)·DEFAULT_DOT  + f·learnedDotAvg
effDashAvg = (1 − f)·DEFAULT_DASH + f·learnedDashAvg
```

- Before any data (`f = 0`): `effDotAvg = 240`, `effDashAvg = 720`,
  `dotDashThreshold = 480` — exactly the defaults.
- After at least 5 dots **and** 5 dashes (`f = 1`): the decoder is fully learned and
  uses 100 % observed data (`effDotAvg = learnedDotAvg`, etc.), while the EWMAs keep
  tracking drift. This satisfies "fully learned after at least five dots and five
  dashes" (Req 7.3) and the optional five-dot/five-dash calibration (Req 7.4):
  calibration simply feeds 5 known dots and 5 known dashes, driving `f` to 1
  immediately; if the user skips it, normal typing reaches `f = 1` online.

### 4.5 Dot/dash classification with ambiguous-zone majority bias (Req 2.1, 2.3)

Given a press `duration` (after the tremor filter of §4.7):

```
dotDashThreshold = (effDotAvg + effDashAvg) / 2          // midpoint between the two averages
span = abs(effDashAvg - effDotAvg)                       // distance between the two averages
zoneHalfWidth = 0.15 · span                             // narrow band: ±15 % of the span
zoneLo = dotDashThreshold − zoneHalfWidth
zoneHi = dotDashThreshold + zoneHalfWidth

if duration < zoneLo          → dot                       // clearly below the band
if duration > zoneHi          → dash                      // clearly above the band
else  (duration is in the narrow ambiguous band around the threshold):
      if dotCount  > dashCount → dot      // lean toward the user's most common symbol
      if dashCount > dotCount  → dash
      else                     → (duration < dotDashThreshold) ? dot : dash   // tie → midpoint
```

The ambiguous zone is no longer the whole span between the two averages; it is a
narrow band of total width `0.30 · span` centred on the threshold. Outside that band
the press is classified by which side of the threshold it falls on, so an ordinary
dash that is slightly shorter than the dash average — which sits at
`dotDashThreshold + 0.5 · span`, well above `zoneHi = dotDashThreshold + 0.15 · span`
— is still a dash, not a dot. This keeps the P1 round-trip correct.

This satisfies Req 2.1 (classification relative to the adaptive threshold, the
midpoint) and Req 2.3 (a press in the narrow ambiguous band around the threshold is
classified toward the user's most common symbol, never toward an error or empty
result). Early on, when counts are equal, the tiebreak reduces to the midpoint
default, so behaviour degrades gracefully to the defaults. As data accrues the
averages track the user's real clusters and the majority bias only fires for
genuinely ambiguous presses near the threshold.

### 4.6 Gap segmentation thresholds (Req 3, 4)

Gaps use the same EWMA + default-blend structure:

```
g = clamp(min(withinLetterGapSamples, betweenLetterGapSamples) / CALIBRATION_TARGET, 0, 1)

effWithinLetterGap   = (1 − g)·DEFAULT_WITHIN_LETTER_GAP    + g·learnedWithinLetterGap
effBetweenLetterGap  = (1 − g)·DEFAULT_BETWEEN_LETTER_GAP   + g·learnedBetweenLetterGap

letterGapThreshold = max(letterGapFloor, (effWithinLetterGap + effBetweenLetterGap) / 2)
```

The word-gap threshold is **derived from the learned between-letter gap** using the
standard ratio, because Req 4.4 forbids learning from the long pauses that actually
create word boundaries (a 30 s thinking pause must not drag the word threshold to
30 s). Between-word is 7 units and between-letter is 3 units, so the word-gap
threshold sits at 5 units = (5/3) × (between-letter gap at 3 units):

```
wordGapThreshold = max(wordGapFloor, (5 / 3) · effBetweenLetterGap)
```

Default check: `effBetweenLetterGap = 720` → `(5/3)·720 = 1200` ✓ (matches the table
in §4.1). The word threshold stays adaptive and distinct from the letter threshold
(Req 4.2).

Gap classification and learning, applied at press time:

```
gap < letterGapThreshold                    → within-letter: continue current letter; learn withinLetterGap
letterGapThreshold ≤ gap < wordGapThreshold  → between-letter: commit current letter; learn betweenLetterGap
gap ≥ wordGapThreshold                       → between-word: commit current letter + ONE space; DO NOT learn (Req 4.4)
```

A between-word pause emits **exactly one space** and is excluded from all gap
learning (Req 4.4). The space is a real space character, never conveyed by colour or
sound alone (Req 4.3).

### 4.7 Minimum press filter — tremor / switch bounce (Req 10)

Very short accidental presses are ignored before classification:

```
minPressDuration = max(MIN_PRESS_FLOOR_MS, MIN_PRESS_FRACTION · effDotAvg)
if pressDuration < minPressDuration → emit {ignored}; do not classify; do not learn
```

`minPressDuration` adapts to the user because `effDotAvg` is learned: as the user's
dot length grows with fatigue, the bounce threshold grows proportionally, so it
tracks "their tremor and switch characteristics" (Req 10.2). The hard floor
`MIN_PRESS_FLOOR_MS = 40` prevents a tiny learned dot average from making the
filter reject real presses.

### 4.8 Pause floors (Req 11)

The user or carer may set `letterGapFloor` and `wordGapFloor` (stored in the
profile, defaults 0). The effective thresholds clamp to them:

```
letterGapThreshold = max(letterGapFloor, …)
wordGapThreshold   = max(wordGapFloor,   …)
```

The engine never lowers a gap threshold below its floor, so a user who stops to
think is not punished with spurious letter or word boundaries (Req 11.2). Setting a
floor higher than the learned threshold pins that threshold at the floor; the EWMA
keeps running underneath and the clamp reapplies on every derivation, so a later
lower floor or reset restores adaptive behaviour.

### 4.9 Calibration and reset (Req 6.7, 7.4)

- **Calibration** is an optional, skippable pass of five dots then five dashes. It
  is just a UI flow that feeds normal press/release events; because the defaults
  already classify correctly, the 5 + 5 samples drive `f` to 1 and seed
  `learnedDotAvg`/`learnedDashAvg` directly. Skipping it leaves online learning to
  reach the same state during normal typing (Req 7.4).
- **Reset** clears the stored profile (handled by the storage module, out of
  scope), which makes the decoder restart from defaults (`f = 0`,
  `learnedDotAvg = DEFAULT_DOT`, etc.).

## 5. The Morse table

`morse-table.ts` holds the full International Morse set from `morse-timing.md`.
Unknown sequences decode to `□` (U+25A1), never to `?` (Req 5.4, 5.7). The eight-dot
`........` is a control prosign, not a table entry that prints.

### Letters

| Char | Code | Char | Code | Char | Code | Char | Code | Char | Code | Char | Code |
|------|------|------|------|------|------|------|------|------|------|------|------|
| A | `.-` | B | `-...` | C | `-.-.` | D | `-..` | E | `.` | F | `..-.` |
| G | `--.` | H | `....` | I | `..` | J | `.---` | K | `-.-` | L | `.-..` |
| M | `--` | N | `-.` | O | `---` | P | `.--.` | Q | `--.-` | R | `.-.` |
| S | `...` | T | `-` | U | `..-` | V | `...-` | W | `.--` | X | `-..-` |
| Y | `-.--` | Z | `--..` | | | | | | | | |

### Digits

| 0 `-----` | 1 `.----` | 2 `..---` | 3 `...--` | 4 `....-` |
|-----------|-----------|-----------|-----------|-----------|
| 5 `.....` | 6 `-....` | 7 `--...` | 8 `---..` | 9 `----.` |

### Punctuation

| Char | Code | Char | Code |
|------|------|------|------|
| `.` period | `.-.-.-` | `,` comma | `--..--` |
| `?` question | `..--..` | `'` apostrophe | `.----.` |
| `!` exclamation | `-.-.--` | `/` slash | `-..-.` |
| `(` open paren | `-.--.` | `)` close paren | `-.--.-` |
| `&` ampersand | `.-...` | `:` colon | `---...` |
| `;` semicolon | `-.-.-.` | `=` equals | `-...-` |
| `+` plus | `.-.-.` | `-` minus/hyphen | `-....-` |
| `_` underscore | `..--.-` | `"` quotation | `.-..-.` |
| `@` at | `.--.-.` | | |

### Prosign

| Name | Code | Effect |
|------|------|--------|
| `ERASE_PROSIGN` | `........` (eight dots) | Control: undo (see §6); prints nothing; never spoken. |

`........` is unambiguous: it is longer than `H` (`....`, 4 dots) and `5`
(`.....`, 5 dots), so it never collides with a printable character.

## 6. Undo (Req 8)

The decoder exposes three undo operations; each is a single action and never
requires a second simultaneous input (Req 8.6).

- `undoSymbol()` — Backspace key (Req 8.2) and the on-screen undo button (Req
  8.3): if `currentLetter` is non-empty, pop its last dot/dash; otherwise remove
  the last committed letter from `text` (Req 8.5).
- `undoLetter()` — the eight-dot `ERASE_PROSIGN` (Req 8.1): deletes the last
  committed letter.
- `undoWord()` — requested via two consecutive `ERASE_PROSIGN`s (Req 8.4): first
  strips any trailing spaces, then removes the last word back to and including the
  space before it, in one action.

### Eight-dot prosign handling

When `currentLetter` commits as `........` (on a letter boundary or on `flush()`),
the decoder does **not** print a character. Instead:

```
on commit of '........':
  if consecutiveErase == true:           // this is the second ........ in a row
      undoWord();                        // strip trailing spaces, then remove the last word
      consecutiveErase = false           //   back to and including the space before it
      emit {erase_word}
  else:
      undoLetter();                      // remove the last committed letter
      consecutiveErase = true
      emit {erase_letter}
// committing a printable character (a table entry or □) resets consecutiveErase = false;
// the dots/dashes of a prosign are symbols held in currentLetter, not committed
// characters, so they do NOT reset it; a word-boundary space does NOT reset it either
```

So one `........` deletes the last letter (Req 8.1), and the same prosign entered
twice in a row deletes the last word (Req 8.4). Worked example for output
`HELLO WORLD`:

1. `........` → delete last letter `D` → `HELLO WORL`, `consecutiveErase = true`.
2. `........` again → `undoWord` strips any trailing spaces (none here), then
   removes `WORL` back to and including the space before it → `HELLO`.

The reset only fires when a printable character is actually committed, so the
symbols (dots) that make up the second `........` never break the chain — that is
why the second prosign reaches `undoWord()` instead of being treated as a fresh
single-letter erase. The net effect of two consecutive prosigns is that the entire
last word is gone, matching the user's mental model "one = letter, two = word".

### Edge cases

- `undoLetter()` on empty `text` and empty `currentLetter` → no-op.
- `undoWord()` when `text` contains no space → clears all of `text`.
- `undoSymbol()` with a non-empty `currentLetter` always pops a dot/dash first
  (Req 8.5); only when `currentLetter` is empty does it remove a committed letter.
- The `□` placeholder is a normal committed character and can be undone like any
  other (Req 5.6); the erase prosign is a control, not the placeholder.

## 7. Serializable profile format (Req 6.6)

The `TimingProfile` (§2) is plain JSON with a `version` number. The storage module
(`src/storage/`, out of scope) reads/writes it via `localStorage`; the decoder only
defines and validates the shape.

```ts
function serializeProfile(p: TimingProfile): string;          // JSON.stringify
function deserializeProfile(json: string): TimingProfile | null; // null on bad shape/version
function defaultProfile(): TimingProfile;                      // all defaults, samples 0
```

- **Version:** `1`. `deserializeProfile` returns `null` for any unknown version or
  malformed JSON so the caller falls back to defaults rather than crashing. (Future
  versions will migrate via an explicit upgrade step, not in scope here.)
- **Stored:** the four learned EWMA averages, the four sample counts, the tremor
  fraction, and the two pause floors.
- **Derived (not stored):** all thresholds, `eff*` blended averages, `f`/`g`
  fractions, `minPressDuration`. Recomputed from stored values + hard-coded defaults
  on every update, so a cleared/reset profile yields exactly the defaults.

"Reset my timing" clears the stored profile (Req 6.7), after which the decoder
loads `defaultProfile()` and re-blends from defaults as new samples arrive.

## 8. Error handling

The decoder is **total**: no sequence of inputs ever throws or leaves it in an
invalid state (this is also a tested property, §9 P3). Specific guards:

- **Release without a matching press** (or vice versa): ignored; the `pressed` flag
  and `lastPressTs`/`lastReleaseTs` are kept consistent. A stray release does not
  classify a symbol or learn.
- **Non-monotonic or negative timestamps:** treated as no-gap / zero-duration and
  skipped for learning; they never produce a negative duration classification.
- **Zero-length press** (`duration == 0`): falls below `minPressDuration` and is
  ignored as tremor.
- **Invalid profile JSON / unknown version / NaN or negative fields:**
  `deserializeProfile` returns `null` and the decoder uses `defaultProfile()`. NaN
  or negative numbers in a syntactically valid payload are replaced with defaults
  before use.
- **Unknown dot/dash sequence on commit:** emits `□` (never throws, never `?`,
  Req 5.4/5.7).
- **Undo past empty:** no-op (never throws, never underflows the buffer).

The decoder performs no runtime network calls and runs with no connection (Req:
Offline). Classification, segmentation, and decoding complete well within a frame
at typical typing speeds (Req: Real-time latency) because the work per event is O(1)
amortised (a table lookup and a few arithmetic ops).

## 9. Testing strategy

Tests live under `tests/` mirroring `src/` paths (Req: Tech conformance), e.g.
`tests/morse/morse-decoder.test.ts`, `tests/morse/adaptive-timing.test.ts`,
`tests/morse/morse-table.test.ts`, `tests/morse/profile.test.ts`. They use Vitest
for unit tests and **fast-check** for property-based tests, driving the pure decoder
with explicit timestamps (no browser needed).

### Unit tests

- Morse table: every entry round-trips (encode char → code; decode code → char);
  `........` is not a printable entry; unknown codes → `□`.
- Adaptive timing: defaults reproduce the §4.1 table exactly; EWMA updates match the
  formula by hand for a few steps; `f`/`g` reach 1 at 5/5 samples; pause floors clamp;
  tremor filter ignores sub-threshold presses.
- Undo: `undoSymbol` pops partial-then-committed; single `........` deletes a
  letter; two in a row delete a word; edge cases above.
- Profile: `serializeProfile ∘ deserializeProfile` is the identity for valid
  profiles; bad JSON / wrong version / NaN fields yield `null`.

### Property-based tests (fast-check)

Correctness properties the implementation must satisfy for all generated inputs:

- **P1 — Encode/decode round-trip with ideal timing.** For any text drawn from the
  supported character set, encode each character to its code with ideal timing
  (dot = 1u, dash = 3u, within-letter gap = 1u, between-letter gap = 3u,
  between-word gap = 7u, in the decoder's own learned unit), feed the resulting
  press/release stream to the decoder, and assert the decoded text equals the
  original. (Req 5.)
- **P2 — Scale invariance once learned.** After the decoder has learned the user's
  timing from a calibration sequence, scaling every duration (press lengths and
  gaps) by a constant factor `k` does not change the decoded text, provided `k` is
  limited to a range where every scaled press stays above the 40 ms tremor floor
  (`MIN_PRESS_FLOOR_MS`) and no pause floor is set (`letterGapFloor = wordGapFloor =
  0`). Within those bounds the tremor filter and every classification/segmentation
  threshold scale with the learned averages, so the decoded text is unchanged.
  (Req 6.4, 6.5.)
- **P3 — Never crashes on random presses.** For any random sequence of
  press/release events with arbitrary (including non-monotonic and zero/negative)
  timestamps and durations, the decoder never throws and `getText()` is always a
  valid string. (Req: Error handling, totality.)
- **P4 — Dot/dash classification consistency.** A press below the narrow ambiguous
  band (`dotDashThreshold − 0.15·span`) is a dot; a press above the band
  (`dotDashThreshold + 0.15·span`) is a dash; a press inside the band follows the
  majority bias (tie → threshold side). Outside the band, classification is solely by
  which side of the threshold the press falls on. (Req 2.)
- **P5 — Segmentation correctness.** A gap below the letter threshold never ends a
  letter; a gap in `[letterThreshold, wordThreshold)` ends exactly one letter; a
  gap at/above the word threshold inserts exactly one space and is excluded from
  gap learning. (Req 3, 4.4.)
- **P6 — Tremor filter.** Any press shorter than `minPressDuration` produces an
  `ignored` event and never adds a symbol or updates an average. (Req 10.)
- **P7 — Pause floors bound thresholds.** After setting `letterGapFloor` and
  `wordGapFloor`, the effective letter and word thresholds are always ≥ their
  floors, for any sequence of presses. (Req 11.2.)
- **P8 — Undo invariants.** `undoSymbol` on a non-empty partial letter strictly
  shrinks `currentLetter`; on an empty partial letter it removes the last committed
  character. One `........` removes exactly one committed letter; two in a row first
  strip any trailing spaces, then remove the last word back to and including the
  space before it. The text length never goes negative. (Req 8.)
- **P9 — Placeholder behaviour.** An unsupported dot/dash sequence yields exactly
  `□`, which is marked non-spoken and is removable by undo. The decoder never emits
  `?` for an unknown sequence. (Req 5.4–5.7.)
- **P10 — Blend monotonicity.** As dot and dash samples grow from 0 to 5 each, `f`
  rises monotonically from 0 to 1 and the effective thresholds move from the
  defaults toward the learned averages; at 5/5 the effective averages equal the
  learned EWMAs. (Req 7.3.)
- **P11 — No fixed unit after learning.** After learning from a user whose unit
  differs from 240 ms, the classification thresholds differ from the hard-coded
  defaults — the decoder never assumes the fixed unit. (Req 2.2, 6.2.)
- **P12 — Profile round-trip.** `deserializeProfile(serializeProfile(p))` deep-equals
  `p` for every valid generated profile, and deserialising corrupted/unknown-version
  data returns `null`. (Req 6.6.)

P1, P2, and P3 are the three examples called out in the spec request; P4–P12 extend
coverage to classification, segmentation, the tremor filter, pause floors, undo,
the placeholder, the blend, the no-fixed-unit rule, and profile persistence.

## 10. Out of scope (unchanged from requirements.md)

Speech output, the quick-phrase board, word prediction, Urdu mode, practice mode,
carer view, two-switch mode, the service worker, the telegraph visual aesthetic,
and the `localStorage` read/write module itself are all out of scope. This design
only fixes the pure decoder, the adaptive timing engine, the Morse table, undo,
the serializable profile, and the input-capture adapter contract.
