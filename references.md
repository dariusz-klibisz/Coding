# References & Sources

This reference set synthesizes widely adopted public guidance and authoritative
primary sources. It does not reproduce proprietary standards. When a project has
binding legal, regulatory, safety, or organization-specific requirements, those
requirements take precedence.

Each root and language document also carries its own inline citations next to the
relevant rules; this file is the consolidated index.

---

## General engineering & design

- Robert C. Martin, *Clean Code*; *Agile Software Development: Principles, Patterns,
  and Practices* (SOLID).
- Andrew Hunt & David Thomas, *The Pragmatic Programmer* (DRY, orthogonality).
- Steve McConnell, *Code Complete*, 2nd ed. (defensive programming,
  self-documenting code).
- Sandi Metz, "The Wrong Abstraction" — https://sandimetz.com/blog/2016/1/20/the-wrong-abstraction
- Joel Spolsky, "The Law of Leaky Abstractions" — https://www.joelonsoftware.com/2002/11/11/the-law-of-leaky-abstractions/
- Semantic Versioning — https://semver.org/
- Martin Fowler, "TechnicalDebt" — https://martinfowler.com/bliki/TechnicalDebt.html
- The Zen of Python (PEP 20) — https://peps.python.org/pep-0020/

## Defensive programming & error handling

- Bertrand Meyer, *Object-Oriented Software Construction* (Design by Contract).
- Alexis King, "Parse, Don't Validate" — https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/
- Tony Hoare, "Null References: The Billion Dollar Mistake."
- Michael Nygard, *Release It!* (circuit breaker, bulkhead, timeouts).
- Joshua Bloch, *Effective Java* (failure atomicity, exceptions).
- AWS Builders' Library, "Timeouts, retries and backoff with jitter" — https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/
- Microsoft, "Best practices for exceptions" — https://learn.microsoft.com/dotnet/standard/exceptions/best-practices-for-exceptions

## Security (secure coding)

- OWASP Top 10 (2021) — https://owasp.org/Top10/
- OWASP Application Security Verification Standard (ASVS) — https://owasp.org/www-project-application-security-verification-standard/
- OWASP Cheat Sheet Series — https://cheatsheetseries.owasp.org/
- OWASP Proactive Controls — https://owasp.org/www-project-proactive-controls/
- OWASP Secure Coding Practices Quick Reference — https://owasp.org/www-project-secure-coding-practices-quick-reference-guide/
- CWE Top 25 — https://cwe.mitre.org/top25/
- NIST SP 800-218, Secure Software Development Framework — https://csrc.nist.gov/publications/detail/sp/800-218/final
- NIST SP 800-63B (Digital Identity / authentication) — https://pages.nist.gov/800-63-3/sp800-63b.html
- SEI CERT C Coding Standard — https://wiki.sei.cmu.edu/confluence/display/c/
- Saltzer & Schroeder, "The Protection of Information in Computer Systems."
- SLSA supply-chain framework — https://slsa.dev/

## Testing

- Mike Cohn, *Succeeding with Agile* (test pyramid).
- Martin Fowler, "Practical Test Pyramid" — https://martinfowler.com/articles/practical-test-pyramid.html
- Kent C. Dodds, "The Testing Trophy" — https://kentcdodds.com/blog/the-testing-trophy-and-testing-classifications
- Kent Beck, *Test-Driven Development: By Example*.
- Gerard Meszaros, *xUnit Test Patterns* (test-double taxonomy).
- Hypothesis docs (property-based testing) — https://hypothesis.readthedocs.io/

## Performance

- Donald Knuth, "Structured Programming with go to Statements" (premature
  optimization).
- Brendan Gregg, *Systems Performance* / flame graphs — https://www.brendangregg.com/
- Gene Amdahl, "Amdahl's Law."
- "Latency Numbers Every Programmer Should Know" — https://gist.github.com/jboner/2841832

## Concurrency

- Brian Goetz et al., *Java Concurrency in Practice*.
- Rob Pike, "Concurrency is not Parallelism" — https://go.dev/blog/waza-talk
- Go proverbs — https://go-proverbs.github.io/
- Leslie Lamport, "Time, Clocks, and the Ordering of Events."
- ThreadSanitizer — https://github.com/google/sanitizers/wiki/ThreadSanitizerCppManual

## Game runtime performance & determinism

- Glenn Fiedler, "Fix Your Timestep!" — https://gafferongames.com/post/fix_your_timestep/
- Glenn Fiedler, "Deterministic Lockstep" — https://gafferongames.com/post/deterministic_lockstep/
- Glenn Fiedler, "Floating Point Determinism" — https://gafferongames.com/post/floating_point_determinism/
- Paul Bettner & Mark Terrano, "1500 Archers on a 28.8: Network Programming in
  Age of Empires and Beyond" — https://www.gamedeveloper.com/programming/1500-archers-on-a-28-8-network-programming-in-age-of-empires-and-beyond
- Factorio, "Friday Facts #302 — The multiplayer megapacket" — https://www.factorio.com/blog/post/fff-302
- Photon Quantum (fixed-point deterministic engine) — https://doc.photonengine.com/quantum/current/getting-started/quantum-intro
- Squirrel Eiserloh, GDC "Math for Game Programmers: Noise-Based RNG" — https://gdcvault.com/play/1024365/Math-for-Game-Programmers-Noise
- Robert Nystrom, *Game Programming Patterns* — https://gameprogrammingpatterns.com/
- Richard Fabian, *Data-Oriented Design* — https://www.dataorienteddesign.com/dodmain/
- Gary Bernhardt, "Functional Core, Imperative Shell" — https://www.destroyallsoftware.com/screencasts/catalog/functional-core-imperative-shell
- Alan Wolfe (demofox), "Demystifying Floating Point Precision" — https://blog.demofox.org/2017/11/21/floating-point-precision/
- Godot Docs, "Optimization: General" — https://docs.godotengine.org/en/stable/tutorials/performance/general_optimization.html
- Godot Docs, "Physics Interpolation" — https://docs.godotengine.org/en/stable/tutorials/physics/interpolation/index.html
- Godot Docs, "Troubleshooting physics issues" — https://docs.godotengine.org/en/stable/tutorials/physics/troubleshooting_physics_issues.html
- Godot Docs, "Using Jolt Physics" — https://docs.godotengine.org/en/stable/tutorials/physics/using_jolt_physics.html
- Godot Docs, "Large world coordinates" (float precision) — https://docs.godotengine.org/en/stable/tutorials/physics/large_world_coordinates.html
- Godot Docs, "Reducing stutter from shader (pipeline) compilations" — https://docs.godotengine.org/en/stable/tutorials/performance/pipeline_compilations.html
- Godot Docs, "Using multiple threads" — https://docs.godotengine.org/en/stable/tutorials/performance/using_multiple_threads.html
- Android vitals, "Slow rendering" (16 ms target; frozen frames > 700 ms) — https://developer.android.com/topic/performance/vitals/render
- Android Dynamic Performance Framework (thermal/performance-hint APIs) — https://developer.android.com/games/optimize/adpf
- Google, "Develop with Godot" (Android) — https://developer.android.com/games/engines/godot/godot-configure

## Version control, CI/CD

- DORA / *Accelerate* (Forsgren, Humble, Kim) — https://dora.dev/
- Jez Humble & David Farley, *Continuous Delivery*.
- Conventional Commits — https://www.conventionalcommits.org/
- Trunk-Based Development — https://trunkbaseddevelopment.com/
- Google Engineering Practices, "Code Review" — https://google.github.io/eng-practices/review/
- Pro Git — https://git-scm.com/book/

## Observability

- Google SRE Book, "Monitoring Distributed Systems" — https://sre.google/sre-book/monitoring-distributed-systems/
- Brendan Gregg, "USE Method" — https://www.brendangregg.com/usemethod.html
- "RED Method" — https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/
- OpenTelemetry — https://opentelemetry.io/docs/
- W3C Trace Context — https://www.w3.org/TR/trace-context/
- The Twelve-Factor App — https://12factor.net/

## Tooling & automation

- EditorConfig — https://editorconfig.org/
- pre-commit framework — https://pre-commit.com/
- OWASP Dependency-Check — https://owasp.org/www-project-dependency-check/
- gitleaks — https://github.com/gitleaks/gitleaks

## Documentation

- PEP 257 — Docstring Conventions — https://peps.python.org/pep-0257/
- Diátaxis documentation framework — https://diataxis.fr/
- Michael Nygard, "Documenting Architecture Decisions" (ADRs) — https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions
- Keep a Changelog — https://keepachangelog.com/
- Write the Docs — https://www.writethedocs.org/

---

## Language-specific

### Embedded C

- MISRA C:2012 / MISRA C:2023 — public overview at https://misra.org.uk/
- SEI CERT C Coding Standard — https://wiki.sei.cmu.edu/confluence/display/c/
- Barr Group, *Embedded C Coding Standard* — https://barrgroup.com/embedded-systems/books/embedded-c-coding-standard
- ISO/IEC 9899 (C standard, C99/C11/C17) — https://www.iso.org/standard/82075.html
- IEC 61508 (functional safety), ISO 26262 (automotive), DO-178C (airborne),
  IEC 62304 (medical device software).
- ARM, "CMSIS" and Cortex-M memory-barrier application notes.

### C# / .NET

- Microsoft, ".NET Coding Conventions (C#)" — https://learn.microsoft.com/dotnet/csharp/fundamentals/coding-style/coding-conventions
- Microsoft, "Framework Design Guidelines" — https://learn.microsoft.com/dotnet/standard/design-guidelines/
- Microsoft, ".NET code analysis overview" — https://learn.microsoft.com/dotnet/fundamentals/code-analysis/overview
- Microsoft, "Nullable reference types" — https://learn.microsoft.com/dotnet/csharp/nullable-references
- Microsoft, "Asynchronous programming" — https://learn.microsoft.com/dotnet/csharp/asynchronous-programming/
- Stephen Cleary, "Async/Await Best Practices" — https://learn.microsoft.com/archive/msdn-magazine/2013/march/async-await-best-practices-in-asynchronous-programming
- .NET Runtime coding style — https://github.com/dotnet/runtime/blob/main/docs/coding-guidelines/coding-style.md

### Python

- PEP 8 — Style Guide — https://peps.python.org/pep-0008/
- PEP 20 — The Zen of Python — https://peps.python.org/pep-0020/
- PEP 257 — Docstring Conventions — https://peps.python.org/pep-0257/
- PEP 484 / 526 / 604 / 695 — typing — https://peps.python.org/pep-0484/
- Python typing docs — https://docs.python.org/3/library/typing.html
- Packaging Python projects — https://packaging.python.org/
- Ruff — https://docs.astral.sh/ruff/ · Black — https://black.readthedocs.io/ ·
  mypy — https://mypy.readthedocs.io/

### TypeScript

- TypeScript Handbook — https://www.typescriptlang.org/docs/handbook/intro.html
- TSConfig reference — https://www.typescriptlang.org/tsconfig/
- "Do's and Don'ts" — https://www.typescriptlang.org/docs/handbook/declaration-files/do-s-and-don-ts.html
- typescript-eslint — https://typescript-eslint.io/
- ESLint — https://eslint.org/docs/latest/ · Prettier — https://prettier.io/docs/en/
- Zod — https://zod.dev/
- *Effective TypeScript* (Dan Vanderkam) — https://effectivetypescript.com/
- TypeScript 5.5 release notes (inferred type predicates) — https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-5.html
- TypeScript 3.9 release notes (`@ts-expect-error`) — https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-9.html

### Vue

- Vue documentation — https://vuejs.org/guide/introduction.html
- Vue Style Guide (A–D) — https://vuejs.org/style-guide/
- Composition API / `<script setup>` — https://vuejs.org/api/sfc-script-setup.html
- Reactivity fundamentals — https://vuejs.org/guide/essentials/reactivity-fundamentals.html
- Composables — https://vuejs.org/guide/reusability/composables.html
- Vue Security guide — https://vuejs.org/guide/best-practices/security.html
- Vue Performance guide — https://vuejs.org/guide/best-practices/performance.html
- Vue TypeScript guide — https://vuejs.org/guide/typescript/overview.html
- Pinia — https://pinia.vuejs.org/ · Vue Router — https://router.vuejs.org/ ·
  Vue Test Utils — https://test-utils.vuejs.org/
- eslint-plugin-vue — https://eslint.vuejs.org/
- Vue — List Rendering (`v-if`/`v-for` precedence in Vue 3) — https://vuejs.org/guide/essentials/list.html
- Vue — Props: Reactive Props Destructure (3.5+) — https://vuejs.org/guide/components/props.html
- Vue — Server-Side Rendering guide (hydration) — https://vuejs.org/guide/scaling-up/ssr.html
- Nuxt — Data Fetching — https://nuxt.com/docs/getting-started/data-fetching ·
  Error Handling — https://nuxt.com/docs/getting-started/error-handling
- Vue I18n — https://vue-i18n.intlify.dev/

### GDScript / Godot

- Godot Docs, "GDScript reference" — https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_basics.html
- Godot Docs, "GDScript style guide" — https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_styleguide.html
- Godot Docs, "Static typing in GDScript" — https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/static_typing.html
- Godot Docs, "Godot notifications" (node lifecycle order) — https://docs.godotengine.org/en/stable/tutorials/best_practices/godot_notifications.html
- Godot Docs, "Using signals" — https://docs.godotengine.org/en/stable/getting_started/step_by_step/signals.html
- Godot Docs, "Autoloads (singletons)" — https://docs.godotengine.org/en/stable/tutorials/scripting/singletons_autoload.html
- Godot Docs, "Autoloads versus regular nodes" — https://docs.godotengine.org/en/stable/tutorials/best_practices/autoloads_versus_regular_nodes.html
- Godot Docs, "Scene organization" — https://docs.godotengine.org/en/stable/tutorials/best_practices/scene_organization.html
- Godot Docs, "When to use scenes versus scripts" — https://docs.godotengine.org/en/stable/tutorials/best_practices/scenes_versus_scripts.html
- Godot Docs, "Project organization" — https://docs.godotengine.org/en/stable/tutorials/best_practices/project_organization.html
- Godot Docs, "Data preferences" — https://docs.godotengine.org/en/stable/tutorials/best_practices/data_preferences.html
- Godot Docs, "Version control systems" — https://docs.godotengine.org/en/stable/tutorials/best_practices/version_control_systems.html
- Godot Docs, "Using InputEvent" — https://docs.godotengine.org/en/stable/tutorials/inputs/inputevent.html
- Godot Docs, "Background loading" — https://docs.godotengine.org/en/stable/tutorials/io/background_loading.html
- Godot Docs, "Thread-safe APIs" — https://docs.godotengine.org/en/stable/tutorials/performance/thread_safe_apis.html
- Godot Docs, "Resources" — https://docs.godotengine.org/en/stable/tutorials/scripting/resources.html
- Godot Docs, "Best Practices: Node alternatives" — https://docs.godotengine.org/en/stable/tutorials/best_practices/node_alternatives.html
- GDScript Toolkit (gdformat/gdlint) — https://github.com/Scony/godot-gdscript-toolkit
- GUT (Godot Unit Test) — https://github.com/bitwes/Gut
- GdUnit4 (third-party test framework) — https://github.com/godot-gdunit-labs/gdUnit4
- godot-ci (headless CI images) — https://github.com/abarichello/godot-ci
- Sentry Godot SDK — https://docs.sentry.io/platforms/godot/
- GDQuest, GDScript guidelines (principles; syntax partially Godot 3-era) — https://gdquest.gitbook.io/gdquests-guidelines/godot-gdscript-guidelines
- Game Accessibility Guidelines — https://gameaccessibilityguidelines.com/
- Xbox Accessibility Guidelines — https://learn.microsoft.com/en-us/gaming/accessibility/guidelines

### SQL / PostgreSQL / PostGIS

- PostgreSQL documentation — https://www.postgresql.org/docs/current/
- PostgreSQL wiki: "Don't Do This" — https://wiki.postgresql.org/wiki/Don%27t_Do_This
- PostGIS documentation — https://postgis.net/documentation/
- PostGIS Workshop: Spatial Indexing — https://postgis.net/workshops/postgis-intro/indexing.html
- PgBouncer features (pooling-mode caveats) — https://www.pgbouncer.org/features.html

### Docker

- Docker docs — Building best practices — https://docs.docker.com/build/building/best-practices/
- Dockerfile reference — https://docs.docker.com/reference/dockerfile/
- Docker build secrets (BuildKit) — https://docs.docker.com/build/building/secrets/
- Compose Specification — https://docs.docker.com/reference/compose-file/
- hadolint — https://github.com/hadolint/hadolint

### YAML

- YAML specification 1.2.2 — https://yaml.org/spec/1.2.2/
- YAML 1.1 bool / merge-key working drafts — https://yaml.org/type/bool.html · https://yaml.org/type/merge.html
- yamllint documentation — https://yamllint.readthedocs.io/

---

## Authority notes

- Official language and framework documentation is **primary** for syntax,
  semantics, and supported APIs.
- Style guides are context-sensitive; they are strongest when aligned with a
  repository's existing conventions.
- Security standards and checklists identify common controls, but
  application-specific threat modeling is still required.
- **MISRA C is a proprietary safety-oriented standard.** This reference discusses
  publicly known embedded-C safety concepts and, where helpful, cites MISRA rule
  *numbers* for traceability, but it does **not** reproduce MISRA rule text. Obtain
  the official MISRA documents for authoritative rule wording and for any
  compliance/certification work.
