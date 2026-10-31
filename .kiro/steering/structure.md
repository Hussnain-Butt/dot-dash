---
inclusion: always
---

# Dot-Dash — Structure

## Layout

```
public/      static assets, service worker, manifest
src/
  morse/     pure decoder and timing logic (no UI imports)
  ui/        components, input handling, rendering
  speech/    Web Speech API wrapper
  storage/   localStorage read/write
  words/     built-in word list and prediction
tests/       unit and property-based tests
```

## Naming rules

- Files: kebab-case (`morse-decoder.ts`).
- Types and classes: PascalCase (`MorseDecoder`).
- Functions and variables: camelCase (`decodeLetter`).
- Tests live under `tests/`, mirroring `src/` paths (`tests/morse/morse-decoder.test.ts`).

## Purity rule

Code in `src/morse/` must stay pure: no DOM, no UI framework, no side effects. It takes timing data in and returns decoded text out. The UI layer calls it; it never calls the UI.
