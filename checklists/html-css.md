# HTML & CSS Review Checklist

> Use with [`../languages/html-css.md`](../languages/html-css.md) and
> [`general.md`](general.md).

- [ ] Prettier/Stylelint + axe/Lighthouse in CI; markup validated
      (`HTMLCSS-TOOL-*`).
- [ ] One naming convention applied consistently; `data-testid` reserved for
      tests (`HTMLCSS-NAME-*`).
- [ ] Semantic elements for meaning (`button`/`a`/landmarks); one `h1`;
      sequential heading levels (`HTMLCSS-SEM-*`).
- [ ] Every image has correct `alt` (`""` if decorative); every form control
      has a real `<label>`; errors linked via `aria-describedby` (`HTMLCSS-A11Y-*`,
      `HTMLCSS-FORM-*`).
- [ ] Visible focus indicator retained/replaced (never `outline: none` with
      nothing); custom widgets keyboard-operable with correct ARIA
      (`HTMLCSS-A11Y-03/04/05`).
- [ ] WCAG 2.2 AA contrast met; color not the only information channel
      (`HTMLCSS-A11Y-06`).
- [ ] Server re-validates everything client-side form validation checks
      (`HTMLCSS-FORM-02`).
- [ ] Low-specificity selectors; no routine `!important`; tokens via CSS
      custom properties; component styles scoped (`HTMLCSS-CSS-*`).
- [ ] Grid vs Flexbox chosen correctly; mobile-first media queries; relative
      units for type/spacing (`HTMLCSS-LAYOUT-*`).
- [ ] Explicit image/video dimensions or `aspect-ratio`; non-critical scripts
      `defer`/`async` (`HTMLCSS-PERF-*`).
- [ ] No unescaped user input in markup; `noopener noreferrer` on external
      `target="_blank"`; no untrusted-value CSS attribute selectors
      (`HTMLCSS-SEC-*`).
- [ ] Automated a11y + visual-regression + keyboard-navigation tests present
      for custom components (`HTMLCSS-TEST-*`).
