# Vue 3 Review Checklist

> Use with [`../languages/vue.md`](../languages/vue.md),
> [`../languages/typescript.md`](../languages/typescript.md), and
> [`general.md`](general.md).

- [ ] Vite + `eslint-plugin-vue` + Prettier + `vue-tsc --noEmit`; TS `strict`
      (`VUE-TOOL-*`).
- [ ] `<script setup lang="ts">`; consistent SFC block order (`VUE-SFC-*`).
- [ ] Multi-word component names; typed `defineProps`/`defineEmits` (+
      `withDefaults`); `defineModel` for `v-model` (`VUE-A-*`, `VUE-PROP-*`).
- [ ] Props not mutated by children (one-way data flow); `ref` default, `reactive`
      only for cohesive never-reassigned objects; no destructuring that loses
      reactivity (`VUE-RX-*`).
- [ ] Derived state via `computed` (pure); `watch`/`watchEffect` for side effects
      with cleanup; teardown in `onUnmounted` (`VUE-CW-*`).
- [ ] Reusable logic in focused composables (not mixins) with cleanup
      (`VUE-COMP-*`).
- [ ] Pinia setup-stores; mutate via actions; `storeToRefs`; server cache in a
      query lib (`VUE-STATE-*`).
- [ ] `v-for` has stable keys; `v-if`+`v-for` not combined; simple template
      expressions (`VUE-A-03`, `VUE-TPL-*`).
- [ ] Slots: named, minimally scoped; slot props documented as contract
      (`VUE-SLOT-*`).
- [ ] No `v-html`/dynamic `<component :is>`/`:href` with untrusted input;
      schema-validate external data; no secrets in bundle (`VUE-SEC-*`).
- [ ] Forms: server re-validates (client validation is UX only); accessible
      per-field errors; user input preserved on failure (`VUE-FORM-*`).
- [ ] Accessibility: semantic HTML; labeled controls; keyboard operable; focus
      managed on dialogs/route changes (`VUE-A11Y-*`).
- [ ] Component tests verify rendered behavior & emitted events (VTU/Testing
      Library); few Playwright E2E flows (`VUE-TEST-*`).
