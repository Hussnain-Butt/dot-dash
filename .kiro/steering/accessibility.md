---
inclusion: always
---

# Dot-Dash — Accessibility

- Target WCAG 2.2 AA. Text contrast ratio at least 4.5:1.
- Every action works with one key or one switch. No action requires a second simultaneous input.
- Touch and click targets at least 44 × 44 px.
- Visible focus indicator on all interactive elements.
- User-interface components, focus indicators, and the large Morse button need at least 3:1 contrast against the area next to them, per WCAG 1.4.11.
- No information is shown by colour alone: dots, dashes, and states also differ in shape or text, per WCAG 1.4.1.
- Screen-reader labels on all controls (aria-label or visible text).
- A live region (aria-live) announces decoded text as it forms.
- Respect prefers-reduced-motion: no non-essential animation.
- No time limits that punish slow users. The adaptive timer adjusts to the user; it never forces a fixed speed.
- The user can always undo the last letter (and the last word) with one action.
