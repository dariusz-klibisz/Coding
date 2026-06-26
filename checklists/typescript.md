# TypeScript Review Checklist

> Use with [`../languages/typescript.md`](../languages/typescript.md) and
> [`general.md`](general.md).

- [ ] Strict `tsconfig` respected (`strict`, `noUncheckedIndexedAccess`, …); ESLint
      (type-aware) + Prettier + `tsc --noEmit` in CI (`TS-CFG-*`, `TS-TOOL-*`).
- [ ] No `any` (explicit or implicit); use `unknown` + narrowing; validate external
      data with a schema (Zod) and infer types (`TS-ANY-*`).
- [ ] Null/undefined narrowed; `?.`/`??`; minimal `!` non-null assertions
      (`TS-NULL-*`).
- [ ] Promises awaited/handled; `Promise.all` for independent work; no floating
      promises (`TS-ASYNC-*`).
- [ ] Caught errors treated as `unknown` and narrowed; throw `Error` subclasses;
      preserve `{ cause }` on rethrow (`TS-ERR-*`).
- [ ] Discriminated unions for variant state; exhaustive switches via `never`
      (`TS-UNION-*`).
- [ ] Type assertions avoided unless justified; prefer `satisfies` to
      validate-without-widening; no `as any`/double-casts (`TS-CAST-*`).
- [ ] Enums used only when a runtime object/namespaced constants are genuinely
      needed (prefer string-literal unions; string enums if needed) (`TS-ENUM-01`).
- [ ] JS footguns avoided: no `new Array`/`new Object`; numeric `sort` has compare
      fn; `Object.keys`/`Object.hasOwn` over `for...in`/`hasOwnProperty`; no
      `export let` (`TS-OBJ-*`).
- [ ] Public exports minimal/stable; named exports; `import type`; no circular deps
      (`TS-MOD-*`).
- [ ] No `eval`/`innerHTML` on untrusted data; CSPRNG for tokens; audited/locked
      deps (`TS-SEC-*`).
- [ ] Tests in TS (type-checked); behavior-focused; fast-check / type-tests for
      libraries (`TS-TEST-*`).
