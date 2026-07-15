# Vue 3 Review Checklist

> Use with [`../languages/vue.md`](../languages/vue.md),
> [`../languages/typescript.md`](../languages/typescript.md), and
> [`general.md`](general.md).

- [ ] Vite + `eslint-plugin-vue` + Prettier + `vue-tsc --noEmit`; TS `strict`
      (`VUE-TOOL-*`).
- [ ] `<script setup lang="ts">`; consistent SFC block order; no `:deep()` into
      child-component internals (`VUE-SFC-*`).
- [ ] Multi-word component names; typed `defineProps`/`defineEmits` (+
      `withDefaults`); `defineModel` for `v-model` (`VUE-A-*`, `VUE-PROP-*`).
- [ ] Props not mutated by children (one-way data flow); `ref` default, `reactive`
      only for cohesive never-reassigned objects; no destructuring that loses
      reactivity (props destructure OK on 3.5+; getter-wrap when passed to
      `watch`/composables) (`VUE-RX-*`).
- [ ] Derived state via `computed` (pure); `watch`/`watchEffect` for side effects
      with cleanup; no watcher-triggers-watcher cascades; teardown in `onUnmounted`
      (`VUE-CW-*`).
- [ ] Reusable logic in focused composables (not mixins) with cleanup; called
      synchronously in setup; `MaybeRefOrGetter` + `toValue()` inside effects;
      async ones return `{ data, error, isLoading }` (`VUE-COMP-*`).
- [ ] Pinia setup-stores; mutate via actions; `storeToRefs`; all state refs
      returned, not `readonly()`-wrapped; no route/router in store state; custom
      `$reset`; server cache in a query lib (`VUE-STATE-*`).
- [ ] `v-for` has stable keys; `v-if`+`v-for` not combined; simple template
      expressions (`VUE-A-03`, `VUE-TPL-*`).
- [ ] Slots: named, minimally scoped; slot props documented as contract
      (`VUE-SLOT-*`).
- [ ] No `v-html`/dynamic `<component :is>`/`:href` with untrusted input;
      schema-validate external data; no secrets in bundle (`VUE-SEC-*`).
- [ ] Forms: server re-validates (client validation is UX only); accessible
      per-field errors; user input preserved on failure (`VUE-FORM-*`).
- [ ] Performance: third-party SDK instances `markRaw()` + `shallowRef`; big data
      shallow-reactive (`VUE-PERF-*`).
- [ ] SSR: deterministic render output (no `Date.now()`/`Math.random()`/ambient
      locale); `useId()` for element IDs; teleports SSR-handled or client-only;
      browser APIs only in `onMounted`/client guards (`VUE-SSR-*`).
- [ ] Nuxt: SSR data via `useFetch`/`useAsyncData` with explicit keys (`$fetch`
      only for user actions); `createError` with static `statusMessage`, detail in
      `data` (`VUE-NUXT-*`).
- [ ] Accessibility: semantic HTML; labeled controls; keyboard operable; focus
      managed on dialogs/route changes (`VUE-A11Y-*`).
- [ ] i18n (when localized): all user-facing strings — incl. aria-labels, toasts,
      placeholders, validation — through i18n; locale files complete in CI; i18n
      pluralization, never `count + noun` (`VUE-I18N-*`).
- [ ] Component tests verify rendered behavior & emitted events (VTU/Testing
      Library); few Playwright E2E flows (`VUE-TEST-*`).
