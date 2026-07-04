# TypeScript (`.ts`) — Coding Standards & State-of-the-Art Practices

> **Scope:** Modern TypeScript (5.x) for application and library code (Node, Deno,
> Bun, browser). Vue-specific TS guidance lives in
> [vue.md](vue.md); this file covers the language itself.
> **Primary sources:** TypeScript Handbook, TypeScript-ESLint, common community
> standards.
> **Relationship to general docs:** extends [general docs](../00-index.md). Where a
> general rule and a TS rule conflict, the TS rule wins for TS.

**Rule ID prefix:** `TS`

---

## Table of contents

1. [Tooling & enforcement](#1-tooling--enforcement)
2. [Strict compiler configuration](#2-strict-compiler-configuration)
3. [Naming conventions](#3-naming-conventions)
4. [Formatting](#4-formatting)
5. [The `any` problem and `unknown`](#5-the-any-problem-and-unknown)
6. [Type vs interface](#6-type-vs-interface)
7. [Type narrowing & guards](#7-type-narrowing--guards)
8. [Discriminated unions & exhaustiveness](#8-discriminated-unions--exhaustiveness)
9. [Generics](#9-generics)
10. [Immutability & readonly](#10-immutability--readonly)
11. [Null & undefined](#11-null--undefined)
12. [Functions](#12-functions)
13. [Classes vs functions](#13-classes-vs-functions)
14. [Modules & imports](#14-modules--imports)
15. [Async & promises](#15-async--promises)
16. [Error handling](#16-error-handling)
17. [Type assertions & casting](#17-type-assertions--casting)
18. [Enums](#18-enums)
18a. [Objects & arrays (JS footguns)](#18a-objects--arrays-js-footguns)
19. [Security](#19-security)
20. [Testing](#20-testing)
21. [Anti-patterns](#21-anti-patterns)
22. [Quick checklist](#quick-checklist)
23. [References](#references)

---

## 1. Tooling & enforcement

**`TS-TOOL-01` (MUST)** Use **ESLint** with `typescript-eslint` (type-aware rules
via `parserOptions.project`) and **Prettier** for formatting. ESLint catches bugs;
Prettier ends style debate. Run both in CI and pre-commit.

**`TS-TOOL-02` (MUST)** Treat `tsc --noEmit` type-checking as a required CI gate
(ESLint alone doesn't fully type-check). Build with the type checker; never ship
type errors.

**`TS-TOOL-03` (SHOULD)** Recommended ESLint configs: `eslint:recommended`,
`typescript-eslint` recommended + `recommended-type-checked` (and `strict-type-
checked` for new projects). Enable `no-floating-promises`, `no-explicit-any`,
`no-unsafe-*`, `switch-exhaustiveness-check`.

---

## 2. Strict compiler configuration

**`TS-CFG-01` (MUST)** Enable **`"strict": true`** in `tsconfig.json`. This turns
on `strictNullChecks`, `noImplicitAny`, `strictFunctionTypes`,
`strictBindCallApply`, `useUnknownInCatchVariables`, and more.

**Why:** Strict mode is where TypeScript delivers its value. Without
`strictNullChecks`, `null`/`undefined` are assignable to everything and the
compiler can't catch the most common runtime error class. Without
`noImplicitAny`, untyped values silently become `any`, defeating the type system.
A non-strict TS codebase has the maintenance cost of types without most of the
safety.

**`TS-CFG-02` (SHOULD)** Additionally enable: `noUncheckedIndexedAccess` (array/
record access yields `T | undefined` — catches out-of-bounds assumptions),
`noImplicitReturns`, `noFallthroughCasesInSwitch`, `noImplicitOverride`,
`exactOptionalPropertyTypes`, `forceConsistentCasingInFileNames`.

**`TS-CFG-03` (SHOULD)** Set a modern `target`/`module`/`moduleResolution`
(`"bundler"` or `"node16"`/`"nodenext"`), `"esModuleInterop": true`, and
`"isolatedModules": true` for compatibility with modern bundlers/transpilers.

---

## 3. Naming conventions

| Construct | Convention | Example |
|-----------|------------|---------|
| Variable, function, method, property | `camelCase` | `itemCount`, `loadUser` |
| Class, interface, type alias, enum, type param | `PascalCase` | `UserService`, `OrderState`, `T`, `TKey` |
| Constant (true compile-time const) | `camelCase` or `UPPER_SNAKE_CASE` | `maxRetries` / `MAX_RETRIES` |
| Enum members | `PascalCase` | `Status.Active` |
| Boolean | `is/has/can/should` prefix | `isLoading`, `hasError` |
| File names | `kebab-case` or `camelCase` (be consistent) | `user-service.ts` |

**`TS-NAME-01` (SHOULD NOT)** Don't prefix interfaces with `I` (`IUser`) — it's a
discouraged C#-ism in the TS community; the type system makes the distinction
unnecessary.

**`TS-NAME-02` (SHOULD)** Don't encode types in names (Hungarian notation); the
type annotation already conveys that. Name by intent/role.

---

## 4. Formatting

**`TS-FMT-01` (SHOULD)** Delegate all formatting to **Prettier** (indentation,
quotes, semicolons, line length, trailing commas). Don't hand-format or argue
style; the Vue style guide explicitly says it doesn't care about semicolons/quotes
— pick a Prettier config and move on.

**`TS-FMT-02` (SHOULD)** Prefer `const`; use `let` only when reassignment is
required; never use `var` (function-scoped, hoisting hazards). Use strict equality
`===`/`!==`, never `==`/`!=` (avoids coercion surprises).

---

## 5. The `any` problem and `unknown`

**`TS-ANY-01` (SHOULD NOT)** Avoid `any`. It is an escape hatch that **disables all
type checking** for that value and silently propagates — one `any` can poison a
whole call chain. Enable `no-explicit-any` and `no-unsafe-*` lint rules.

**`TS-ANY-02` (SHOULD)** Use **`unknown`** instead of `any` for genuinely-unknown
values (e.g. parsed JSON, `catch` variables). `unknown` is type-safe: you must
narrow it before use, so the compiler still protects you.

```ts
// any: no protection — typo compiles, crashes at runtime
function parseAny(json: string): any {
  return JSON.parse(json);
}
parseAny("{}").user.naem.toUpperCase();   // compiles, throws at runtime

// unknown: forces validation before use
function parse(json: string): unknown {
  return JSON.parse(json);
}
const data = parse("{}");
if (isUser(data)) {                        // narrow with a type guard
  data.name.toUpperCase();                 // safe
}
```

**`TS-ANY-03` (SHOULD)** Validate external data (API responses, JSON, form input)
at the boundary with a runtime schema validator (**Zod**, Valibot, io-ts) and
*infer* the static type from the schema. This is "parse, don't validate"
(`GEN-DEF`) for TS — the compile-time type and runtime check stay in sync.

```ts
import { z } from "zod";
const UserSchema = z.object({ id: z.number(), name: z.string() });
type User = z.infer<typeof UserSchema>;          // type derived from schema
const user = UserSchema.parse(await res.json()); // validated at the boundary
```

---

## 6. Type vs interface

Both describe object shapes. Guidance:

**`TS-TYPE-01` (CONSIDER)** Use **`interface`** for object/class shapes that may be
extended or implemented and for public API contracts; use **`type`** for unions,
intersections, tuples, mapped/conditional types, function types, and primitives/
aliases.

| | `interface` | `type` |
|---|-------------|--------|
| Object shapes | ✅ | ✅ |
| Declaration merging (re-open to add members) | ✅ | ❌ |
| Unions / intersections / tuples / primitives | ❌ | ✅ |
| Mapped & conditional types | ❌ | ✅ |
| `extends`/`implements` ergonomics | ✅ | (via `&`) |

**Pros of `interface`:** extendable, better error messages in some cases,
declaration merging (useful for augmenting library types).
**Pros of `type`:** more expressive (unions, conditionals, mapped types).
Pick a default for object shapes and be consistent; reach for `type` when you need
its extra power.

---

## 7. Type narrowing & guards

**`TS-NARROW-01` (SHOULD)** Narrow union types with the language's control-flow
analysis: `typeof`, `instanceof`, `in`, equality checks, and **user-defined type
guards** (`x is T`). Let the compiler track the narrowed type per branch.

```ts
function isString(x: unknown): x is string {
  return typeof x === "string";
}
```

**`TS-NARROW-02` (SHOULD)** Prefer narrowing over assertions (`as`) — narrowing is
*checked* by the compiler/runtime; assertions are unchecked promises that can lie
(§17).

---

## 8. Discriminated unions & exhaustiveness

**`TS-UNION-01` (SHOULD)** Model "one of several shapes" with **discriminated
unions** (a shared literal `kind`/`type` tag) rather than optional fields and
boolean flags. This makes illegal states unrepresentable (`GEN-PRIN-28`).

```ts
type RequestState =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: User }
  | { status: "error"; message: string };
// Can't access .data unless status === "success" — compiler enforces it.
```

**`TS-UNION-02` (SHOULD)** Make `switch` over a discriminated union **exhaustive**
using a `never` check in the default branch, so adding a new variant becomes a
compile error everywhere it must be handled (`GEN-DEF-14`):

```ts
function render(s: RequestState): string {
  switch (s.status) {
    case "idle":    return "";
    case "loading": return "Loading…";
    case "success": return s.data.name;
    case "error":   return s.message;
    default:
      const _exhaustive: never = s;   // compile error if a case is missing
      return _exhaustive;
  }
}
```

---

## 9. Generics

**`TS-GEN-01` (SHOULD)** Use generics to write reusable, type-preserving code
instead of `any` or overloads. Constrain type parameters (`<T extends ...>`) to
the minimum needed so you can safely use their members.

**`TS-GEN-02` (SHOULD)** Keep generics readable: meaningful names for non-trivial
parameters (`TItem`, `TKey`), and don't over-engineer with deeply nested
conditional/mapped types when a simpler signature suffices (KISS, `GEN-PRIN-02`).

**`TS-GEN-03` (CONSIDER)** Leverage built-in utility types (`Partial`, `Required`,
`Readonly`, `Pick`, `Omit`, `Record`, `ReturnType`, `Awaited`, `NonNullable`)
before writing custom type machinery.

---

## 10. Immutability & readonly

**`TS-IMM-01` (SHOULD)** Mark data immutable: `readonly` properties,
`ReadonlyArray<T>`/`readonly T[]`, `Readonly<T>`, and `as const` for literal
constants. Prevents accidental mutation and documents intent (`GEN-PRIN-26`).

```ts
const ROLES = ["admin", "editor", "viewer"] as const;   // readonly tuple
type Role = (typeof ROLES)[number];                     // "admin"|"editor"|"viewer"
```

**`TS-IMM-02` (SHOULD)** Prefer non-mutating array/object operations (`map`,
`filter`, spread) for derived data; `readonly` types are compile-time only (no
runtime cost) but catch real bugs.

---

## 11. Null & undefined

**`TS-NULL-01` (MUST)** With `strictNullChecks`, handle `null`/`undefined`
explicitly. Use optional chaining `?.` and nullish coalescing `??` (not `||`,
which also triggers on `0`/`""`/`false`).

```ts
const port = config.port ?? 8080;   // only when null/undefined (correct)
const bad  = config.port || 8080;   // also overrides a valid 0 (bug)
```

**`TS-NULL-02` (SHOULD)** Prefer modeling absence explicitly. Decide a convention
(`undefined` for "not provided", reserve `null` for "explicitly empty" if needed)
and stay consistent. Avoid the non-null assertion `!` except where you provably
know better than the compiler — and comment why.

**`TS-NULL-03` (MUST)** After coercing external input with `Number(...)` (or
`parseFloat`/`parseInt`), guard with `Number.isFinite` — `NaN` and `Infinity`
survive coercion, serialize to `null` in JSON, and poison every comparison
(`NaN` compares false to everything, so range checks silently pass or fail
wrongly). Check nullable numerics with `x != null`, never truthiness: `if (x)`
silently drops a legitimate `0` — the same falsy-zero hazard as `||` defaults
(`TS-NULL-01`).

```ts
// Bad — NaN flows through; 0 is treated as missing
const qty = Number(req.query.qty);
if (qty) applyQuantity(qty);

// Good — finite check after coercion; null check for absence
const qty = Number(req.query.qty);
if (!Number.isFinite(qty)) throw new BadRequestError("qty must be a number");
if (limit != null && qty > limit) throw new BadRequestError("qty over limit");
```

---

## 12. Functions

**`TS-FUNC-01` (SHOULD)** Annotate function **parameters** and **public/exported
return types** explicitly. Return-type annotations prevent accidental widening/
leaking of internal types and document the contract; rely on inference for trivial
local/internal functions.

**`TS-FUNC-02` (SHOULD)** Prefer small, pure functions; avoid long parameter lists
— use an options object (`GEN-PRIN-22`). Use default and rest parameters over
`arguments`.

**`TS-FUNC-03` (MUST)** Handle promises: never leave a floating promise (enable
`no-floating-promises`). `await` it, `void` it deliberately, or `.catch()` it —
unhandled rejections crash Node and hide errors (§15).

---

## 13. Classes vs functions

**`TS-CLASS-01` (CONSIDER)** TS/JS is multi-paradigm. Prefer plain functions and
modules for stateless logic; use classes when you have encapsulated state +
behavior, need polymorphism, or integrate with class-based frameworks. Don't force
OOP where a function suffices (`GEN-PRIN-12`, composition over inheritance).

**`TS-CLASS-02` (SHOULD)** Use `private`/`protected`/`readonly` and the
`#private` (true runtime-private) fields for encapsulation; parameter properties
for concise constructors. Favor composition/dependency injection over deep
inheritance.

---

## 14. Modules & imports

**`TS-MOD-01` (SHOULD)** Use ES modules (`import`/`export`). Prefer **named
exports** over default exports — named exports are refactor-safe, autocomplete
well, and avoid the "same default imported under different names" inconsistency.

**`TS-MOD-02` (SHOULD)** Use `import type` / `export type` for type-only imports
(elided at compile time; required under `isolatedModules`/`verbatimModuleSyntax`).
Keep imports ordered (ESLint `import/order`).

**`TS-MOD-03` (SHOULD NOT)** Avoid circular dependencies between modules — they
cause `undefined`-at-import-time bugs and complicate bundling. Avoid deep relative
import chains (`../../../`); configure path aliases.

**`TS-MOD-04` (SHOULD NOT)** Avoid namespaces (`namespace`/`module` keyword) for
new code — ES modules are the standard. Avoid side-effectful imports unless
intentional.

**`TS-MOD-05` (SHOULD NOT)** Don't capture `process.env` (or config derived from
it) in module-top-level constants in code that must be testable or reloadable —
the value freezes at first import, defeating per-test overrides and runtime
config reloads. Read env inside functions, or expose an explicit reset seam.
Additionally, coerce env-derived booleans through a typed helper: every env
value is a string, and the string `"false"` is truthy.

```ts
// Bad — frozen at first import; "false" is a truthy string
const DEBUG = process.env.DEBUG || false;

// Good — read at call time; explicit string→boolean coercion
const envBool = (name: string): boolean =>
  ["1", "true", "yes"].includes((process.env[name] ?? "").toLowerCase());
const isDebug = () => envBool("DEBUG");
```

---

## 15. Async & promises

Builds on [concurrency](../07-concurrency.md).

**`TS-ASYNC-01` (SHOULD)** Use `async`/`await` over raw `.then()` chains for
readability; `await` inside `try/catch` for error handling.

**`TS-ASYNC-02` (MUST)** Don't leave floating promises (`TS-FUNC-03`). Don't mix
`await` and unhandled promises.

**`TS-ASYNC-03` (SHOULD)** Run independent async work concurrently with
`Promise.all` (fail-fast) / `Promise.allSettled` (collect all results); don't
`await` in a loop when the iterations are independent (serializes them
needlessly — `GEN-PERF-11`).

```ts
// Serial (slow) — each await blocks the next
for (const id of ids) results.push(await fetchUser(id));

// Concurrent (fast)
const results = await Promise.all(ids.map(fetchUser));
```

**`TS-ASYNC-04` (SHOULD)** Avoid the `async` Promise-executor anti-pattern; don't
`await` non-promises needlessly; propagate `AbortSignal` for cancellation and add
timeouts (`GEN-ERR-19`).

**`TS-ASYNC-05` (MUST)** Check `response.ok` (or the status code) before
consuming a `fetch` response — `fetch` resolves successfully on HTTP 4xx/5xx and
rejects only on network failure. Parsing the body of an error response
propagates garbage downstream, and a destructive operation keyed on an unchecked
response can, e.g., treat "reference list unavailable" as "reference list empty"
and delete everything it should have kept.

```ts
// Bad — a 500 error page is parsed as if it were data
const data = await (await fetch(url)).json();

// Good — status checked before the body is consumed
const res = await fetch(url);
if (!res.ok) throw new Error(`GET ${url} failed: ${res.status}`);
const data = await res.json();
```

---

## 16. Error handling

Builds on [error handling](../03-error-handling.md).

**`TS-ERR-01` (SHOULD)** Throw `Error` (or subclasses), never strings/objects —
only `Error` carries a stack trace. Create domain error subclasses for cases
callers need to distinguish.

**`TS-ERR-02` (SHOULD)** With `useUnknownInCatchVariables`, the `catch` binding is
`unknown` — narrow it (`if (e instanceof Error)`) before use rather than assuming
`.message` exists.

**`TS-ERR-03` (CONSIDER)** For expected, frequent failures, return a `Result<T,E>`
discriminated union instead of throwing (`GEN-ERR-10`) — explicit and type-checked.

**`TS-ERR-04` (SHOULD)** When wrapping/re-throwing, preserve the original error
with the standard **`cause`** option (ES2022) instead of discarding it
(`GEN-ERR-04`): `throw new OrderError("failed to place order", { cause: err })`.
This keeps the original stack/context for diagnosis.

---

## 17. Type assertions & casting

**`TS-CAST-01` (SHOULD NOT)** Avoid type assertions (`x as T`, `<T>x`). An
assertion **overrides** the compiler's judgement without any runtime check — if
you're wrong, it crashes later with no warning. Prefer type guards/narrowing
(`TS-NARROW-01`) and schema validation (`TS-ANY-03`).

**`TS-CAST-02` (MUST NOT)** Never use double assertions (`x as unknown as T`)
except in tightly-scoped, commented, unavoidable interop. Never use `as any`.

**`TS-CAST-03` (SHOULD)** `as const` is **not** a type assertion in the dangerous
sense — it narrows literals to their most specific readonly type and is encouraged
for constants (`TS-IMM-01`).

**`TS-CAST-04` (SHOULD)** Prefer the **`satisfies`** operator (TS 4.9+) over `as`
when you want to check that a value conforms to a type **without widening or
losing** its specific inferred type. `satisfies` validates against the type but
keeps the narrow literal type for downstream inference; `as` discards the
compiler's checking.

```ts
// Bad: `as` silences checking and widens
const routes = { home: "/", about: "/about" } as Record<string, string>;

// Good: `satisfies` verifies the shape but keeps literal keys/values
const routes = {
  home: "/",
  about: "/about",
} satisfies Record<string, string>;
// routes.home is "/" (literal), and a typo key is still a compile error
```

---

## 18. Enums

**`TS-ENUM-01` (CONSIDER)** Prefer **union of string literals** or `as const`
objects over TS `enum` for most cases. Numeric enums have surprising behavior
(reverse mapping, assignable from any number); `const enum` has been problematic
with isolated modules/bundlers.

```ts
// Preferred: string-literal union — simple, erasable, exhaustiveness-friendly
type Direction = "north" | "south" | "east" | "west";

// or an as-const object when you need values + a type
const LogLevel = { Debug: 0, Info: 1, Warn: 2, Error: 3 } as const;
type LogLevel = (typeof LogLevel)[keyof typeof LogLevel];
```

**Pros of literal unions:** zero runtime footprint, great narrowing/exhaustiveness,
no enum quirks.
**Cons:** no namespaced member access (`Direction.North`) — use the const-object
pattern if you want that.

> **Note (when an `enum` is the right tool):** `enum` is not banned. Use it
> *deliberately* when you genuinely need a **runtime object** to iterate members,
> a stable set of **namespaced numeric constants**, or interop with code/APIs that
> already expect a TS enum. The guidance is *prefer literal unions by default*,
> not *never use enums*. If you do use one, prefer **string enums** over numeric
> enums (numeric enums allow assignment from arbitrary numbers and add reverse
> mappings), and avoid `const enum` in codebases using `isolatedModules`/bundlers.

---

## 18a. Objects & arrays (JS footguns)

TypeScript types don't protect against several inherited JavaScript runtime
footguns. Apply these regardless of types:

**`TS-OBJ-01` (SHOULD)** Use literal `[]` and `{}` over `new Array(...)`/`new
Object()`. `new Array(2)` creates a length-2 **sparse** array (holes), not `[2]` or
two zeroed slots — a common surprise.

**`TS-OBJ-02` (SHOULD)** Always pass a compare function to `Array.prototype.sort`
for non-string data. The default sort coerces elements to **strings**, so
`[10, 9, 100].sort()` yields `[10, 100, 9]`. Use `arr.sort((a, b) => a - b)`.

**`TS-OBJ-03` (SHOULD)** Prefer `Object.keys/values/entries` (or a `Map`) over
`for...in`, which also iterates inherited enumerable properties. To test own
properties use **`Object.hasOwn(obj, key)`** (ES2022) rather than
`obj.hasOwnProperty(key)` (which breaks on null-prototype objects or a shadowed
`hasOwnProperty`).

**`TS-OBJ-04` (SHOULD NOT)** Don't export mutable bindings (`export let`); export
`const`/`readonly` values or accessor functions so consumers can't mutate module
state.

---

## 19. Security

See [security](../04-security.md). TS/JS-specific:

**`TS-SEC-01` (MUST NOT)** Never use `eval`/`new Function` on untrusted input; never
assign untrusted data to `innerHTML`/`dangerouslySetInnerHTML`/`v-html` (XSS) —
sanitize (DOMPurify) or use text APIs (`GEN-SEC-08`).
**`TS-SEC-02` (MUST)** Validate/parse all external input with a schema (`TS-ANY-03`)
— never trust `any`/`unknown` data shapes.
**`TS-SEC-03` (MUST)** Parameterize DB queries; use `crypto`/`crypto.subtle` (not
`Math.random()`) for tokens (`GEN-SEC-22`).
**`TS-SEC-04` (SHOULD)** Run `npm audit`/Snyk and pin/lockfile deps; beware
prototype pollution (`__proto__`) when merging untrusted objects; avoid `child_
process` with shell + untrusted input.
**`TS-SEC-05` (MUST)** Compare secrets, tokens, and signatures with
`crypto.timingSafeEqual` on equal-length buffers (hash or HMAC both sides first
if lengths can differ — it throws on length mismatch); never `===`/`!==` on
secret strings, which short-circuits at the first differing character and leaks
timing information (`GEN-SEC-13`).

---

## 20. Testing

See [testing](../05-testing.md). TS-specific:

**`TS-TEST-01` (SHOULD)** Use **Vitest** (or Jest) with the type checker as part of
CI. Write tests in TS so test code is type-checked too. Test through public
behavior, not internals.
**`TS-TEST-02` (SHOULD)** Use `@testing-library` for component/DOM testing
(behavior-focused), **Playwright/Cypress** for E2E (few, critical paths),
**fast-check** for property-based tests (`GEN-TEST-17`).
**`TS-TEST-03` (SHOULD)** Type-test public APIs (`expectTypeOf`/`tsd`) for library
code so type regressions are caught.

---

## 21. Anti-patterns

- `any` (explicit or implicit); type assertions (`as`) instead of narrowing;
  `as any`/double-casts.
- Non-strict `tsconfig` (no `strictNullChecks`/`noImplicitAny`).
- Trusting external data without schema validation.
- `var`; `==`/`!=`; `||` for defaults instead of `??`.
- Floating/unhandled promises; `await` in a loop for independent work.
- Consuming a `fetch` body without checking `response.ok`/status.
- Truthiness checks on nullable numerics (drops `0`); unguarded `Number(...)`
  coercion letting `NaN`/`Infinity` flow through.
- `===` on secret strings instead of `crypto.timingSafeEqual`.
- Module-top-level `process.env` capture; treating env strings (`"false"`) as
  booleans.
- Throwing non-`Error` values; assuming `catch` var is an `Error`; discarding the
  original error instead of passing `{ cause }`.
- `!` non-null assertions sprinkled to silence the compiler.
- `as` to silence/widen where `satisfies` would verify and preserve the type.
- Boolean-flag/optional-field state instead of discriminated unions; non-exhaustive
  switches.
- Default exports everywhere; circular deps; `namespace` for new code; `export let`
  (mutable exported bindings).
- `new Array(n)`/`new Object()` literals; `Array.sort` without a compare fn;
  `for...in`/`hasOwnProperty` instead of `Object.keys`/`Object.hasOwn`.
- `innerHTML`/`v-html` with untrusted data; `eval`; `Math.random()` for tokens.
- Deep conditional/mapped-type gymnastics where a simple type would do.

---

## Quick checklist

- [ ] ESLint(type-aware)+Prettier+`tsc --noEmit` in CI/pre-commit.
- [ ] `strict: true` plus `noUncheckedIndexedAccess` and friends.
- [ ] camelCase/PascalCase naming; no `I`-prefix; no type-encoding names.
- [ ] `const` by default, no `var`, `===` only, `??` for defaults.
- [ ] No `any`; use `unknown` + narrowing; validate external data with a schema
      (Zod) and infer types from it.
- [ ] `interface` for shapes, `type` for unions/mapped; consistent default.
- [ ] Narrowing/type guards over assertions; no `as`/`as any`/double-casts;
      `satisfies` to validate-without-widening.
- [ ] Discriminated unions for variant state; exhaustive switches via `never`.
- [ ] Constrained, readable generics; utility types reused.
- [ ] `readonly`/`Readonly`/`as const` for immutable data.
- [ ] Explicit null/undefined handling; `?.`/`??`; minimal `!`;
      `Number.isFinite` after numeric coercion; `!= null` (not truthiness) for
      nullable numerics.
- [ ] Annotated params & exported return types; options objects; no floating
      promises.
- [ ] Named exports; `import type`; no circular deps/namespaces; env read inside
      functions (no frozen module-level env constants); env booleans via typed
      helper.
- [ ] async/await; `Promise.all` for concurrency; cancellation + timeouts;
      `response.ok` checked before consuming `fetch` bodies.
- [ ] Throw `Error` subclasses; narrow `unknown` catch vars; preserve `{ cause }`;
      Result type for expected failures.
- [ ] String-literal unions/const-objects preferred over enums (string enums if an
      enum is genuinely needed).
- [ ] No `new Array`/`new Object`; numeric `sort` has compare fn;
      `Object.keys`/`Object.hasOwn` over `for...in`/`hasOwnProperty`; no `export let`.
- [ ] No `eval`/`innerHTML` on untrusted data; schema-validated input;
      parameterized queries; CSPRNG; audited/locked deps;
      `crypto.timingSafeEqual` for secret comparison.
- [ ] Vitest/Jest in TS; testing-library + Playwright + fast-check; type tests for
      libraries.

---

## References

- TypeScript Handbook — https://www.typescriptlang.org/docs/handbook/intro.html
- TSConfig reference (`strict`, etc.) — https://www.typescriptlang.org/tsconfig
- typescript-eslint — https://typescript-eslint.io/
- "Do's and Don'ts" (TS Handbook) — https://www.typescriptlang.org/docs/handbook/declaration-files/do-s-and-don-ts.html
- Zod — https://zod.dev/
- Effective TypeScript (Dan Vanderkam) — https://effectivetypescript.com/
- Total TypeScript (Matt Pocock) — https://www.totaltypescript.com/
