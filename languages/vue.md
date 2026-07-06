# Vue 3 SFC (`.vue`) — Coding Standards & State-of-the-Art Practices

> **Scope:** Vue 3 Single-File Components with `<script setup>` + TypeScript +
> Composition API + Vite + Pinia.
> **Primary source:** the official Vue.js Style Guide (Priority A–D). The official
> A and B rules quoted here are verified against vuejs.org.
> **Relationship to general docs:** extends [general docs](../00-index.md) and builds
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
12a. [Server-side rendering (SSR) and hydration](#12a-server-side-rendering-ssr-and-hydration)
13. [Security](#13-security)
13a. [Forms](#13a-forms)
13b. [Accessibility](#13b-accessibility)
13c. [Internationalization (i18n)](#13c-internationalization-i18n)
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

**`VUE-A-04` (MUST)** **Never use `v-if` on the same element as `v-for`.** In
Vue 3, `v-if` has the **higher** priority and is evaluated first, so the
condition cannot access variables from the `v-for` scope —
`v-if="user.isActive"` references a `user` that does not exist yet (an error).
To filter a list, use a `computed` (e.g. `activeUsers`); to conditionally skip
items where the condition needs the loop variable, put `v-for` on a wrapping
`<template>` and `v-if` on the inner element; to hide the whole list, move the
`v-if` to a container element.

```vue
<!-- Bad -->
<li v-for="user in users" v-if="user.isActive" :key="user.id">{{ user.name }}</li>

<!-- Good: filter in a computed -->
<li v-for="user in activeUsers" :key="user.id">{{ user.name }}</li>

<!-- Good: condition needs the loop variable -->
<template v-for="user in users" :key="user.id">
  <li v-if="user.isActive">{{ user.name }}</li>
</template>
```

> **Note (Vue 2 → Vue 3):** the precedence **flipped**. In Vue 2, `v-for` had the
> higher priority — the `v-if` could see the loop variable and silently
> re-evaluated per iteration (wasteful but working). In Vue 3, `v-if` runs first
> and cannot see it (broken outright). Code relying on either implicit ordering
> breaks on migration — one more reason to never combine them.

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

**`VUE-SFC-04` (SHOULD NOT)** Avoid `:deep()` selectors that reach into a child
component's internal DOM (e.g. `.card :deep(.child-internal-class)`). This
couples the parent's stylesheet to the child's private markup — any child
refactor silently breaks the parent's styling, and neither side is protected by
`scoped` anymore. Instead, have the child expose an explicit theming API: CSS
custom properties, props/variant classes, or slots the parent fills with its own
(scoped-styled) markup. Reserve `:deep()` for third-party components that offer
no other styling hook, and comment why it is needed.

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

**`VUE-RX-02` (SHOULD)** Don't destructure a **`reactive()` object** — the
destructured bindings are plain values disconnected from the source; use
`toRefs`/`toRef` or keep accessing via the object. For **props**, the behavior is
version-dependent:

- **Vue 3.5+** — reactive props destructure is **stable and enabled by
  default**: destructuring `defineProps()` keeps reactivity, because the
  compiler rewrites accesses to destructured variables into `props.x` within the
  same `<script setup>` block. JavaScript default-value syntax
  (`const { count = 0 } = defineProps<{ count?: number }>()`) replaces
  `withDefaults`.
- **Pre-3.5** (or with the destructure transform disabled) — destructured props
  are plain constants captured once and never update. Don't destructure; access
  `props.x` or use `toRefs`.

Even in 3.5+, a destructured prop passed **into a function** is passed by value.
When handing one to `watch()` or a composable, wrap it in a getter so the callee
receives a reactive source (the compiler warns on the bare form):

```ts
const { foo } = defineProps<{ foo: string }>()

watch(() => foo, () => { /* ... */ })  // ✅ getter — changes tracked
useComposable(() => foo)               // ✅ callee unwraps with toValue()
watch(foo, () => { /* ... */ })        // ❌ plain value — compiler warning
```

Be explicit about which behavior the project's minimum Vue version guarantees,
and apply one style consistently.

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
On Vue 3.5+, reactive props destructure with native default values
(`const { count = 0 } = defineProps<…>()`) is an equally valid alternative to
`withDefaults` — see `VUE-RX-02`; pick one style per project.

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

**`VUE-CW-05` (SHOULD NOT)** Don't trigger one watcher from another watcher's
callback — watcher A mutates state that watcher B watches, which mutates state
watcher C watches. Cascading watchers form an implicit, hard-to-trace update
chain: the data flow is invisible at any single call site, intermediate states
are observable, re-runs multiply, and ordering/infinite-loop bugs appear.
Restructure instead: derive values with `computed` (chains of pure derivation
are explicit and safe), or merge the logic into a single watcher with multiple
explicit sources (`watch([a, b], …)`).

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

**`VUE-COMP-03` (SHOULD)** Accept flexible reactive inputs as
`MaybeRefOrGetter<T>` (plain value, ref, or getter) and unwrap with `toValue()`
**inside** the tracking effect (`watchEffect`, `computed`, or a watcher getter).
Unwrapping outside the effect reads the value once — subsequent changes are not
tracked and the composable silently goes stale.

```ts
import { toValue, watchEffect, type MaybeRefOrGetter } from 'vue'

export function useTitle(title: MaybeRefOrGetter<string>) {
  watchEffect(() => {
    document.title = toValue(title) // read inside the effect → tracked
  })
}
```

**`VUE-COMP-04` (MUST)** Call composables **synchronously** at the top level of
`setup()`/`<script setup>` (or inside lifecycle hooks). Vue associates lifecycle
registrations and `provide`/`inject` with the *currently active component
instance*, which is only set during synchronous setup — this is instance-based,
**not** call-order-based like React hooks. Calls in async continuations, timers,
or event handlers run with no active instance: their `onMounted`/`onUnmounted`
registrations and injections fail (with a warning at best, a silent leak at
worst). The one exception: `<script setup>` restores the instance context after
`await`; a plain `setup()` function does not.

**`VUE-COMP-05` (SHOULD)** Design async composables to *return* reactive result
state — `{ data, error, isLoading }`-shaped refs — rather than returning a bare
promise and assuming every caller `catch`es rejections. Callers bind the refs
directly in templates; failures become renderable state instead of unhandled
promise rejections, and loading UX is uniform across call sites.

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

**`VUE-STATE-05` (MUST)** In setup stores, **return every state ref** from the
setup function. Pinia registers only returned refs as `$state`; an unreturned
"private" ref is invisible to SSR state serialization/hydration, devtools, and
plugins (persistence, reset) — its value silently desyncs between server and
client. If encapsulation is wanted, return the ref anyway and expose a `computed`
view for consumers, or move the private logic into a plain composable used by
the store.

**`VUE-STATE-06` (MUST NOT)** Don't wrap the refs a setup store returns in
`readonly()`. The returned refs **are** the store's `$state`; a readonly wrapper
means Pinia itself can no longer write into them, breaking `$patch`, SSR
hydration, and state-writing plugins. Enforce mutate-through-actions
(`VUE-STATE-03`) by convention and review, or expose separate readonly
`computed` views alongside the (returned, writable) state refs.

**`VUE-STATE-07` (SHOULD NOT)** Don't store or return request-scoped framework
objects — `useRoute()`/`useRouter()` results, request headers, other per-request
composables or injections — as store state. Pinia explicitly warns against
returning them from setup stores; captured request-scoped objects can leak
across concurrent requests (and users) under SSR, and they don't belong to the
store's state anyway. Call `useRoute()`/`useRouter()` at the component or
action call site where they're needed.

**`VUE-STATE-08` (MUST)** Don't call the built-in `$reset()` on a setup store —
Pinia implements it only for option stores (where `state()` can be re-invoked);
on a setup store it **throws at runtime**. Define an explicit `$reset` action
that reassigns each state ref to its initial value (or install a plugin that
provides one) and return it from the store.

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

**`VUE-PERF-06` (SHOULD)** Wrap non-reactive third-party class instances — map,
chart, and editor SDK objects (a MapLibre/Leaflet map, a Chart.js chart, an
editor instance) — in `markRaw()` before storing them in refs, `reactive` state,
or Pinia stores, and hold them in a `shallowRef` (`VUE-PERF-04`). Deep-proxying
such instances is pure overhead at best; instances with circular internals, DOM
references, or identity-sensitive checks (`this === instance`) break outright
when accessed through a reactive Proxy.

```ts
import { markRaw, shallowRef } from 'vue'

const map = shallowRef<maplibregl.Map>()
onMounted(() => {
  map.value = markRaw(new maplibregl.Map({ container: el.value! }))
})
```

---

## 12a. Server-side rendering (SSR) and hydration

Applies whenever components render on the server (Vue SSR, Nuxt, or another
meta-framework). During hydration the client re-renders and must produce the
**same HTML** the server sent; mismatches cause hydration warnings, full
client re-renders, and subtle DOM corruption.

**`VUE-SSR-01` (MUST)** Keep nondeterministic values out of render-driving code:
`Date.now()`, `Math.random()`, and ambient locale/timezone-dependent formatting
(`toLocaleString()` without an explicit locale) produce different output on the
server and the client → hydration mismatch. Compute such values in `onMounted`
(client-only, after hydration) or use SSR-stable inputs: pass an explicit
locale/timezone, snapshot the time once on the server and transfer it, seed any
randomness deterministically.

**`VUE-SSR-02` (SHOULD)** Use `useId()` (Vue 3.5+) to generate unique element IDs
(form control/`<label for>` pairs, `aria-describedby` targets). It is stable
across server and client renders — unlike module-level counters, which advance
independently in each environment and desynchronize the IDs.

**`VUE-SSR-03` (SHOULD)** Treat `<Teleport>` specially under SSR: teleported
content is **not** rendered in place in the server output, so the teleport
target must be handled by the framework or the teleport deferred to the client.
Meta-frameworks support specific targets only — Nuxt SSR-renders teleports to
the `#teleports` target; other targets need a client-only wrapper (e.g.
`<ClientOnly>`) or a client-conditional `disabled`/mount. Keep SEO-critical
content in the normal document flow, never inside a teleport.

**`VUE-SSR-04` (MUST)** Put DOM-touching side effects and browser-only API access
(`window`, `document`, `localStorage`, `navigator`, element measurement) in
`onMounted`/`onBeforeMount` — which never run on the server — or behind explicit
client-only guards. `setup()`/`<script setup>` executes on the server too, where
none of these globals exist; an unguarded access is a `ReferenceError` that
aborts server rendering.

### SSR meta-frameworks (Nuxt)

Nuxt-specific rules — apply only when the project uses Nuxt.

**`VUE-NUXT-01` (MUST)** Fetch data needed during SSR with `useFetch`/
`useAsyncData`: they run on the server, transfer the result to the client in the
payload, and skip the duplicate client fetch. A bare `$fetch` at the top level
of `setup` executes on the server **and again** on the client (double-fetch, no
payload transfer, possible state mismatch). Reserve `$fetch` for user-triggered
actions (event handlers, form submits) and for calls inside a `useAsyncData`
handler.

**`VUE-NUXT-02` (SHOULD)** Give `useAsyncData`/`useFetch` explicit, stable keys.
Auto-generated keys are derived from file and line: opaque in devtools and
payloads, invalidated by unrelated refactors (moving the call changes the key),
and unable to share cached data between call sites that intentionally request
the same thing.

**`VUE-NUXT-03` (MUST)** In Nuxt 3 API routes (`server/`), signal failures with
`throw createError({ statusCode, statusMessage })`. Keep `statusMessage` short
and **static** — never interpolate dynamic user input into it (reflected-content
risk, HTTP status-line semantics). A `message` passed on an API-route error does
not propagate to the client in Nuxt 3; put structured client-facing detail in
the `data` property instead.

> **Note:** h3 v2 moves toward carrying the error message in the JSON body —
> revisit this rule on the next major Nuxt/h3 upgrade.

Type checking: `vue-tsc --noEmit` in CI is already mandated by `VUE-TOOL-01`; it
applies unchanged to Nuxt projects (`nuxi typecheck` runs it).

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

## 13c. Internationalization (i18n)

Applies to applications with localization requirements (e.g. `vue-i18n`).

**`VUE-I18N-01` (SHOULD)** Route **all** user-facing strings through the i18n
layer — including the strings a visible-copy review misses: `aria-label`s,
tooltips, toast/snackbar messages, `placeholder`s, validation messages, `alt`
text, and `document.title`. Keep locale files complete in CI: verify that every
key exists in every locale (missing-key check) so untranslated strings never
ship silently.

**`VUE-I18N-02` (MUST NOT)** Never build plural or composed phrases by string
concatenation (`count + ' ' + noun`, fragments joined around variables). Plural
rules, word order, and grammatical agreement differ per language — English's
two plural forms are the exception, not the rule. Use the i18n library's
pluralization (message with plural forms selected by `count`) and named
interpolation parameters, so translators control the full sentence.

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
- Mutating props or `reactive` state from a child; destructuring `reactive` (or
  props pre-3.5) and losing reactivity; passing a destructured prop to
  `watch`/composables without a getter.
- Complex logic in templates instead of `computed`/methods; giant computed props.
- Using `watch` to derive what a `computed` should express; side effects/async in
  `computed`; watchers triggering other watchers (cascading update chains).
- Composables called after `await` (outside `<script setup>`), in timers, or in
  event handlers; composables that accept only raw values instead of
  `MaybeRefOrGetter` + `toValue()`.
- Global Pinia state for everything; mutating store state outside actions; losing
  reactivity by destructuring a store without `storeToRefs`.
- Setup stores with unreturned "private" state refs or `readonly()`-wrapped
  returned refs; router/route or other request-scoped objects kept in store
  state; calling the built-in `$reset()` on a setup store.
- Unscoped styles leaking globally; `:deep()` reaching into a child component's
  internals instead of a CSS-custom-property/prop/slot theming API.
- Third-party SDK instances (maps, charts, editors) stored in reactive state
  without `markRaw()` + `shallowRef`.
- `Date.now()`/`Math.random()`/ambient locale formatting in SSR-rendered output
  (hydration mismatch); browser APIs touched in `setup` (SSR crash);
  SEO-critical content inside a `<Teleport>`.
- Nuxt: bare `$fetch` at setup top level (double-fetch); keyless `useAsyncData`;
  dynamic user input in `createError`'s `statusMessage`.
- Hardcoded user-facing strings bypassing i18n (aria-labels, toasts,
  placeholders, validation messages); `count + noun` concatenation instead of
  i18n pluralization.
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
- [ ] `<script setup lang="ts">`; consistent block order; code grouped by concern;
      no `:deep()` into child internals (theming via CSS vars/props/slots).
- [ ] `ref` as default; no destructuring `reactive` (or props pre-3.5); destructured
      props getter-wrapped for `watch`/composables; one-way data flow (props
      read-only).
- [ ] Type-based `defineProps`/`defineEmits` + `withDefaults`; `defineModel` for
      `v-model`; literal-union prop types.
- [ ] `computed` (pure) for derived state; `watch`/`watchEffect` for side effects
      with cleanup; no watcher-triggers-watcher cascades; teardown in `onUnmounted`.
- [ ] Reusable logic in focused composables (not mixins) with their own cleanup;
      called synchronously in setup; `MaybeRefOrGetter` inputs + `toValue()` inside
      effects; async composables return `{ data, error, isLoading }`.
- [ ] Pinia setup-stores for shared state; mutate via actions; `storeToRefs`;
      all state refs returned (not `readonly()`-wrapped); no route/router in state;
      custom `$reset` for setup stores; local state stays local; server cache in a
      query lib.
- [ ] `v-show` vs `v-if` chosen by toggle frequency; loose coupling via
      props/emits/slots.
- [ ] Keyed lists; lazy/code-split heavy routes/components; `shallowRef` for big
      data; `markRaw` for third-party SDK instances; `v-memo`/`KeepAlive`/virtual
      scroll where measured.
- [ ] SSR: deterministic render output (no `Date.now()`/`Math.random()`/ambient
      locale); `useId()` for element IDs; teleports SSR-handled or client-only;
      browser APIs only in `onMounted`/client guards.
- [ ] Nuxt: `useFetch`/`useAsyncData` with explicit keys for SSR data ($fetch only
      for user actions); `createError` with static `statusMessage`, detail in
      `data`.
- [ ] No `v-html`/dynamic component/`:href` with untrusted input; schema-validate
      external data; no secrets in the bundle.
- [ ] Named, minimally-scoped slots; slot props documented as contract.
- [ ] Forms: server re-validates; accessible per-field errors; user input preserved
      on failure.
- [ ] Accessibility: semantic HTML; labeled controls; keyboard operable; focus
      managed on dialogs/route changes.
- [ ] i18n (when localized): all user-facing strings — incl. aria-labels, tooltips,
      toasts, placeholders, validation — through i18n; locale files complete in CI;
      i18n pluralization, no `count + noun` concatenation.
- [ ] Vitest + VTU/Testing-Library behavior tests; isolated composable/store tests;
      few Playwright E2E flows.

---

## References

- Vue Style Guide (A–D) — https://vuejs.org/style-guide/
- Vue Style Guide, Priority A (Essential) — https://vuejs.org/style-guide/rules-essential.html
- Vue Style Guide, Priority B (Strongly Recommended) — https://vuejs.org/style-guide/rules-strongly-recommended.html
- Composition API & `<script setup>` — https://vuejs.org/api/sfc-script-setup.html
- Reactivity fundamentals (`ref`/`reactive`) — https://vuejs.org/guide/essentials/reactivity-fundamentals.html
- List rendering (`v-for` with `v-if` precedence) — https://vuejs.org/guide/essentials/list.html
- Props (Reactive Props Destructure, 3.5+) — https://vuejs.org/guide/components/props.html
- Composables — https://vuejs.org/guide/reusability/composables.html
- Server-Side Rendering guide (hydration) — https://vuejs.org/guide/scaling-up/ssr.html
- Pinia — https://pinia.vuejs.org/
- Pinia core concepts (setup stores, state) — https://pinia.vuejs.org/core-concepts/
- Vue Security guide — https://vuejs.org/guide/best-practices/security.html
- Performance guide — https://vuejs.org/guide/best-practices/performance.html
- Nuxt data fetching — https://nuxt.com/docs/getting-started/data-fetching
- Nuxt error handling — https://nuxt.com/docs/getting-started/error-handling
- Vue I18n — https://vue-i18n.intlify.dev/
- eslint-plugin-vue — https://eslint.vuejs.org/
