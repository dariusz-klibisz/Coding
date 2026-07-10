# GDScript / Godot — Coding Standards & State-of-the-Art Practices

> **Scope:** GDScript on current Godot (4.2+; examples assume 4.x's typed-GDScript
> and `@export`/`@onready` annotation syntax). Notes on C# in Godot are included
> where the runtime model (node lifecycle, threading) differs from plain .NET —
> see [`csharp.md`](csharp.md) for general C# rules, which still apply to Godot C#.
> **Primary sources:** Godot Engine official documentation (GDScript reference,
> Style Guide, Best Practices, Performance docs).
> **Relationship to general docs:** extends [general docs](../00-index.md). Where a
> general rule and a GDScript rule conflict, the GDScript rule wins for `.gd` files.
> Real-time/runtime-performance and determinism concerns that apply beyond
> GDScript specifically are in
> [`13-game-runtime-and-determinism.md`](../13-game-runtime-and-determinism.md).

**Rule ID prefix:** `GD`

---

## Table of contents

1. [Tooling & enforcement](#1-tooling--enforcement)
2. [Naming conventions](#2-naming-conventions)
3. [Formatting & layout](#3-formatting--layout)
4. [Static typing](#4-static-typing)
5. [Node lifecycle](#5-node-lifecycle)
6. [Signals & await](#6-signals--await)
7. [Exports, resources & scene composition](#7-exports-resources--scene-composition)
8. [Autoloads / singletons](#8-autoloads--singletons)
9. [Error handling](#9-error-handling)
10. [Memory / resource management](#10-memory--resource-management)
11. [Concurrency](#11-concurrency)
12. [Security](#12-security)
13. [Testing](#13-testing)
14. [Anti-patterns](#14-anti-patterns)
15. [Quick checklist](#quick-checklist)
16. [References](#references)

---

## 1. Tooling & enforcement

**`GD-TOOL-01` (SHOULD)** Enable **typed GDScript** project-wide (variables,
parameters, return types) and treat the editor's type-checking warnings as
review blockers — see §4. This is the single highest-leverage tooling choice:
it catches a large class of runtime `null`/type errors at edit time and enables
autocompletion.

**`GD-TOOL-02` (SHOULD)** Use `gdformat`/`gdlint` (GDToolkit) or the built-in
script editor formatting in CI/pre-commit so style is automatic and reviews
focus on logic, not whitespace (builds on
[tooling & automation](../11-tooling-and-automation.md)).

**`GD-TOOL-03` (SHOULD)** Keep the Godot editor's **Debugger → Errors** and
**Profiler** panels part of the normal workflow, not a last resort — GDScript
errors that don't halt execution (e.g. calling a method on a freed object) are
easy to miss without watching the panel during play-testing.

---

## 2. Naming conventions

Per the official GDScript style guide:

| Construct | Convention | Example |
|-----------|------------|---------|
| Class name (`class_name`), node names in scenes | **PascalCase** | `HeroUnit`, `EnemySpawner` |
| Functions, variables, signals | **snake_case** | `take_damage`, `current_health`, `signal health_changed` |
| Constants, enums (values) | **CONSTANT_CASE** | `MAX_PARTY_SIZE`, `DamageType.FIRE` |
| Private/internal members (convention, not enforced) | leading underscore | `_internal_state` |
| File names | snake_case matching the class | `hero_unit.gd` |

**`GD-NAME-01` (SHOULD)** Name signals in the **past tense** for events that
already happened (`health_changed`, `item_equipped`) and boolean-returning
functions/properties as assertions (`is_alive`, `can_cast`). This matches
Godot's own API and keeps signal-driven code readable.

**`GD-NAME-02` (SHOULD)** Prefix a leading underscore on virtual/lifecycle
overrides only where Godot's own convention already uses it (`_ready`,
`_process`) — don't invent new leading-underscore private-by-convention members
inconsistently within the same class.

---

## 3. Formatting & layout

**`GD-FMT-01` (MUST)** Use **tabs** for indentation (the Godot style guide's
default and what `gdformat` produces) — mixing tabs/spaces in `.gd` files is a
common source of "works in editor, breaks on load" diffs.

**`GD-FMT-02` (SHOULD)** Order a script's contents consistently:
`class_name` → `extends` → docstring → signals → enums → constants →
`@export` vars → public vars → private (`_`) vars → `@onready` vars →
`_ready`/lifecycle methods → public methods → private methods. Consistent
ordering makes large `.gd` files skimmable and diffs smaller.

**`GD-FMT-03` (SHOULD)** One blank line between logical sections (signals,
exports, methods); no blank line between an `@export` annotation and the
variable it decorates.

---

## 4. Static typing

**`GD-TYPE-01` (SHOULD)** Type every variable, parameter, and return value you
control:

```gdscript
# Bad — untyped, no autocomplete, no compile-time check
func compute_damage(attacker, target, modifier):
    return attacker.power * modifier - target.armor

# Good — typed, editor catches a wrong-type call, autocompletes members
func compute_damage(attacker: Hero, target: Enemy, modifier: float) -> float:
    return attacker.power * modifier - target.armor
```

**Why:** Untyped GDScript defers every member-access and argument-type error to
runtime, often deep into a play session. Typed GDScript also compiles to
somewhat faster bytecode than the dynamic path (`GD-PERF` note in
[`13-game-runtime-and-determinism.md`](../13-game-runtime-and-determinism.md)).

**`GD-TYPE-02` (SHOULD)** Prefer `class_name` + typed references over
`get_node()` with string paths for anything accessed more than once — string
paths break silently on scene refactors; typed exports/`class_name` references
fail loudly (editor error) instead.

**`GD-TYPE-03` (CONSIDER)** Use the `:=` inferred-typing operator
(`var speed := 5.0`) when the right-hand side makes the type obvious, matching
`CS-VAR-01`'s "only when obvious" reasoning for C#'s `var`.

---

## 5. Node lifecycle

**`GD-LIFE-01` (MUST)** Understand the standard callback order and use each for
its intended purpose:

| Callback | Runs | Use for |
|----------|------|---------|
| `_init()` | Object construction (scene tree not yet available) | Pure in-memory setup only; **no node-tree access** |
| `_enter_tree()` | Node enters the scene tree | Rare; tree-structure-dependent setup |
| `_ready()` | Node + all children are in the tree | Cache child references (`@onready`), connect signals, one-time setup |
| `_process(delta)` | Every rendered frame (variable `delta`) | Visual/UI updates, animation, anything tied to frame rate |
| `_physics_process(delta)` | Fixed physics tick (constant `delta`) | Movement, collision-affecting logic, anything needing determinism |
| `_exit_tree()` | Node about to leave the tree | Cleanup: disconnect signals connected in `_ready`, release external resources |

**`GD-LIFE-02` (MUST)** Put **gameplay-simulation logic that must be
deterministic or frame-rate-independent** in `_physics_process`, never
`_process`. `_process` runs once per rendered frame — its `delta` varies with
frame rate and dropped frames, so anything computed there (damage-over-time
ticks, cooldown countdowns feeding combat resolution) will diverge across
machines and across an offline-catch-up recompute. See
[`13-game-runtime-and-determinism.md`](../13-game-runtime-and-determinism.md)
§ fixed-timestep for why this matters project-wide, not just per-script.

**`GD-LIFE-03` (SHOULD)** Cache child-node references in `@onready var`, not by
calling `get_node()`/`$Path` repeatedly inside `_process`/`_physics_process` —
tree lookups are needless per-frame overhead for something that doesn't move.

```gdscript
# Bad — re-resolves the node path every physics tick
func _physics_process(delta: float) -> void:
    $HealthBar.value = health

# Good — resolved once in _ready
@onready var health_bar: ProgressBar = $HealthBar

func _physics_process(delta: float) -> void:
    health_bar.value = health
```

**`GD-LIFE-04` (SHOULD NOT)** Don't assume a node is still valid after any
`await` or after emitting a signal that might free it — re-check
`is_instance_valid()` (§10) before touching a node reference you held across a
suspension point.

---

## 6. Signals & await

**`GD-SIG-01` (SHOULD)** Prefer **signals** over polling or direct
parent/sibling method calls for anything crossing a node-ownership boundary
(a `Hero` node should not reach up and call methods on its parent `Squad`
directly). Signals decouple emitter from listener and match Godot's own API
idiom.

**`GD-SIG-02` (MUST)** Disconnect signals connected manually
(`signal.connect(...)`) in `_exit_tree()` (or use `CONNECT_ONE_SHOT` /
one-shot patterns where appropriate) unless connected via the editor/`@onready`
pattern that Godot tears down automatically with the node. An undetached
handler on a freed emitter is Godot's version of a listener leak
(`GD-RES-02`).

**`GD-SIG-03` (SHOULD)** Use `await signal_name` for sequencing one-shot
asynchronous flows (animation finished, timer elapsed, HTTP request
completed) instead of polling a boolean in `_process`. This reads linearly and
avoids a busy-wait.

```gdscript
# Good — linear, no busy-wait polling
func play_attack_sequence() -> void:
    animation_player.play("attack")
    await animation_player.animation_finished
    apply_damage()
```

**`GD-SIG-04` (MUST NOT)** Don't `await` inside a code path that the
deterministic combat sim relies on for **ordering** without accounting for the
fact that other coroutines/signals may run interleaved during the suspension —
an `await` yields control back to the engine, and unrelated `_process` /
input / signal callbacks can execute before it resumes. Keep simulation-order-
sensitive logic synchronous within one `_physics_process` tick; use `await`
only for presentation-layer sequencing (see
[`13-game-runtime-and-determinism.md`](../13-game-runtime-and-determinism.md)
§ sim/presentation separation).

---

## 7. Exports, resources & scene composition

**`GD-EXP-01` (SHOULD)** Use `@export` to expose designer-tunable fields
(stats, references, tuning numbers) instead of hardcoding them, and type every
export. Group related exports with `@export_group`/`@export_category` once a
script has more than ~6 exported fields.

**`GD-EXP-02` (SHOULD)** Model **data-driven content** (classes, skills, items,
affixes, enemies — anything content designers author) as custom `Resource`
subclasses (`.tres`/`.res`) rather than hardcoded dictionaries or scene-tree
nodes. `Resource` is Godot's serializable-data type: it gets the inspector UI,
versioned `.tres` diffs, and reuse across scenes for free.

```gdscript
# item_definition.gd — a data-driven content resource, not a Node
class_name ItemDefinition
extends Resource

@export var item_id: StringName
@export var display_name: String
@export var rarity: Rarity
@export var affixes: Array[AffixDefinition]
```

**Why this matters here:** the project's itemization/skill/enemy content is
meant to scale via data files, not code changes (see the Design library's
[game architecture doc](../../Design/10-game-architecture.md) § data-driven
content pipeline). `Resource` is the concrete Godot mechanism for that.

**`GD-EXP-03` (SHOULD NOT)** Don't put gameplay **logic** in a `Resource` that
mutates shared state — `Resource` instances can be shared by reference across
multiple nodes unless duplicated (`resource_local_to_scene` or `.duplicate()`).
Mutating a shared `Resource` at runtime causes one hero's buff to silently
affect every hero using the same equipped-item resource. Keep `Resource`
subclasses as **immutable data templates**; runtime mutable state lives on the
`Node`/instance that owns it.

**`GD-EXP-04` (CONSIDER)** Favor **composition via child scenes/nodes** over
deep inheritance chains for hero/enemy variants — an `Enemy` scene composed of
a `HealthComponent`, `StatusEffectComponent`, and `AIComponent` child node is
easier to mix-and-match than an `Enemy → EliteEnemy → EliteFireEnemy`
inheritance tree. See the tech-agnostic ECS-vs-node-composition discussion in
[Design § game architecture](../../Design/10-game-architecture.md).

---

## 8. Autoloads / singletons

**`GD-AUTO-01` (SHOULD)** Reserve **autoloads** (Project Settings → Autoload)
for genuine cross-scene singletons with global lifetime: a `GameState`,
`AudioManager`, or the deterministic `CombatSimulator` service. Don't autoload
something that is really per-scene or per-instance state — that's a global
mutable variable in disguise and reintroduces the hazards in
[concurrency](../07-concurrency.md) §3 (shared mutable state) even in a
single-threaded script.

**`GD-AUTO-02` (SHOULD NOT)** Don't reach for autoloads as a substitute for
signals/dependency passing between unrelated nodes — overuse turns every
script into a hidden-coupling mess where "what can change this value" requires
searching the whole project. Prefer passing dependencies explicitly
(constructor-style `setup()` calls, exported node references) when the
relationship isn't genuinely global.

---

## 9. Error handling

Builds on [error handling](../03-error-handling.md).

**`GD-ERR-01` (SHOULD)** Use `assert()` liberally in debug builds for
programmer-error invariants (a function precondition that should never be
false if the rest of the code is correct) — `assert()` is stripped in release
exports, so it costs nothing in production while catching bugs during
development (`GEN-DEF` design-by-contract).

**`GD-ERR-02` (MUST)** Check `Error` return codes from engine APIs that can
fail (`file.open()`, `signal.connect()`) rather than assuming success —
GDScript's engine bindings return `Error` enums (`OK`, `ERR_*`), not
exceptions; an unchecked `ERR_FILE_NOT_FOUND` silently leaves state
uninitialized.

**`GD-ERR-03` (SHOULD)** Guard against calling methods on a node that may have
been freed: check `is_instance_valid(node)` before use if the reference was
held across a frame boundary, a signal, or an `await` (§10, §6). Godot does not
null out a stale reference to a freed `Node` automatically in all cases —
calling into one raises a runtime error.

---

## 10. Memory / resource management

Builds on [defensive programming](../02-defensive-programming.md) §11.

**`GD-RES-01` (MUST)** Call `queue_free()` to remove a node (never `free()`
inside a signal callback or during physics/process callbacks — it can free a
node mid-iteration of the scene tree and crash). Reserve direct `free()` for
`RefCounted`/plain `Object` cleanup outside the tree.

**`GD-RES-02` (SHOULD)** Disconnect long-lived signal connections before
freeing the connected node if the *other* side of the connection outlives it
(see `GD-SIG-02`) — otherwise the surviving object holds a dangling
callable that errors the next time the signal fires.

**`GD-RES-03` (SHOULD)** Use **object pooling** for anything spawned/destroyed
at high frequency during combat (damage-number labels, hit-VFX, projectiles) —
instancing and freeing a `PackedScene` every frame causes allocation churn and
GC/deallocation spikes. Pre-instantiate a pool and toggle
`visible`/`process_mode` instead. See
[`13-game-runtime-and-determinism.md`](../13-game-runtime-and-determinism.md)
§ object pooling for the frame-budget reasoning.

**`GD-RES-04` (SHOULD)** Preload/cache `PackedScene` and `Resource` references
used repeatedly (`preload()` at parse time, or a cached `load()` result) rather
than calling `load()` on the same path every time an instance is spawned —
`load()` re-touches the resource cache and is unnecessary overhead in a hot
spawn path.

---

## 11. Concurrency

Builds on [concurrency](../07-concurrency.md). Godot-specific:

**`GD-CONC-01` (MUST)** Treat the **scene tree as main-thread-affine**: node
creation, node-tree mutation, and most engine API calls from a background
thread are unsafe. Use `call_deferred()` (or `call_thread_safe()` in Godot
4.x) to marshal a result back onto the main thread instead of touching nodes
directly from a `WorkerThreadPool` task or `Thread`.

**`GD-CONC-02` (SHOULD)** Offload genuinely CPU-heavy, tree-independent work
(procedural generation, large batch calculations for offline-catch-up
simulation) to `WorkerThreadPool` tasks or a `Thread`, and hand results back
via `call_deferred` — don't block `_process`/`_physics_process` with long
synchronous computation, which stalls rendering and input for every node
(`GEN-CONC-14`).

**`GD-CONC-03` (SHOULD)** For Godot **C#** specifically, the same main-thread
affinity applies to the scene tree even though .NET's `Task`/`async` machinery
is available — don't call into `Node` APIs from a `Task.Run` continuation
without marshaling back to the main thread (Godot doesn't provide a
`SynchronizationContext` the way a UI framework's dispatcher does; use
`CallDeferred` from the C# `GodotObject` API). General C# async rules in
[`csharp.md`](csharp.md) §13 still apply on top of this constraint.

---

## 12. Security

See [security](../04-security.md). Godot-specific:

**`GD-SEC-01` (MUST)** Never deserialize or `load()` untrusted/user-supplied
`.tres`/`.res`/`.gd` content from outside the shipped project — Godot resource
loading can construct arbitrary registered classes and, for scripts, execute
code. Treat any moddability/import feature (see the itemization/content
pipeline's moddability goal) as a security boundary: load community content
through a restricted, whitelisted data format (plain JSON/plain-data
`Resource` subclasses with no executable fields), never as a loaded `.gd`
script or a `Resource` type that can reference arbitrary scripts.

**`GD-SEC-02` (SHOULD)** Validate and clamp any value that reaches gameplay
math from an external source (imported save file, cloud-sync payload, PvP
snapshot from another player) before using it — a corrupted or tampered save
must not be able to produce negative health pools, NaN stats, or out-of-range
array indices that crash the sim.

**`GD-SEC-03` (SHOULD)** For save files, use Godot's built-in encryption
(`FileAccess.open_encrypted_with_pass` / `ConfigFile` with encryption) or an
explicit checksum/signature only as a **tamper-resistance** measure for a
no-P2W economy (detect and reject an edited save; don't rely on it as strong
security) — a local single-player save is not a trust boundary in the
cryptographic sense, but consistency validation still matters (`GD-SEC-02`).

---

## 13. Testing

See [testing](../05-testing.md). Godot-specific:

**`GD-TEST-01` (SHOULD)** Use **GUT (Godot Unit Test)** or Godot 4's built-in
`GdUnit4` for GDScript unit/integration tests; both run inside the engine so
tests can instantiate real scenes/nodes rather than mocking the entire engine
API surface.

**`GD-TEST-02` (SHOULD)** Write **headless, engine-run tests for the
deterministic combat sim** specifically: given a fixed seed and a fixed
loadout, assert the resulting damage/outcome log is byte-identical across
repeated runs. This is the concrete test for the determinism requirement in
[`13-game-runtime-and-determinism.md`](../13-game-runtime-and-determinism.md)
and should run in CI headless (`godot --headless --script res://test_runner.gd`).

**`GD-TEST-03` (CONSIDER)** Separate "logic" scripts (plain `RefCounted`/pure
functions with no node dependency) from "glue" scripts (`Node`-derived,
wiring logic to the scene) so the logic half is trivially unit-testable
without spinning up a scene tree at all.

---

## 14. Anti-patterns

- Untyped GDScript in new code (no static analysis, silent runtime type errors).
- Gameplay-affecting logic in `_process` instead of `_physics_process`
  (frame-rate-dependent, non-deterministic).
- Repeated `get_node()`/`$Path` string lookups inside per-frame callbacks.
- Mutating a shared `Resource` instance at runtime and having it silently
  affect every node referencing it.
- Autoloads used as global mutable-state dumping grounds.
- Calling `free()` on a node during signal/physics callbacks instead of
  `queue_free()`.
- Spawning/freeing scenes at high frequency instead of pooling
  (damage numbers, VFX, projectiles).
- Touching `Node`/scene-tree APIs from a background `Thread`/`WorkerThreadPool`
  task without `call_deferred`.
- Loading untrusted external content as executable `Resource`/script types.
- `await`ing inside sim-order-critical combat-resolution code without
  accounting for interleaved execution.

---

## Quick checklist

- [ ] Static typing on every variable/parameter/return you control (`GD-TYPE-*`).
- [ ] Deterministic/frame-rate-independent gameplay logic lives in
      `_physics_process`, not `_process` (`GD-LIFE-02`).
- [ ] Child-node references cached via `@onready`, not re-resolved per frame
      (`GD-LIFE-03`).
- [ ] Signals connected manually are disconnected in `_exit_tree`; freed-node
      references guarded with `is_instance_valid` (`GD-SIG-02`, `GD-ERR-03`).
- [ ] Data-driven content (classes/skills/items/affixes/enemies) modeled as
      `Resource` subclasses, treated as immutable templates (`GD-EXP-02/03`).
- [ ] Autoloads reserved for genuine global singletons, not convenience globals
      (`GD-AUTO-*`).
- [ ] High-frequency spawn/destroy (VFX, damage numbers, projectiles) pooled,
      not instanced-and-freed per event (`GD-RES-03`).
- [ ] Scene-tree/`Node` APIs never touched from a background thread without
      `call_deferred`/`call_thread_safe` (`GD-CONC-01`).
- [ ] Untrusted content (mods, imports, sync payloads) loaded only as
      restricted plain data, never as executable scripts (`GD-SEC-01`).
- [ ] External/tampered values entering gameplay math are validated/clamped
      (`GD-SEC-02`).
- [ ] Deterministic combat sim covered by headless, seeded, repeatable tests
      (`GD-TEST-02`).

---

## References

- Godot Docs, "GDScript reference" — https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_basics.html
- Godot Docs, "GDScript style guide" — https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_styleguide.html
- Godot Docs, "Static typing in GDScript" — https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/static_typing.html
- Godot Docs, "Godot notification order" / node lifecycle — https://docs.godotengine.org/en/stable/tutorials/best_practices/godot_notifications.html
- Godot Docs, "Using signals" — https://docs.godotengine.org/en/stable/getting_started/step_by_step/signals.html
- Godot Docs, "Autoloads (singletons)" — https://docs.godotengine.org/en/stable/tutorials/scripting/singletons_autoload.html
- Godot Docs, "Thread-safe APIs" — https://docs.godotengine.org/en/stable/tutorials/performance/thread_safe_apis.html
- Godot Docs, "Using multiple threads" — https://docs.godotengine.org/en/stable/tutorials/performance/using_multiple_threads.html
- Godot Docs, "C# basics" (threading & lifecycle differences) — https://docs.godotengine.org/en/stable/tutorials/scripting/c_sharp/c_sharp_basics.html
- Godot Docs, "Resources" — https://docs.godotengine.org/en/stable/tutorials/scripting/resources.html
- Godot Docs, "Best Practices: Node alternatives" (composition guidance) — https://docs.godotengine.org/en/stable/tutorials/best_practices/node_alternatives.html
- GUT (Godot Unit Test) — https://github.com/bitwes/Gut
