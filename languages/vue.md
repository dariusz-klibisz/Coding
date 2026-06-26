# Vue 3 SFC (`.vue`) — Coding Standards & State-of-the-Art Practices

> **Scope:** Vue 3 Single-File Components with `<script setup>` + TypeScript +
> Composition API + Vite + Pinia.
> **Primary source:** the official Vue.js Style Guide (Priority A–D). The official
> A and B rules quoted here are verified against vuejs.org.
> **Relationship to general docs:** extends [general docs](../00-README.md) and builds
> directly on [typescript.md](typescript.md) (all TS rules apply inside `.vue`
> `<script setup lang="ts">`). Where rules conflict, this file wins for `.vue`.

**Rule ID prefix:** `VUE`

---

## Table of contents

1. [How the official style guide is organized](#1-how-the-official-style-guide-is-organized)
2. [Tooling & enforcement](#2-tooling--enforcement)
3. [Priority A — Essential (error prevention)](#3-priority-a--essential-error-prevention)
4. [Priority B — Strongly Recommended](#4-priority-b--strongly-recommended)
5. [SFC structure & `<script setup>`](#5-sfc-structure--script-setup)
6. [Composition API: reactivity](#6-composition-api-reactivity)
7. [Props & emits with TypeScript](#7-props--emits-with-typescript)
8. [Computed, watch, and lifecycle](#8-computed-watch-and-lifecycle)
9. [Composables](#9-composables)
10. [State management (Pinia)](#10-state-management-pinia)
11. [Templates & directives](#11-templates--directives)
11a. [Slots](#11a-slots)
12. [Performance](#12-performance)
13. [Security](#13-security)
13a. [Forms](#13a-forms)
13b. [Accessibility](#13b-accessibility)
14. [Testing](#14-testing)
15. [Anti-patterns](#15-anti-patterns)
16. [Quick checklist](#quick-checklist)
17. [References](#references)

---

## 1. How the official style guide is organized

The Vue Style Guide ranks rules by priority:

- **A — Essential:** prevent errors; follow at all costs.
- **B — Strongly Recommended:** improve readability/DX; violations should be rare.
- **C — Recommended:** consistency among equally-good options.
- **D — Use with Caution:** features that are risky when overused.

**`VUE-STD-01` (SHOULD)** Adopt the official style guide and enforce its
mechanizable rules with `eslint-plugin-vue` (the `vue3-recommended`/`flat/
recommended` config encodes most A/B rules).

---

## 2. Tooling & enforcement

**`VUE-TOOL-01` (MUST)** Use **Vite** + **`eslint-plugin-vue`** + **Prettier** +
**`vue-tsc`** for type-checking `.vue` files (template + script). Run `vue-tsc
--noEmit` in CI — the regular `tsc` doesn't understand SFC templates.

**`VUE-TOOL-02` (SHOULD)** Use Volar (Vue Language Tools) in the editor for
template type-checking and IntelliSense. Enable `eslint-plugin-vue` flat
recommended config; treat lint/type errors as build failures.

**`VUE-TOOL-03` (SHOULD)** All [typescript.md](typescript.md) tooling and `strict`
config apply to `<script setup lang="ts">`.

---

## 3. Priority A — Essential (error prevention)

These are verbatim-sourced from the official guide.

**`VUE-A-01` (MUST)** **Multi-word component names** (except the root `App`).
Single-word names risk conflicting with current/future HTML elements (all of which
are single words). `TodoItem`, not `Todo`/`Item`.

**`VUE-A-02` (MUST)** **Detailed prop definitions.** In committed code, define
props with at least their type — ideally `required`/`default`/`validator`. This
documents the component API and lets Vue warn on misuse in development.

**`VUE-A-03` (MUST)** **Keyed `v-for`.** Always use `:key` with `v-for` (required
on components; best practice on elements) to maintain component state and
predictable DOM behavior (object constancy). Use a stable unique id, not the array
index, when the list can reorder/insert/delete.

**`VUE-A-04` (MUST)** **Never use `v-if` on the same element as `v-for`.** `v-for`
has higher priority, so the `v-if` may reference a not-yet-defined iteration
variable, and you re-evaluate the condition every iteration. Filter with a
`computed` (e.g. `activeUsers`) or move `v-if` to a wrapping `<template>`/
container.

```vue
<!-- Bad -->
<li v-for="user in users" v-if="user.isActive" :key="user.id">{{ user.name }}</li>

<!-- Good: filter in a computed -->
<li v-for="user in activeUsers" :key="user.id">{{ user.name }}</li>
```

**`VUE-A-05` (MUST)** **Component-scoped styling.** Outside the top-level `App`/
layout components, styles should be scoped — via `<style scoped>`, CSS Modules, or
a class strategy (BEM). Prevents styles leaking across components. (Component
libraries should prefer a class-based strategy over `scoped` for easier overrides.)

---

## 4. Priority B — Strongly Recommended

Sourced from the official guide; apply unless you have a documented reason.

**`VUE-B-01` (SHOULD)** **One component per file** (`.vue`).

**`VUE-B-02` (SHOULD)** **SFC filename casing** consistently **PascalCase**
(`TodoItem.vue`) — best for editor autocompletion and JS/template consistency — or
consistently kebab-case. Don't mix.

**`VUE-B-03` (SHOULD)** **Base/presentational components** that apply app-wide
styling and contain no global state get a consistent prefix: `Base`, `App`, or
`V` (`BaseButton`, `AppTable`, `VIcon`).

**`VUE-B-04` (SHOULD)** **Tightly-coupled child components** include the parent name
as a prefix (`TodoList`, `TodoListItem`, `TodoListItemButton`) — keeps related
files together and signals the relationship.

**`VUE-B-05` (SHOULD)** **Order words from general → specific** in names
(`SearchButtonClear`, not `ClearSearchButton`) so alphabetical file listings group
related components.

**`VUE-B-06` (SHOULD)** **Self-close** components with no content in SFCs
(`<MyComponent />`).

**`VUE-B-07` (SHOULD)** **PascalCase component names in SFC templates**
(`<MyComponent />`) — distinct from HTML elements and consistent with JS.

**`VUE-B-08` (SHOULD)** **Full words over abbreviations** in names
(`UserProfileOptions`, not `UProfOpts`).

**`VUE-B-09` (SHOULD)** **camelCase prop declarations**; in SFC templates use a
consistent style (kebab-case or camelCase — don't mix).

**`VUE-B-10` (SHOULD)** **One attribute per line** for multi-attribute elements
(readability, cleaner diffs).

**`VUE-B-11` (SHOULD)** **Only simple expressions in templates** — move complex
logic into `computed` properties or methods. Templates should declare *what*
renders, not *how* it's computed.

**`VUE-B-12` (SHOULD)** **Split complex computed properties** into several small,
well-named ones — easier to read, test, and adapt.

**`VUE-B-13` (SHOULD)** **Quote attribute values** (`type="text"`).

**`VUE-B-14` (SHOULD)** **Directive shorthands consistently** — use `:`/`@`/`#` (or
the long forms) *always or never*, not mixed.

---

## 5. SFC structure & `<script setup>`

**`VUE-SFC-01` (SHOULD)** Use **`<script setup lang="ts">`** for components — it's
the modern, most concise, best-typed authoring style (compile-time props/emits
typing, less boilerplate than `setup()` or the Options API).

**`VUE-SFC-02` (SHOULD)** Order blocks consistently: **`<script setup>`**, then
**`<template>`**, then **`<style scoped>`**. Pick one order and keep it project-wide
(`<script>`-first is the common modern convention).

**`VUE-SFC-03` (SHOULD)** Within `<script setup>`, group logically: imports →
props/emits → composables/stores → reactive state → computed → watchers →
lifecycle hooks → functions. Co-locate code by *feature/concern* (the key
advantage of Composition API over Options API), not by option type.

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'
import { useUserStore } from '@/stores/user'

const props = defineProps<{ userId: number }>()
const emit = defineEmits<{ save: [id: number] }>()

const store = useUserStore()
const isEditing = ref(false)
const displayName = computed(() => store.userById(props.userId)?.name ?? 'Unknown')

function startEdit() {
  isEditing.value = true
}
</script>

<template>
  <button @click="startEdit">{{ displayName }}</button>
</template>

<style scoped>
button { font-weight: 600; }
</style>
```

---

## 6. Composition API: reactivity

**`VUE-RX-01` (SHOULD)** Use **`ref()`** as the default for reactive state
(primitives and objects); use **`reactive()`** only for objects where you never
reassign the whole value.

**`ref` vs `reactive`:**

| | `ref` | `reactive` |
|---|-------|-----------|
| Works with | any value (primitive or object) | objects/arrays/maps/sets only |
| Access in script | `.value` | direct |
| Reassignable wholesale | ✅ `r.value = newObj` | ❌ loses reactivity if replaced |
| Destructuring keeps reactivity | with `toRefs`/`.value` | ❌ destructuring breaks it |

**Pros of standardizing on `ref`:** one consistent mental model; can be reassigned;
survives being returned from composables. **Cons:** the `.value` ceremony in
script (templates auto-unwrap). **`reactive` pros:** no `.value`; **cons:** loses
reactivity on reassignment/destructuring and can't hold primitives — which is why
`ref` is the recommended default.

> **Note:** `reactive()` is reasonable for a **cohesive object whose whole value is
> never reassigned** (e.g. a local form state grouped together). The trade-off is
> the reassignment/destructuring hazards above. When in doubt, default to `ref`;
> pick one approach per module and stay consistent rather than mixing both for the
> same kind of state.

**`VUE-RX-02` (SHOULD)** Don't destructure a `reactive` object or props (loses
reactivity). Use `toRefs`/`toRef`, or keep accessing via the object. Props
destructuring with reactivity is supported in newer Vue but be explicit/consistent.

**`VUE-RX-03` (SHOULD)** Don't mutate state outside its owner. Child components must
**not** mutate props (one-way data flow, §7); use a `computed` or local copy, and
emit events upward.

---

## 7. Props & emits with TypeScript

**`VUE-PROP-01` (SHOULD)** Declare props and emits with **type-based** generics
(best typing) and use `withDefaults` for defaults:

```ts
const props = withDefaults(
  defineProps<{ status: 'idle' | 'loading' | 'error'; count?: number }>(),
  { count: 0 },
)
const emit = defineEmits<{
  change: [value: number]   // named tuple = event payload types
  close: []
}>()
```

**Why type-based:** the compiler validates prop/emit usage at build time
(`VUE-A-02` satisfied with full static types), and consumers get IntelliSense.

**`VUE-PROP-02` (MUST)** Treat props as **read-only**. Never assign to a prop —
mutating props breaks one-way data flow and triggers warnings. Derive a `computed`
or emit an event for the parent to update.

**`VUE-PROP-03` (SHOULD)** Use `defineModel()` for two-way binding (`v-model`)
instead of the manual `modelValue` prop + `update:modelValue` emit boilerplate.

**`VUE-PROP-04` (SHOULD)** Validate/constrain props with precise types (literal
unions over bare `string`) so illegal values are unrepresentable (`GEN-PRIN-28`).

---

## 8. Computed, watch, and lifecycle

**`VUE-CW-01` (SHOULD)** Use **`computed`** for derived state (cached, recomputed
only when dependencies change). Don't use a `watch` to manually recompute a value
that a `computed` expresses declaratively.

**`VUE-CW-02` (SHOULD)** Keep `computed` getters **pure** — no side effects, no
async. Side effects belong in `watch`/`watchEffect` or event handlers.

**`VUE-CW-03` (SHOULD)** Reserve **`watch`/`watchEffect`** for side effects
reacting to state change (fetching on id change, syncing to storage). Prefer
`watch` with explicit sources for clarity; use `watchEffect` when dependencies are
many/obvious. Clean up side effects (`onWatcherCleanup`/returned cleanup) to avoid
leaks and races.

**`VUE-CW-04` (SHOULD)** Use lifecycle hooks (`onMounted`, `onUnmounted`, …) for
DOM/subscription setup and teardown; always remove listeners/intervals/observers
in `onUnmounted` to prevent memory leaks (`GEN-DEF-15`).

**Computed pros:** cached, declarative, dependency-tracked. **Cons:** must be pure
and synchronous — for async/side-effects use `watch`.

---

## 9. Composables

**`VUE-COMP-01` (SHOULD)** Extract reusable stateful logic into **composables**
(`useXxx` functions) — the Composition API replacement for mixins. Name them
`useFeature`, return refs/computed/functions.

**Why composables over mixins:** mixins have implicit name collisions and unclear
data sources; composables make inputs/outputs explicit, are typed, and compose
cleanly (`GEN-PRIN-12` composition over inheritance).

**`VUE-COMP-02` (SHOULD)** Keep composables focused (one concern), return a plain
object of refs/functions, accept refs/getters as inputs where reactivity must flow
in, and handle their own cleanup (`onUnmounted`).

```ts
export function useMouse() {
  const x = ref(0)
  const y = ref(0)
  function update(e: MouseEvent) { x.value = e.pageX; y.value = e.pageY }
  onMounted(() => window.addEventListener('mousemove', update))
  onUnmounted(() => window.removeEventListener('mousemove', update))
  return { x, y }
}
```

---

## 10. State management (Pinia)

**`VUE-STATE-01` (SHOULD)** Use **Pinia** (the official store) for cross-component/
app-wide state; keep component-local state in the component. Don't lift everything
into global stores — local state is simpler and more encapsulated.

**`VUE-STATE-02` (SHOULD)** Prefer the **setup-store** style (a function returning
refs/computed/actions) for the best TypeScript inference and consistency with
`<script setup>`. Define `state` as data, `getters` as derived (`computed`), and
`actions` as the only place that mutates state.

**`VUE-STATE-03` (SHOULD)** Mutate store state only through actions (single source
of change → predictable, testable, traceable). Destructure store state with
`storeToRefs()` to preserve reactivity; actions can be destructured directly.

**`VUE-STATE-04` (SHOULD)** Don't put server-cache concerns (caching, refetching,
dedup of API data) into Pinia by hand — consider a data-fetching/query library
(e.g. TanStack Query/`@pinia/colada`) for that, and keep Pinia for genuine client
state. (Separation of concerns, `GEN-PRIN-05`.)

---

## 11. Templates & directives

**`VUE-TPL-01` (SHOULD)** Keep templates declarative: simple expressions only
(`VUE-B-11`), logic in `computed`/methods, and break large templates into smaller
components when they grow unwieldy (single responsibility).

**`VUE-TPL-02` (SHOULD)** Use `v-show` for elements toggled frequently (keeps them
in the DOM, toggles CSS) and `v-if` for rarely-toggled/conditionally-mounted
content (adds/removes from DOM). Choosing wrong wastes render or memory.

**`VUE-TPL-03` (SHOULD NOT)** Avoid `v-html` with untrusted content — it's an XSS
sink (`GEN-SEC-08`, §13). Never bind user input to it without sanitizing.

**`VUE-TPL-04` (SHOULD)** Bind `key` thoughtfully for `<transition>`/list animations
and to force remounts when intended. Use `<template>` wrappers to apply directives
to groups without extra DOM nodes.

**`VUE-TPL-05` (SHOULD)** Prefer slots and props for composition over reaching into
child internals (`$refs`/`$parent`) — keep components loosely coupled
(`GEN-PRIN-06`, Law of Demeter).

---

## 11a. Slots

**`VUE-SLOT-01` (SHOULD)** Use slots to let a parent inject markup, keeping reusable
components open for composition without prop explosion. Name multiple slots
(`<slot name="header" />`) rather than overloading the default slot.

**`VUE-SLOT-02` (SHOULD)** Document slot props (scoped slots) as part of the
component's public contract — they are an API surface bound by the same
compatibility expectations as props/emits (`GEN-PRIN-30`).

**`VUE-SLOT-03` (SHOULD NOT)** Avoid overly broad slot APIs that expose internal
structure; expose the minimum scoped data callers need so internals stay free to
change.

---

## 12. Performance

See [performance](../06-performance.md). Vue-specific:

**`VUE-PERF-01` (SHOULD)** Always key lists with stable ids (`VUE-A-03`) — index
keys cause incorrect DOM reuse and state bugs, not just slowness.

**`VUE-PERF-02` (SHOULD)** Lazy-load routes/heavy components with
`defineAsyncComponent`/dynamic `import()` and route-level code splitting to shrink
the initial bundle.

**`VUE-PERF-03` (CONSIDER)** Use `v-once` for static content rendered once, and
`v-memo` to skip re-rendering subtrees whose dependencies haven't changed — only in
measured hot lists (`GEN-PERF-01`); don't sprinkle prematurely.

**`VUE-PERF-04` (SHOULD)** Avoid creating new objects/arrays/functions inline in
templates on every render where it causes excess re-renders; hoist constants and
use `computed`. Use `shallowRef`/`shallowReactive` for large, externally-managed
data structures to avoid deep-reactivity overhead.

**`VUE-PERF-05` (CONSIDER)** Use `<KeepAlive>` to cache expensive component
instances across toggles, and virtual scrolling for very long lists.

---

## 13. Security

See [security](../04-security.md) and
[typescript.md](typescript.md) §19. Vue-specific:

**`VUE-SEC-01` (MUST NOT)** Never pass untrusted/user content to `v-html` — Vue
auto-escapes `{{ }}` interpolation (safe), but `v-html` renders raw HTML (XSS). If
unavoidable, sanitize with DOMPurify first.

**`VUE-SEC-02` (MUST)** Never bind untrusted data to dangerous attributes/URLs
(`:href`, `:src`) without validation — block `javascript:` URLs; validate schemes.

**`VUE-SEC-03` (SHOULD)** Validate all external data at the boundary with a schema
(Zod) and type it (`TS-ANY-03`); don't trust API shapes. Keep secrets out of
client bundles (anything shipped to the browser is public).

**`VUE-SEC-04` (SHOULD)** Don't render dynamic component names/templates from
untrusted input (`<component :is>` with user-controlled values) — restrict to an
allowlist.

---

## 13a. Forms

**`VUE-FORM-01` (MUST)** Treat all form data as untrusted. Client-side validation is
for UX only; the server must always re-validate (`GEN-SEC-05`). Never assume the
browser enforced your constraints.

**`VUE-FORM-02` (SHOULD)** Validate on submit and at meaningful interaction points
(blur/change), and surface clear, accessible error messages tied to their fields
(see §13b). Validate against a schema (Zod) and map to a typed domain model.

**`VUE-FORM-03` (SHOULD)** Preserve user input when validation fails — don't clear
the form. Keep the raw form model separate from the validated/normalized domain
object you submit.

---

## 13b. Accessibility

**`VUE-A11Y-01` (SHOULD)** Use semantic HTML first (`<button>`, `<nav>`, `<label>`,
`<table>`) before reaching for ARIA — native elements bring focus, keyboard, and
role behavior for free. ARIA supplements semantics; it does not replace them.

**`VUE-A11Y-02` (SHOULD)** Associate every form control with a `<label>` (via `for`/
`id` or wrapping), and give icon-only buttons an accessible name
(`aria-label`/visually-hidden text).

**`VUE-A11Y-03` (SHOULD)** Ensure full keyboard operability (tab order, `Enter`/
`Space` activation, visible focus) and manage focus on route changes and when
opening/closing dialogs (move focus in, trap it, restore it on close).

**`VUE-A11Y-04` (CONSIDER)** Prefer screen-reader-oriented queries (roles, labels)
in component tests so accessibility is verified as part of behavior.

---

## 14. Testing

See [testing](../05-testing.md). Vue-specific:

**`VUE-TEST-01` (SHOULD)** Use **Vitest** + **Vue Test Utils** and/or **Testing
Library (Vue)**; test components through rendered behavior and user interaction,
not internal state (`GEN-TEST-05`).

**`VUE-TEST-02` (SHOULD)** Unit-test composables and Pinia stores in isolation
(they're plain functions) — fast and high-value. Use `createTestingPinia` for
store-dependent components.

**`VUE-TEST-03` (SHOULD)** Use **Playwright/Cypress** for a few critical-path E2E
flows (`GEN-TEST-14`). Mock the network at the boundary (MSW) for integration
tests.

---

## 15. Anti-patterns

- Single-word component names; abbreviated names.
- `v-if` + `v-for` on the same element; missing/index `:key`.
- Untyped/`['prop']`-array prop definitions in committed code.
- Mutating props or `reactive` state from a child; destructuring `reactive`/props
  and losing reactivity.
- Complex logic in templates instead of `computed`/methods; giant computed props.
- Using `watch` to derive what a `computed` should express; side effects/async in
  `computed`.
- Global Pinia state for everything; mutating store state outside actions; losing
  reactivity by destructuring a store without `storeToRefs`.
- Unscoped styles leaking globally.
- `v-html` / dynamic `<component :is>` / `:href` with untrusted input.
- Reaching into `$refs`/`$parent`/child internals instead of props/emits/slots.
- Not cleaning up listeners/timers in `onUnmounted` (leaks).
- Mixing directive shorthand and longhand; multi-attribute elements on one line.
- Relying on client-side form validation alone; clearing user input on validation
  error.
- Div/span "buttons" without roles/keyboard support; unlabeled controls; icon-only
  buttons with no accessible name; broken focus management in dialogs/routes.
- Overly broad scoped-slot APIs that leak internal structure.

---

## Quick checklist

- [ ] Vite + `eslint-plugin-vue` + Prettier + `vue-tsc --noEmit`; "Vue - Official"
      (Volar) in editor; TS `strict`.
- [ ] **A rules:** multi-word names; detailed (typed) props; keyed `v-for` (stable
      id); no `v-if`+`v-for` together; scoped styles.
- [ ] **B rules:** one component/file; consistent PascalCase filenames; Base/parent
      prefixes; general→specific naming; self-closing; PascalCase in templates; full
      words; camelCase props; one attribute/line; simple template expressions; split
      computeds; quoted attrs; consistent directive shorthands.
- [ ] `<script setup lang="ts">`; consistent block order; code grouped by concern.
- [ ] `ref` as default; no destructuring `reactive`/props without `toRefs`; one-way
      data flow (props read-only).
- [ ] Type-based `defineProps`/`defineEmits` + `withDefaults`; `defineModel` for
      `v-model`; literal-union prop types.
- [ ] `computed` (pure) for derived state; `watch`/`watchEffect` for side effects
      with cleanup; teardown in `onUnmounted`.
- [ ] Reusable logic in focused composables (not mixins) with their own cleanup.
- [ ] Pinia setup-stores for shared state; mutate via actions; `storeToRefs`;
      local state stays local; server cache in a query lib.
- [ ] `v-show` vs `v-if` chosen by toggle frequency; loose coupling via
      props/emits/slots.
- [ ] Keyed lists; lazy/code-split heavy routes/components; `shallowRef` for big
      data; `v-memo`/`KeepAlive`/virtual scroll where measured.
- [ ] No `v-html`/dynamic component/`:href` with untrusted input; schema-validate
      external data; no secrets in the bundle.
- [ ] Named, minimally-scoped slots; slot props documented as contract.
- [ ] Forms: server re-validates; accessible per-field errors; user input preserved
      on failure.
- [ ] Accessibility: semantic HTML; labeled controls; keyboard operable; focus
      managed on dialogs/route changes.
- [ ] Vitest + VTU/Testing-Library behavior tests; isolated composable/store tests;
      few Playwright E2E flows.

---

## References

- Vue Style Guide (A–D) — https://vuejs.org/style-guide/
- Vue Style Guide, Priority A (Essential) — https://vuejs.org/style-guide/rules-essential.html
- Vue Style Guide, Priority B (Strongly Recommended) — https://vuejs.org/style-guide/rules-strongly-recommended.html
- Composition API & `<script setup>` — https://vuejs.org/api/sfc-script-setup.html
- Reactivity fundamentals (`ref`/`reactive`) — https://vuejs.org/guide/essentials/reactivity-fundamentals.html
- Composables — https://vuejs.org/guide/reusability/composables.html
- Pinia — https://pinia.vuejs.org/
- Vue Security guide — https://vuejs.org/guide/best-practices/security.html
- Performance guide — https://vuejs.org/guide/best-practices/performance.html
- eslint-plugin-vue — https://eslint.vuejs.org/
