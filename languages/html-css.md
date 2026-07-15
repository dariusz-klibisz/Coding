# HTML & CSS — Coding Standards & State-of-the-Art Practices

> **Scope:** Semantic HTML5 and modern CSS3 (Grid, Flexbox, custom properties,
> nesting, container/media queries) as authored directly or via a framework's
> template/style layer. Coding-level rules only — page/application architecture,
> rendering strategy, and Core Web Vitals *targets* are a design concern, not a
> coding one, and live outside this library.
> **Primary sources:** WHATWG HTML Living Standard, MDN Web Docs, W3C CSS
> specifications, WCAG 2.2.
> **Relationship to general docs:** extends [general docs](../00-index.md). Where a
> general rule and an HTML/CSS rule conflict, the HTML/CSS rule wins for markup and
> styles. See also [`languages/vue.md`](vue.md) and [`languages/typescript.md`](typescript.md)
> for component-authored markup in those stacks.

**Rule ID prefix:** `HTMLCSS`

---

## Table of contents

1. [Tooling & enforcement](#1-tooling--enforcement)
2. [Naming conventions](#2-naming-conventions)
3. [Formatting & layout](#3-formatting--layout)
4. [Semantic HTML](#4-semantic-html)
5. [Accessible markup (coding-level)](#5-accessible-markup-coding-level)
6. [Forms](#6-forms)
7. [CSS architecture & selectors](#7-css-architecture--selectors)
8. [Layout: Flexbox, Grid & responsive design](#8-layout-flexbox-grid--responsive-design)
9. [Error handling & graceful degradation](#9-error-handling--graceful-degradation)
10. [Resource loading & performance idioms](#10-resource-loading--performance-idioms)
11. [Security](#11-security)
12. [Testing](#12-testing)
13. [Anti-patterns](#13-anti-patterns)
14. [Quick checklist](#quick-checklist)
15. [References](#references)

---

## 1. Tooling & enforcement

**`HTMLCSS-TOOL-01` (SHOULD)** Format HTML/CSS with **Prettier** (or the
framework's bundled formatter) and lint with **Stylelint** (CSS) and
`eslint-plugin-jsx-a11y`/**axe-core**-based linters where markup is generated
from JS/TS. Builds on [tooling & automation](../11-tooling-and-automation.md).

**`HTMLCSS-TOOL-02` (SHOULD)** Run an automated accessibility linter (axe,
Lighthouse, or Pa11y) in CI against key pages/components — it catches missing
labels, contrast failures, and ARIA misuse that a formatter cannot.

**`HTMLCSS-TOOL-03` (SHOULD)** Validate HTML against the WHATWG spec (the W3C
Nu Html Checker or an editor's built-in validator) — invalid markup causes
inconsistent parsing/rendering across browsers and assistive technology.

---

## 2. Naming conventions

| Construct | Convention | Example |
|-----------|------------|---------|
| CSS class (BEM) | `block__element--modifier` | `card__title--highlighted` |
| CSS class (utility-first, e.g. Tailwind) | framework's own convention | `flex items-center gap-2` |
| Custom property (CSS variable) | `--kebab-case`, namespaced | `--color-primary-500` |
| `id` attribute | `kebab-case`, used sparingly (see §7) | `main-nav` |
| `data-*` attribute | `kebab-case`, purpose-prefixed | `data-testid`, `data-analytics-id` |
| File name | `kebab-case.css` / `.html` | `product-card.css` |

**`HTMLCSS-NAME-01` (SHOULD)** Pick **one** class-naming convention per project
(BEM, utility-first, or CSS Modules' scoped names) and apply it consistently —
mixing conventions makes selector specificity and ownership unpredictable.

**`HTMLCSS-NAME-02` (SHOULD)** Reserve `data-testid`/`data-test` attributes
exclusively for test selectors; do not repurpose them for styling or behavior
hooks, and do not style off them (`[data-testid=...] { }`) — a test attribute
change should never break layout.

---

## 3. Formatting & layout

**`HTMLCSS-FMT-01` (SHOULD)** 2-space indentation for both HTML and CSS; one
selector/declaration block per rule; one property per line. Let Prettier/
Stylelint enforce this rather than debating it per PR (`GEN-TOOL-02`).

**`HTMLCSS-FMT-02` (SHOULD)** Order CSS properties consistently (logical
grouping: box model → typography → visual → misc, or an automated
property-order Stylelint rule) so diffs are predictable.

**`HTMLCSS-FMT-03` (SHOULD)** Always declare `<!DOCTYPE html>`, `lang` on
`<html>`, and `<meta charset="utf-8">` first in `<head>` — omitting them
triggers quirks mode or mis-detected encoding.

---

## 4. Semantic HTML

**`HTMLCSS-SEM-01` (SHOULD)** Use the element whose native semantics match the
content's *meaning*, not its default appearance: `<button>` for actions,
`<a href>` for navigation, `<nav>`/`<main>`/`<header>`/`<footer>`/`<article>`/
`<section>`/`<aside>` for page structure, `<time datetime>` for dates,
`<table>` for tabular data.

**Why:** Semantic elements carry built-in keyboard interaction, focus
behavior, and ARIA roles for free — screen readers, browser find-in-page, and
translation tools all rely on them. A `<div onclick>` styled as a button has
none of that; it is not keyboard-operable, not announced as a button, and not
included in the accessibility tree's interactive-elements list without manual
ARIA to reconstruct what `<button>` already provides.

```html
<!-- Bad: reimplements a button with none of its behavior -->
<div class="btn" onclick="submit()">Submit</div>

<!-- Good: native semantics, keyboard/focus/role for free -->
<button type="submit">Submit</button>
```

**`HTMLCSS-SEM-02` (SHOULD)** Use exactly one `<h1>` per page (or per
top-level document in a component library) and keep heading levels
sequential (`h1`→`h2`→`h3`, no skipped levels) — headings form the
document outline assistive tech uses for navigation.

**`HTMLCSS-SEM-03` (SHOULD)** Use `<ul>`/`<ol>`/`<li>` for any list of items,
even when styled to not look like a list — screen readers announce list
semantics ("list, 5 items") which orients the user regardless of visual
styling.

---

## 5. Accessible markup (coding-level)

Design-level accessibility strategy (WCAG conformance targets, audits, policy)
is outside this library's scope; these are the coding idioms that satisfy it.

**`HTMLCSS-A11Y-01` (MUST)** Give every non-decorative `<img>` a meaningful
`alt`; give purely decorative images `alt=""` (not a missing attribute) so
screen readers skip them instead of announcing the filename.

**`HTMLCSS-A11Y-02` (MUST)** Associate every form control with a visible
`<label>` (via `for`/`id`, or by wrapping) — a placeholder is not a label; it
disappears on input and is not reliably announced.

```html
<!-- Bad: placeholder as the only "label" -->
<input type="email" placeholder="Email">

<!-- Good: explicit, persistent label -->
<label for="email">Email</label>
<input type="email" id="email" name="email">
```

**`HTMLCSS-A11Y-03` (SHOULD)** Preserve a visible **focus indicator** for
every interactive element; if overriding the default outline, replace it with
an equally visible custom style (`:focus-visible`) — never `outline: none`
with nothing in its place.

**`HTMLCSS-A11Y-04` (SHOULD)** Ensure interactive elements are reachable and
operable via keyboard alone (native `<button>`/`<a>`/`<input>` are by
default). If a custom interactive widget is unavoidable (`<div role="button">`),
it MUST additionally get `tabindex="0"`, a keydown handler for
Enter/Space, and the correct ARIA role — this is strictly more work than
using the native element, which is why native elements are preferred (§4).

**`HTMLCSS-A11Y-05` (SHOULD)** Use ARIA only to fill a gap native HTML cannot
— "No ARIA is better than bad ARIA" (WAI-ARIA Authoring Practices). Don't add
`role`/`aria-*` to an element that already has the correct implicit semantics
(`role="button"` on a `<button>` is redundant at best).

**`HTMLCSS-A11Y-06` (SHOULD)** Meet **WCAG 2.2 AA** contrast ratios in CSS
color choices — 4.5:1 for normal text, 3:1 for large text/UI components —
and don't convey information (errors, required fields, status) through color
alone; pair it with text, an icon, or a pattern.

---

## 6. Forms

**`HTMLCSS-FORM-01` (SHOULD)** Use the correct `input` `type` (`email`, `tel`,
`number`, `date`, `url`) and `inputmode`/`autocomplete` attributes — this
gets the right mobile keyboard and browser autofill for free and is a coding
change, not a design one.

**`HTMLCSS-FORM-02` (MUST)** Never rely on client-side HTML validation
(`required`, `pattern`, `min`/`max`) as the only validation — it is a UX
convenience, not a security boundary; the server must re-validate everything
(`GEN-SEC-01`, `GEN-DEF-02`).

**`HTMLCSS-FORM-03` (SHOULD)** Associate error messages with their field via
`aria-describedby` and mark the invalid field with `aria-invalid="true"`, so
assistive tech announces the error when the field receives focus.

---

## 7. CSS architecture & selectors

**`HTMLCSS-CSS-01` (SHOULD)** Prefer low-specificity, class-based selectors
over `id` selectors and deep descendant chains
(`.card .header .title` → `.card__title`). High-specificity selectors are
hard to override later and force `!important` escalation wars.

**`HTMLCSS-CSS-02` (MUST NOT)** Avoid `!important` except as a narrow,
commented escape hatch (overriding an inline style you don't control, a
utility class's deliberate design). Reaching for it routinely means the
underlying specificity/cascade design has failed.

**`HTMLCSS-CSS-03` (SHOULD)** Use CSS **custom properties** (`--token-name`)
for design tokens (color, spacing, radius) instead of hard-coded literals
scattered across the stylesheet — one change point instead of a
find-and-replace (`GEN-PRIN-04` DRY).

```css
/* Bad: the same magic value repeated, and will drift */
.button { padding: 12px 16px; }
.card   { padding: 12px 16px; }

/* Good: one source of truth */
:root { --space-3: 12px; --space-4: 16px; }
.button { padding: var(--space-3) var(--space-4); }
.card   { padding: var(--space-3) var(--space-4); }
```

**`HTMLCSS-CSS-04` (SHOULD)** Scope component styles (CSS Modules, `scoped`/
`<style module>` blocks, Shadow DOM, or a strict BEM discipline) so a
component's styles cannot leak into or be overridden by unrelated markup
elsewhere in the page.

**`HTMLCSS-CSS-05` (CONSIDER)** Use native **CSS nesting** and `@layer` (all
evergreen browsers, 2023+) to express component hierarchy and control cascade
order explicitly, instead of a preprocessor purely for that purpose.

---

## 8. Layout: Flexbox, Grid & responsive design

**`HTMLCSS-LAYOUT-01` (SHOULD)** Use **Grid** for two-dimensional layout
(rows and columns together) and **Flexbox** for one-dimensional layout
(a single row or column of items) — reaching for the wrong one produces
awkward workarounds (Flexbox "grid hacks" with manual `flex-basis` math where
Grid's `grid-template-columns` states the intent directly).

**`HTMLCSS-LAYOUT-02` (SHOULD)** Write layout mobile-first: base styles for
the smallest viewport, then widen with `min-width` media queries — this
keeps the common case (small viewport) unconditional and additive rather
than an override chain.

**`HTMLCSS-LAYOUT-03` (CONSIDER)** Use **container queries**
(`@container`) instead of viewport media queries when a component's layout
should respond to its container's size, not the viewport's — this makes the
component correctly reusable in a sidebar, a modal, or a full-width section
without per-context overrides.

**`HTMLCSS-LAYOUT-04` (SHOULD)** Use relative units (`rem` for spacing/type
scale, `%`/`fr`/`ch` for layout) over hardcoded `px` for anything that should
scale with the user's font-size preference or container — this is an
accessibility requirement (WCAG 1.4.4 reflow/zoom) as much as a layout one.

---

## 9. Error handling & graceful degradation

**`HTMLCSS-ERR-01` (SHOULD)** Provide a functional fallback (server-rendered
markup, a `<noscript>` message, native form submission) for the core action
of a page when JavaScript fails to load or execute — progressive enhancement,
not a hard JS requirement, for content that doesn't need it.

**`HTMLCSS-ERR-02` (SHOULD)** Use `<picture>`/`srcset` with a sensible
fallback `<img src>` so a browser or network condition that can't use the
optimal format/size still renders a working image, not a broken one.

---

## 10. Resource loading & performance idioms

Cross-cutting performance strategy (bundling, rendering strategy, Core Web
Vitals targets) is out of scope here; these are markup-level idioms. Builds
on [performance](../06-performance.md).

**`HTMLCSS-PERF-01` (SHOULD)** Set explicit `width`/`height` (or `aspect-ratio`
in CSS) on `<img>`/`<video>` so the browser reserves layout space before the
asset loads — omitting it causes layout shift as content loads in.

**`HTMLCSS-PERF-02` (SHOULD)** Use `loading="lazy"` on below-the-fold images
and `decoding="async"` where supported; don't lazy-load the largest
above-the-fold image (it is usually the Largest Contentful Paint candidate).

**`HTMLCSS-PERF-03` (SHOULD)** Load third-party/non-critical `<script>` tags
with `defer` or `async` (never a blocking synchronous `<script>` in `<head>`)
so parsing and rendering aren't stalled waiting on network scripts.

---

## 11. Security

Builds on [security](../04-security.md). HTML/CSS-specific:

**`HTMLCSS-SEC-01` (MUST)** Never inject unescaped user input into HTML —
this is the DOM-based/stored-XSS sink (`GEN-SEC-08`). Use the templating
engine/framework's auto-escaping; avoid `innerHTML`/`insertAdjacentHTML`/
`document.write` with untrusted strings.

**`HTMLCSS-SEC-02` (MUST)** Add `rel="noopener noreferrer"` to any
`target="_blank"` link to an untrusted/external URL — without it, the opened
page can access `window.opener` and navigate the original tab (reverse
tabnabbing).

**`HTMLCSS-SEC-03` (SHOULD)** Avoid CSS-based data exfiltration vectors:
don't build attribute-selector CSS from untrusted values
(`input[value^="a"] { background: url(...) }` can leak input character-by
-character to an attacker-controlled URL); this pattern has been used to
exfiltrate CSRF tokens and form values purely through CSS.

---

## 12. Testing

Builds on [testing](../05-testing.md). HTML/CSS-specific:

**`HTMLCSS-TEST-01` (SHOULD)** Run automated accessibility checks (axe,
`jest-axe`, Playwright's accessibility snapshot) as part of component/page
tests, not only as a manual pre-release pass.

**`HTMLCSS-TEST-02` (SHOULD)** Use visual-regression testing (Percy,
Chromatic, Playwright screenshots) for CSS-heavy components — logic-level
unit tests cannot catch a layout regression.

**`HTMLCSS-TEST-03` (SHOULD)** Test keyboard navigation explicitly (Tab
order, Enter/Space activation, Escape to close) for any custom interactive
component — this is exactly the behavior native elements give for free and
custom ones must prove they've reimplemented (§5).

---

## 13. Anti-patterns

- `<div>`/`<span>` with a click handler standing in for `<button>`/`<a>`.
- Missing/empty `alt` on meaningful images; decorative images with a
  descriptive `alt` (should be `alt=""`).
- Placeholder text used as the only label.
- `outline: none` with no replacement focus style.
- Skipped heading levels; more than one `<h1>`.
- Deep, high-specificity selector chains; routine `!important`.
- Hardcoded color/spacing literals instead of custom properties/tokens.
- Global, unscoped component styles that leak or get overridden.
- Client-side-only form validation with no server-side re-validation.
- `target="_blank"` without `rel="noopener noreferrer"`.
- Unescaped user input rendered via `innerHTML`/`document.write`.
- Images/video with no reserved space, causing layout shift.
- Blocking synchronous third-party `<script>` in `<head>`.
- Building CSS attribute selectors from untrusted values.

---

## Quick checklist

- [ ] Prettier/Stylelint + axe/Lighthouse accessibility lint in CI; HTML
      validated.
- [ ] One naming convention (BEM/utility/CSS Modules) applied consistently;
      `data-testid` reserved for tests only.
- [ ] Semantic elements used for their meaning (`button`/`a`/`nav`/headings
      sequential, one `h1`).
- [ ] Every image has correct `alt` (`""` for decorative); every form control
      has a real `<label>`.
- [ ] Visible focus indicator on all interactive elements; custom widgets are
      keyboard-operable with correct ARIA role.
- [ ] WCAG 2.2 AA contrast met; color never the sole information channel.
- [ ] Correct `input type`/`autocomplete`; server re-validates everything
      client-side validation checks.
- [ ] Low-specificity selectors; no routine `!important`; design tokens via
      custom properties.
- [ ] Component styles scoped; Grid vs Flexbox chosen by dimensionality;
      mobile-first media queries; container queries where apt.
- [ ] Relative units for type/spacing that should scale/reflow.
- [ ] Progressive enhancement / fallback for critical actions without JS.
- [ ] Explicit image/video dimensions or `aspect-ratio`; lazy-load
      below-the-fold only; non-critical scripts `defer`/`async`.
- [ ] No unescaped user input in markup; `noopener noreferrer` on external
      `target="_blank"`; no untrusted-value CSS attribute selectors.
- [ ] Automated a11y tests + visual regression + keyboard-navigation tests
      for custom components.

---

## References

- WHATWG HTML Living Standard — https://html.spec.whatwg.org/
- MDN Web Docs, HTML reference — https://developer.mozilla.org/en-US/docs/Web/HTML
- MDN Web Docs, CSS reference — https://developer.mozilla.org/en-US/docs/Web/CSS
- W3C, Web Content Accessibility Guidelines (WCAG) 2.2 — https://www.w3.org/TR/WCAG22/
- WAI-ARIA Authoring Practices Guide — https://www.w3.org/WAI/ARIA/apg/
- web.dev, "Learn CSS" and "Learn HTML" — https://web.dev/learn/
- Stylelint — https://stylelint.io/ · Prettier — https://prettier.io/
- axe-core — https://github.com/dequelabs/axe-core
