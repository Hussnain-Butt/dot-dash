---
inclusion: fileMatch
fileMatchPattern:
  - src/morse/**
  - tests/**
---

# Dot-Dash — Morse Timing

## Standard timing (in relative units)

| Element          | Units |
|------------------|-------|
| Dot              | 1     |
| Dash             | 3     |
| Gap inside letter| 1     |
| Gap between letters | 3  |
| Gap between words | 7    |

## Character set (International Morse)

- Letters A–Z and digits 0–9.
- Common punctuation: period `.`, comma `,`, question mark `?`, apostrophe `'`, exclamation `!`, slash `/`, parentheses `(` `)`, ampersand `&`, colon `:`, semicolon `;`, equals `=`, plus `+`, minus/hyphen `-`, underscore `_`, quotation mark `"`, at sign `@`.
- Unknown or invalid sequences decode to a placeholder character, not silence.

## Adaptive timing rule

Adaptive timing must tolerate slow and uneven presses without misreading. The decoder must not assume a fixed unit length; it must learn each user's own dot, dash, and pause thresholds and adjust continuously. A press that is ambiguous should err toward the user's most common pattern, not toward an error.
