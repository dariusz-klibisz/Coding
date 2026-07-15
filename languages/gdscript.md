# GDScript / Godot — Coding Standards & State-of-the-Art Practices

> **Scope:** GDScript on current Godot 4.x (validated against Godot 4.7; features
> introduced after 4.2 are version-tagged, e.g. "4.4+"). Examples assume typed
> GDScript and `@export`/`@onready` annotation syntax. Notes on C# in Godot are
> included where the runtime model (node lifecycle, threading) differs from plain
> .NET — see [`csharp.md`](csharp.md) for general C# rules, which still apply to
> Godot C#.
> **Primary sources:** Godot Engine official documentation (GDScript reference,
> Style Guide, Best Practices, Performance docs) and the sources listed under
> [References](#references).
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
8. [Scene & project organization](#8-scene--project-organization)
9. [Autoloads / singletons](#9-autoloads--singletons)
10. [Input handling](#10-input-handling)
11. [Error handling](#11-error-handling)
12. [Memory / resource management](#12-memory--resource-management)
13. [Performance idioms](#13-performance-idioms)
14. [Concurrency](#14-concurrency)
15. [Security](#15-security)
16. [Testing](#16-testing)
17. [Observability & telemetry](#17-observability--telemetry)
18. [Version control & CI](#18-version-control--ci)
19. [Accessibility](#19-accessibility)
20. [Anti-patterns](#20-anti-patterns)
21. [Quick checklist](#quick-checklist)
22. [References](#references)

---

## 1. Tooling & enforcement

**`GD-TOOL-01` (SHOULD)** Enable **typed GDScript** project-wide (variables,
parameters, return types) and treat the editor's type-checking warnings as
review blockers — see §4. This is the single highest-leverage tooling choice:
it catches a large class of runtime `null`/type errors at edit time and enables
autocompletion.

**`GD-TOOL-02` (SHOULD)** Use `gdformat`/`gdlint` (GDScript Toolkit) in
CI/pre-commit so style is automatic and reviews focus on logic, not whitespace
(builds on [tooling & automation](../11-tooling-and-automation.md)). Add
`godot --headless --check-only --script <file>` (or a project-wide script-load
pass) to CI to catch parse/type errors without opening the editor.

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
| Enum names | **PascalCase**, singular | `enum Element` |
| Private/internal members (convention, not enforced) | leading underscore | `_internal_state` |
| File names | snake_case matching the class | `hero_unit.gd` |

**`GD-NAME-01` (SHOULD)** Name signals in the **past tense** for events that
already happened (`health_changed`, `item_equipped`) and boolean-returning
functions/properties as assertions with an `is_`/`can_`/`has_` prefix
(`is_alive`, `can_cast`). This matches Godot's own API and keeps signal-driven
code readable.

**`GD-NAME-02` (SHOULD)** Prefix a leading underscore on virtual/lifecycle
overrides only where Godot's own convention already uses it (`_ready`,
`_process`) — don't invent new leading-underscore private-by-convention members
inconsistently within the same class.

---

## 3. Formatting & layout

**`GD-FMT-01` (MUST)** Use **tabs** for indentation (the Godot style guide's
default and what `gdformat` produces) — mixing tabs/spaces in `.gd` files is a
common source of "works in editor, breaks on load" diffs.

**`GD-FMT-02` (SHOULD)** Order a script's contents per the official style
guide's code order: `@tool`/`@icon`/`@static_unload` → `@abstract` (4.5+) +
`class_name` → `extends` → doc comment → signals → enums → constants → static
variables → `@export` vars → public vars → private (`_`) vars → `@onready`
vars → `_static_init()`/static methods → overridden virtual methods
(`_init`, `_enter_tree`, `_ready`, `_process`, `_physics_process`, remaining
virtuals) → public methods → private methods → inner classes. Consistent
ordering makes large `.gd` files skimmable and diffs smaller.

**`GD-FMT-03` (SHOULD)** One blank line between logical sections (signals,
exports, methods); two blank lines between functions; no blank line between an
`@export` annotation and the variable it decorates.

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
somewhat faster bytecode than the dynamic path (`GD-PERF` notes in
[`13-game-runtime-and-determinism.md`](../13-game-runtime-and-determinism.md)).

**`GD-TYPE-02` (SHOULD)** Prefer `class_name` + typed references over
`get_node()` with string paths for anything accessed more than once — string
paths break silently on scene refactors; typed exports/`class_name` references
fail loudly (editor error) instead.

**`GD-TYPE-03` (CONSIDER)** Use the `:=` inferred-typing operator
(`var speed := 5.0`) when the right-hand side makes the type obvious, matching
the official style guide's "prefer `:=` when the type is written on the same
line" guidance and `CS-VAR-01`'s "only when obvious" reasoning for C#'s `var`.

**`GD-TYPE-04` (SHOULD)** Use **typed collections** where element/key types are
known: `Array[Type]` and, on Godot 4.4+, `Dictionary[KeyType, ValueType]`. Typed
collections catch wrong-element assignments at parse time and enable
autocompletion on elements. On 4.5+, use `@abstract` classes/methods to define
contracts that concrete subclasses must implement (e.g. a base `SkillEffect`
with an `@abstract func apply(...)`) instead of runtime `push_error` stubs.

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
| `_physics_process(delta)` | Fixed physics tick (constant `delta`) | Movement, collision-affecting logic, fixed-rate gameplay logic |
| `_exit_tree()` | Node about to leave the tree | Cleanup: disconnect signals connected in `_ready`, release external resources |

**`GD-LIFE-02` (MUST)** Put **gameplay-simulation logic that must be
frame-rate-independent** in `_physics_process`, never `_process`. `_process`
runs once per rendered frame — its `delta` varies with frame rate and dropped
frames, so anything computed there (damage-over-time ticks, cooldown countdowns
feeding combat resolution) will diverge across machines and across a headless
recompute. Note that a fixed tick makes logic *frame-rate-independent*, not
automatically *deterministic* — the engine's physics simulation itself does not
guarantee reproducible results (`GEN-GAME-36`). See
[`13-game-runtime-and-determinism.md`](../13-game-runtime-and-determinism.md)
§ fixed-timestep and § determinism engineering.

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
`is_instance_valid()` (§12) before touching a node reference you held across a
suspension point.

---

## 6. Signals & await

**`GD-SIG-01` (SHOULD)** Prefer **signals** over polling or direct
parent/sibling method calls for anything crossing a node-ownership boundary
(a child unit node should not reach up and call methods on its parent squad
node directly). Signals decouple emitter from listener and match Godot's own
API idiom — see "call down, signal up" (`GD-ORG-01`).

**`GD-SIG-02` (SHOULD)** Disconnect manually-connected signals
(`signal.connect(...)`) when the *receiver* can outlive its interest in the
signal — typically in `_exit_tree()` — or use `CONNECT_ONE_SHOT` for one-shot
handlers. Note that Godot cleans up connections automatically when either
endpoint object is **freed** ("if the callable's object is freed, the
connection will be lost" — `Object.connect` docs), so the genuine hazards are:
(a) a node that is *removed from the tree but kept alive* (e.g. pooled — §12)
still receiving and acting on signals; and (b) lambda/bound `Callable`s that
capture state and aren't tied to a node's lifetime.

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

**`GD-SIG-04` (MUST NOT)** Don't `await` inside a code path that a
deterministic simulation relies on for **ordering** without accounting for the
fact that other coroutines/signals may run interleaved during the suspension —
an `await` yields control back to the engine, and unrelated `_process` /
input / signal callbacks can execute before it resumes. Keep simulation-order-
sensitive logic synchronous within one `_physics_process` tick; use `await`
only for presentation-layer sequencing (see
[`13-game-runtime-and-determinism.md`](../13-game-runtime-and-determinism.md)
§ sim/presentation separation).

**`GD-SIG-05` (SHOULD)** Use `Tween` and `AnimationPlayer` for presentation
animation (UI slides, fades, punch-scale effects) instead of hand-rolled
lerp-in-`_process` code — tweens are allocated once, run engine-side, kill
cleanly with the node, and compose with `await` (`tween.finished`; on 4.7+,
`Tween.tween_await()` can pause a tween until a signal fires).

---

## 7. Exports, resources & scene composition

**`GD-EXP-01` (SHOULD)** Use `@export` to expose designer-tunable fields
(stats, references, tuning numbers) instead of hardcoding them, and type every
export. Group related exports with `@export_group`/`@export_category` once a
script has more than ~6 exported fields.

**`GD-EXP-02` (SHOULD)** Model **data-driven content** (character classes,
skills, items, enemies — anything content designers author) as custom
`Resource` subclasses (`.tres`/`.res`) rather than hardcoded dictionaries or
scene-tree nodes. `Resource` is Godot's serializable-data type: it gets the
inspector UI, versioned `.tres` diffs, and reuse across scenes for free, and it
lets content scale via data files instead of code changes.

```gdscript
# item_definition.gd — a data-driven content resource, not a Node
class_name ItemDefinition
extends Resource

@export var item_id: StringName
@export var display_name: String
@export var rarity: Rarity
@export var affixes: Array[AffixDefinition]
```

**`GD-EXP-03` (SHOULD NOT)** Don't put gameplay **logic** in a `Resource` that
mutates shared state — `Resource` instances can be shared by reference across
multiple nodes unless duplicated (`resource_local_to_scene` or `.duplicate()`).
Mutating a shared `Resource` at runtime causes one unit's buff to silently
affect every unit using the same equipped-item resource. Keep `Resource`
subclasses as **immutable data templates**; runtime mutable state lives on the
`Node`/instance that owns it. (On 4.5+, `duplicate_deep()` gives explicit
control over how nested subresources are copied.)

**`GD-EXP-04` (CONSIDER)** Favor **composition via child scenes/nodes** over
deep inheritance chains for entity variants — an `Enemy` scene composed of a
`HealthComponent`, `StatusEffectComponent`, and `AIComponent` child node is
easier to mix-and-match than an `Enemy → EliteEnemy → EliteFireEnemy`
inheritance tree. This is the Component pattern (Nystrom, *Game Programming
Patterns*) expressed in Godot's scene system.

---

## 8. Scene & project organization

Based on Godot's official Best Practices (Scene organization, When to use
scenes versus scripts, Project organization).

**`GD-ORG-01` (SHOULD)** Follow **"call down, signal up"**: a parent may call
methods on the children it owns; a child communicates upward/outward only by
emitting signals (or through an explicitly injected dependency). Children that
reach up (`get_parent().something`) hard-couple the scene to one specific
ancestor arrangement and break reuse.

**`GD-ORG-02` (SHOULD)** Build scenes as **self-contained, reusable units**
with no hard external dependencies: a scene should be instantiable on its own
(or in a minimal test harness) without erroring. Dependencies a scene genuinely
needs from outside should be injected by the parent (exported `NodePath`/node
references, a `setup()` call), not discovered by absolute paths like
`/root/Level/...`.

**`GD-ORG-03` (SHOULD)** Use **scenes** for anything with composition/structure
(entities, UI panels, reusable effects) and plain **scripts** (`class_name`,
often `RefCounted`) for pure logic and data types with no tree presence —
per the official "scenes versus scripts" guidance. This keeps logic testable
(`GD-TEST-03`) and scenes shallow.

**`GD-ORG-04` (SHOULD)** Organize project folders **by feature/domain**
(`player/`, `enemies/goblin/`, `ui/inventory/`), keeping each scene next to its
script and the assets exclusive to it; put genuinely shared assets in a
`common/` (or `shared/`) directory. snake_case file/folder names, no spaces
(official project-organization guidance; also avoids case-sensitivity export
bugs across platforms).

---

## 9. Autoloads / singletons

**`GD-AUTO-01` (SHOULD)** Reserve **autoloads** (Project Settings → Autoload)
for genuine cross-scene singletons with global lifetime: a `GameState`,
`AudioManager`, or a simulation service that must survive scene changes. Don't
autoload something that is really per-scene or per-instance state — that's a
global mutable variable in disguise and reintroduces the hazards in
[concurrency](../07-concurrency.md) §3 (shared mutable state) even in a
single-threaded script.

**`GD-AUTO-02` (SHOULD NOT)** Don't reach for autoloads as a substitute for
signals/dependency passing between unrelated nodes — overuse turns every
script into a hidden-coupling mess where "what can change this value" requires
searching the whole project. Prefer passing dependencies explicitly
(constructor-style `setup()` calls, exported node references) when the
relationship isn't genuinely global (official "Autoloads versus regular
nodes" guidance).

---

## 10. Input handling

**`GD-INPUT-01` (SHOULD)** Define **InputMap actions** (Project Settings →
Input Map) and query actions (`Input.is_action_pressed("jump")`,
`event.is_action_pressed("jump")`) instead of hardcoding physical keys/buttons.
Actions decouple gameplay code from bindings, make gamepad/touch/keyboard
support uniform, and are the prerequisite for user-remappable controls
(`GD-ACC-01`).

**`GD-INPUT-02` (SHOULD)** Route input through the right callback:
`_unhandled_input()` for gameplay (so UI `Control` nodes consume clicks/keys
first), `_input()` only for things that must see events before the UI, and
`_gui_input()` for per-Control handling. Call `set_input_as_handled()` (or
`accept_event()`) when an event is consumed so it doesn't double-trigger.

**`GD-INPUT-03` (CONSIDER)** For touch platforms, use the built-in
`VirtualJoystick` node (4.7+) rather than a hand-rolled or third-party
on-screen stick, and test touch targets at real device DPI.

---

## 11. Error handling

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
held across a frame boundary, a signal, or an `await` (§12, §6). Godot does not
null out a stale reference to a freed `Node` automatically in all cases —
calling into one raises a runtime error.

---

## 12. Memory / resource management

Builds on [defensive programming](../02-defensive-programming.md) §11.

**`GD-RES-01` (MUST)** Call `queue_free()` to remove a node (never `free()`
inside a signal callback or during physics/process callbacks — it can free a
node mid-iteration of the scene tree and crash). Reserve direct `free()` for
`RefCounted`/plain `Object` cleanup outside the tree.

**`GD-RES-02` (SHOULD)** Disconnect long-lived signal connections when a node
is deactivated-but-alive (pooled, detached) and the emitter keeps firing — see
`GD-SIG-02` for when disconnection is actually required versus handled
automatically on free.

**`GD-RES-03` (SHOULD)** Use **object pooling** for anything spawned/destroyed
at high frequency during gameplay (damage-number labels, hit-VFX, projectiles) —
instancing and freeing a `PackedScene` every frame causes allocation churn and
deallocation spikes. Pre-instantiate a pool and toggle
`visible`/`process_mode` instead. See
[`13-game-runtime-and-determinism.md`](../13-game-runtime-and-determinism.md)
§ object pooling for the frame-budget reasoning.

**`GD-RES-04` (SHOULD)** Preload/cache `PackedScene` and `Resource` references
used repeatedly (`preload()` at parse time, or a cached `load()` result) rather
than calling `load()` on the same path every time an instance is spawned —
`load()` re-touches the resource cache and is unnecessary overhead in a hot
spawn path. On 4.5+, prefer preloading by **UID** (`uid://…`, Ctrl-drag a
resource into the script editor) over raw `res://` paths — UIDs survive file
moves/renames.

---

## 13. Performance idioms

Frame-budget reasoning lives in
[`13-game-runtime-and-determinism.md`](../13-game-runtime-and-determinism.md);
these are the GDScript-level mechanics.

**`GD-PERF-01` (SHOULD)** Use `StringName` (`&"group_name"`) for identifiers
compared or looked up in hot paths (group names, action names, animation
names, dictionary keys used per frame) — `StringName` comparisons are
pointer-equality, not character-by-character; most engine APIs take
`StringName` natively.

**`GD-PERF-02` (SHOULD)** Load heavyweight scenes/resources in the
**background** (`ResourceLoader.load_threaded_request()` +
`load_threaded_get()`) behind a loading indicator instead of a blocking
`load()` on the main thread — a synchronous multi-hundred-ms load in a
button handler is a guaranteed frozen frame (`GEN-GAME-31`).

**`GD-PERF-03` (CONSIDER)** For extreme entity/instance counts that outgrow
node-per-entity, drop to **packed arrays** (`PackedFloat32Array`, …),
`MultiMesh`, or the **Servers API** (`RenderingServer`, `PhysicsServer2D/3D`)
— the data-oriented layout avoids per-node overhead entirely
(`GEN-GAME-32`). Measure first; node-per-entity is fine for most counts.

---

## 14. Concurrency

Builds on [concurrency](../07-concurrency.md). Godot-specific:

**`GD-CONC-01` (MUST)** Treat the **scene tree as main-thread-affine**: node
creation, node-tree mutation, and most engine API calls from a background
thread are unsafe. Use `call_deferred()` (or `call_thread_safe()`) to marshal
a result back onto the main thread instead of touching nodes directly from a
`WorkerThreadPool` task or `Thread`.

**`GD-CONC-02` (SHOULD)** Offload genuinely CPU-heavy, tree-independent work
(procedural generation, large batch simulation passes) to `WorkerThreadPool`
tasks or a `Thread`, and hand results back via `call_deferred` — don't block
`_process`/`_physics_process` with long synchronous computation, which stalls
rendering and input for every node (`GEN-CONC-14`).

**`GD-CONC-03` (SHOULD)** For Godot **C#** specifically: Godot installs a
`GodotSynchronizationContext`, so `await` continuations in code started on the
main thread resume **on the main thread** (safe to touch nodes after a plain
`await` there). The hazards are code that *leaves* that context: the body of a
`Task.Run(...)` and any continuation after `ConfigureAwait(false)` run on
thread-pool threads and must not touch `Node`/scene-tree APIs — marshal back
with `CallDeferred` (or by awaiting back on the main context). General C#
async rules in [`csharp.md`](csharp.md) §13 still apply on top of this
constraint.

---

## 15. Security

See [security](../04-security.md). Godot-specific:

**`GD-SEC-01` (MUST)** Never deserialize or `load()` untrusted/user-supplied
`.tres`/`.res`/`.gd` content from outside the shipped project — Godot resource
loading can construct arbitrary registered classes and, for scripts, execute
code. Treat any moddability/import feature as a security boundary: load
external content through a restricted, whitelisted data format (plain
JSON/plain-data `Resource` subclasses with no executable fields), never as a
loaded `.gd` script or a `Resource` type that can reference arbitrary scripts.

**`GD-SEC-02` (SHOULD)** Validate and clamp any value that reaches gameplay
math from an external source (imported save file, cloud-sync payload, data
received from another player) before using it — a corrupted or tampered input
must not be able to produce negative health pools, NaN stats, or out-of-range
array indices that crash the simulation.

**`GD-SEC-03` (SHOULD)** For save files, use Godot's built-in encryption
(`FileAccess.open_encrypted_with_pass` / `ConfigFile` with encryption) or an
explicit checksum/signature as a **tamper-evidence** measure proportionate to
what's at stake (detect and reject an edited save; don't treat it as strong
security) — a local single-player save is not a cryptographic trust boundary,
but consistency validation still matters (`GD-SEC-02`, `GEN-GAME-30`).

---

## 16. Testing

See [testing](../05-testing.md). Godot-specific:

**`GD-TEST-01` (SHOULD)** Use a Godot-native test framework — **GUT (Godot
Unit Test)** or the third-party **GdUnit4** addon (GdUnit4 v5+ requires Godot
4.3+, v6+ requires 4.5+); Godot itself ships no built-in unit-test framework.
Both run inside the engine, so tests can instantiate real scenes/nodes rather
than mocking the entire engine API surface, and both provide a CLI/CI runner
(GUT's `gut_cmdln.gd`, the `gdunit4-action` GitHub Action).

**`GD-TEST-02` (SHOULD)** Write **headless, engine-run tests for any
deterministic simulation** specifically: given a fixed seed and fixed inputs,
assert the resulting outcome/event log is byte-identical across repeated runs.
This is the concrete test for the determinism requirement in
[`13-game-runtime-and-determinism.md`](../13-game-runtime-and-determinism.md)
and should run in CI headless (`godot --headless --script res://test_runner.gd`).

**`GD-TEST-03` (CONSIDER)** Separate "logic" scripts (plain `RefCounted`/pure
functions with no node dependency) from "glue" scripts (`Node`-derived,
wiring logic to the scene) so the logic half is trivially unit-testable
without spinning up a scene tree at all — the "functional core, imperative
shell" pattern (Bernhardt).

---

## 17. Observability & telemetry

Builds on [observability](../10-observability.md). Godot-specific:

**`GD-OBS-01` (SHOULD)** Ship production error visibility: on Godot 4.5+,
register a custom `Logger` to intercept errors/warnings, and enable
**Debug > Settings > GDScript > Always Track Call Stacks** so script
backtraces are available even in release exports. `push_error`/`push_warning`
without a consumer in a shipped build is telemetry that goes nowhere.

**`GD-OBS-02` (CONSIDER)** Use a crash/error-reporting service with first-class
Godot support (e.g. the official **Sentry Godot SDK**) for released games —
field crashes on player hardware (GPU drivers, thermal kills, OOM on mobile)
are otherwise invisible; store-level dashboards (Android vitals) only show
aggregates, not stack traces.

---

## 18. Version control & CI

Builds on [version control & CI](../08-version-control-and-ci.md) and
[tooling & automation](../11-tooling-and-automation.md). Godot-specific (per
the official "Version control systems" best-practices page):

**`GD-VCS-01` (MUST)** Ignore the `.godot/` directory (editor cache /
imported-asset artifacts — regenerated on first import) and **commit the
`.uid` files** Godot 4.4+ generates next to scripts/resources: UIDs are how
the engine tracks references across file moves/renames; deleting or ignoring
them breaks references for every other clone.

**`GD-VCS-02` (SHOULD)** Prefer **text-based** formats (`.tscn`, `.tres`) over
binary (`.scn`, `.res`) for scenes and resources you author — text formats
diff and merge; binary blobs don't. Use Git LFS (or equivalent) for genuinely
binary assets (textures, audio, models).

**`GD-VCS-03` (SHOULD)** Run engine-version upgrades that rewrite many files —
e.g. Godot 4.6's **Project > Tools > Upgrade Project Files** (which persists
unique node IDs into every scene) — as a **dedicated, mechanical commit** with
no logic changes mixed in, so the mass diff is reviewable and bisectable.

**`GD-CI-01` (SHOULD)** Automate a headless pipeline in CI: import the project
(`godot --headless --import`), run script checks (`GD-TOOL-02`) and the test
suite (`GD-TEST-01/02`), and produce export builds
(`godot --headless --export-release <preset>`) using pinned export templates —
the community-standard `godot-ci` Docker images package all of this.

---

## 19. Accessibility

**`GD-ACC-01` (CONSIDER)** Apply the basics of the industry accessibility
standards (Game Accessibility Guidelines; Xbox Accessibility Guidelines):
user-remappable controls (trivial if inputs are InputMap actions,
`GD-INPUT-01`), UI text scaling (theme font-size overrides), and never
encoding information in **color alone**. On Godot 4.5+, `Control` nodes carry
screen-reader (AccessKit) support — set accessible names/descriptions on
custom UI controls so they are readable at all.

---

## 20. Anti-patterns

- Untyped GDScript in new code (no static analysis, silent runtime type errors).
- Frame-rate-sensitive gameplay logic in `_process` instead of
  `_physics_process` (diverges with frame rate).
- Repeated `get_node()`/`$Path` string lookups inside per-frame callbacks.
- Mutating a shared `Resource` instance at runtime and having it silently
  affect every node referencing it.
- Autoloads used as global mutable-state dumping grounds.
- Children reaching up via `get_parent()`/absolute paths instead of
  "call down, signal up".
- Hardcoded key/button checks instead of InputMap actions.
- Calling `free()` on a node during signal/physics callbacks instead of
  `queue_free()`.
- Spawning/freeing scenes at high frequency instead of pooling
  (damage numbers, VFX, projectiles).
- Blocking `load()` of large scenes on the main thread instead of
  `ResourceLoader.load_threaded_request()`.
- Touching `Node`/scene-tree APIs from a background `Thread`/`WorkerThreadPool`
  task (or a C# thread-pool continuation) without `call_deferred`.
- Loading untrusted external content as executable `Resource`/script types.
- `await`ing inside sim-order-critical resolution code without accounting for
  interleaved execution.
- Committing `.godot/`, or ignoring/deleting `.uid` files (4.4+).

---

## Quick checklist

- [ ] Static typing on every variable/parameter/return you control, including
      typed `Array`/`Dictionary` collections (`GD-TYPE-*`).
- [ ] Frame-rate-independent gameplay logic lives in `_physics_process`, not
      `_process`; determinism-critical logic additionally avoids engine physics
      results (`GD-LIFE-02`, `GEN-GAME-36`).
- [ ] Child-node references cached via `@onready`, not re-resolved per frame
      (`GD-LIFE-03`).
- [ ] Signal hygiene: alive-but-detached receivers disconnected; freed-node
      references guarded with `is_instance_valid` (`GD-SIG-02`, `GD-ERR-03`).
- [ ] Presentation animation uses `Tween`/`AnimationPlayer`, not `_process`
      lerps (`GD-SIG-05`).
- [ ] Data-driven content modeled as `Resource` subclasses, treated as
      immutable templates (`GD-EXP-02/03`).
- [ ] Scenes are self-contained; "call down, signal up"; folders grouped by
      feature (`GD-ORG-*`).
- [ ] Autoloads reserved for genuine global singletons, not convenience globals
      (`GD-AUTO-*`).
- [ ] Input goes through InputMap actions and the correct
      `_input`/`_unhandled_input` callback (`GD-INPUT-*`).
- [ ] High-frequency spawn/destroy (VFX, damage numbers, projectiles) pooled,
      not instanced-and-freed per event (`GD-RES-03`).
- [ ] Hot paths use `StringName`; big loads run in the background
      (`GD-PERF-01/02`).
- [ ] Scene-tree/`Node` APIs never touched from a background thread or C#
      thread-pool continuation without `call_deferred` (`GD-CONC-01/03`).
- [ ] Untrusted content (mods, imports, sync payloads) loaded only as
      restricted plain data, never as executable scripts (`GD-SEC-01`).
- [ ] External/tampered values entering gameplay math are validated/clamped
      (`GD-SEC-02`).
- [ ] Deterministic simulation covered by headless, seeded, repeatable tests
      run in CI (`GD-TEST-02`, `GD-CI-01`).
- [ ] Release builds have error telemetry (custom `Logger`/backtraces, crash
      reporting) (`GD-OBS-*`).
- [ ] `.godot/` ignored; `.uid` files committed; text-based scene/resource
      formats; LFS for binaries (`GD-VCS-*`).
- [ ] Controls remappable, text scalable, color not the sole information
      channel (`GD-ACC-01`).

---

## References

- Godot Docs, "GDScript reference" — https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_basics.html
- Godot Docs, "GDScript style guide" — https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_styleguide.html
- Godot Docs, "Static typing in GDScript" — https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/static_typing.html
- Godot Docs, "Godot notifications" (node lifecycle order) — https://docs.godotengine.org/en/stable/tutorials/best_practices/godot_notifications.html
- Godot Docs, "Using signals" — https://docs.godotengine.org/en/stable/getting_started/step_by_step/signals.html
- Godot Docs, "Scene organization" — https://docs.godotengine.org/en/stable/tutorials/best_practices/scene_organization.html
- Godot Docs, "When to use scenes versus scripts" — https://docs.godotengine.org/en/stable/tutorials/best_practices/scenes_versus_scripts.html
- Godot Docs, "Project organization" — https://docs.godotengine.org/en/stable/tutorials/best_practices/project_organization.html
- Godot Docs, "Autoloads versus regular nodes" — https://docs.godotengine.org/en/stable/tutorials/best_practices/autoloads_versus_regular_nodes.html
- Godot Docs, "Autoloads (singletons)" — https://docs.godotengine.org/en/stable/tutorials/scripting/singletons_autoload.html
- Godot Docs, "Using InputEvent" — https://docs.godotengine.org/en/stable/tutorials/inputs/inputevent.html
- Godot Docs, "Thread-safe APIs" — https://docs.godotengine.org/en/stable/tutorials/performance/thread_safe_apis.html
- Godot Docs, "Using multiple threads" — https://docs.godotengine.org/en/stable/tutorials/performance/using_multiple_threads.html
- Godot Docs, "C# basics" (threading & lifecycle differences) — https://docs.godotengine.org/en/stable/tutorials/scripting/c_sharp/c_sharp_basics.html
- Godot Docs, "Resources" — https://docs.godotengine.org/en/stable/tutorials/scripting/resources.html
- Godot Docs, "Best Practices: Node alternatives" (composition guidance) — https://docs.godotengine.org/en/stable/tutorials/best_practices/node_alternatives.html
- Godot Docs, "Background loading" — https://docs.godotengine.org/en/stable/tutorials/io/background_loading.html
- Godot Docs, "Version control systems" — https://docs.godotengine.org/en/stable/tutorials/best_practices/version_control_systems.html
- GDScript Toolkit (gdformat/gdlint) — https://github.com/Scony/godot-gdscript-toolkit
- GUT (Godot Unit Test) — https://github.com/bitwes/Gut
- GdUnit4 — https://github.com/godot-gdunit-labs/gdUnit4
- godot-ci (headless CI images) — https://github.com/abarichello/godot-ci
- Sentry Godot SDK — https://docs.sentry.io/platforms/godot/
- Google, "Develop with Godot" (Android) — https://developer.android.com/games/engines/godot/godot-configure
- Robert Nystrom, *Game Programming Patterns* (Component pattern) — https://gameprogrammingpatterns.com/component.html
- Gary Bernhardt, "Functional Core, Imperative Shell" — https://www.destroyallsoftware.com/screencasts/catalog/functional-core-imperative-shell
- GDQuest, GDScript guidelines (principles; syntax partially Godot 3-era) — https://gdquest.gitbook.io/gdquests-guidelines/godot-gdscript-guidelines
- Game Accessibility Guidelines — https://gameaccessibilityguidelines.com/
- Xbox Accessibility Guidelines — https://learn.microsoft.com/en-us/gaming/accessibility/guidelines
