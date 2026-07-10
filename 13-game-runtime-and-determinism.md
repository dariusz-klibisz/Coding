# Game Runtime Performance & Determinism

> Real-time game loops run under a hard per-frame deadline and, for simulations
> that must be replayed, resumed offline, or compared across machines
> (netcode, offline-progress catch-up, save/replay verification), must also
> produce **bit-identical results** given the same inputs. This file covers
> both: the frame-budget performance discipline all real-time engines share,
> and the determinism engineering that turns "runs fast" into "runs the same
> everywhere." Engine examples use Godot 4.x / GDScript (validated against
> Godot 4.7; see [`languages/gdscript.md`](languages/gdscript.md) for the
> language-level rules); the reasoning generalizes to any real-time engine.

**Rule ID prefix:** `GEN-GAME`

---

## Table of contents

1. [The game loop and the frame budget](#1-the-game-loop-and-the-frame-budget)
2. [Fixed timestep vs render frame](#2-fixed-timestep-vs-render-frame)
3. [Avoiding per-frame allocation](#3-avoiding-per-frame-allocation)
4. [Object pooling](#4-object-pooling)
5. [Draw calls, batching & shader stutter](#5-draw-calls-batching--shader-stutter)
6. [Mobile-specific constraints](#6-mobile-specific-constraints)
7. [Profiling workflow](#7-profiling-workflow)
8. [Determinism engineering](#8-determinism-engineering)
9. [Seeded PRNG discipline](#9-seeded-prng-discipline)
10. [Deterministic update ordering](#10-deterministic-update-ordering)
11. [Sim/presentation separation](#11-simpresentation-separation)
12. [Threading model for real-time engines](#12-threading-model-for-real-time-engines)
13. [Save/serialization for simulated state](#13-saveserialization-for-simulated-state)
14. [Anti-patterns](#14-anti-patterns)
15. [Quick checklist](#quick-checklist)
16. [References](#references)

---

## 1. The game loop and the frame budget

**`GEN-GAME-01` (MUST)** Treat the frame budget as a hard deadline, not a
target to average out. At 60 Hz, each frame has **~16.6 ms** to update
simulation, run physics, and render; at 30 Hz, ~33.3 ms. Missing the budget
doesn't degrade gracefully like a slow web request — it drops a frame, which
is directly perceptible as stutter/jank.

**Why:** Unlike most backend/web work (where "measure first" in
[performance](06-performance.md) §1 still applies), a game loop's success
criterion is a **percentile deadline met every frame**, not an average
throughput number. A profiler run that shows a good *average* frame time can
still ship a stuttery game if the *worst-case* frames blow the budget.

**`GEN-GAME-02` (SHOULD)** Budget subsystems explicitly (e.g. simulation X ms,
physics Y ms, rendering Z ms, UI W ms) once frame time gets tight, so a
regression is traceable to the subsystem that grew rather than diagnosed from
a single aggregate number.

**`GEN-GAME-31` (SHOULD)** Track **worst-case frame metrics in production**,
not just in the profiler: percentile frame times and a hitch/frozen-frame
count. These are also what stores measure — Android vitals flags *slow
rendering* against the 16 ms smooth-UI target and counts any frame over
**700 ms** as a *frozen frame* in the Play Console quality dashboard, which
affects store visibility. Instrument the game to log its own worst frames
(scene, entity count, subsystem timings) so a field report is actionable.

---

## 2. Fixed timestep vs render frame

**`GEN-GAME-03` (MUST)** Separate the **simulation update rate** (fixed
timestep — Godot's `_physics_process`) from the **render rate** (variable —
Godot's `_process`). Gameplay/physics/combat logic that reads elapsed time
must use the fixed step, not the variable render delta.

**Why:** A variable render delta means simulation math run in the render
callback produces a *different result depending on the machine's frame rate* —
a hit-chance roll, a DoT tick, or a cooldown countdown computed against a
render-frame `delta` will diverge between a 60 fps and a 144 fps machine, and
between a live run and a headless recompute that isn't rendering frames at
all. Decoupling the two — the classic "fix your timestep" pattern — makes
simulation results reproducible regardless of display frame rate, and lets
rendering **interpolate** between simulation states for smoothness without
touching simulation correctness (Godot ships physics interpolation for 2D
since 4.3 and 3D since 4.4).

**`GEN-GAME-04` (SHOULD)** When the fixed step must "catch up" after a stall
(loading hitch, OS scheduling pause, or a large elapsed-time recompute), cap
the number of catch-up steps per real frame rather than iterating an unbounded
number of fixed steps synchronously — an unbounded catch-up loop can itself
blow the frame budget trying to repay simulation debt in one frame. This is
the *physics spiral of death* the Godot docs name explicitly; the engine-level
cap is the **Max Physics Steps per Frame** project setting. For a genuinely
large gap (e.g. an idle game's "player was away 8 hours" offline-progress
calculation), run that as an explicit **batch/headless simulation pass**
decoupled from the live frame loop entirely, not as an oversized catch-up
inside `_physics_process`.

---

## 3. Avoiding per-frame allocation

**`GEN-GAME-05` (SHOULD)** Avoid allocating (heap objects, arrays, strings)
inside code that runs every frame or every physics tick. Builds on
[performance](06-performance.md) §4 (memory & allocation), sharpened for
real-time. The cost model depends on the runtime: **GDScript is
reference-counted, not garbage-collected** — there are no GC pauses, but
per-tick construction/destruction of objects and containers still costs
allocation, refcount traffic, and cache misses on exactly the frames that can
least afford them. **C# (and other GC-based scripting runtimes) add GC
pauses** on top: a collection triggered mid-frame is a frame-budget spike that
reads as a visible stutter even if the average allocation rate is low.

```gdscript
# Bad — allocates a new Array and a new Dictionary every physics tick
func _physics_process(delta: float) -> void:
    var nearby: Array = []
    for enemy in get_tree().get_nodes_in_group("enemies"):
        var info := {"node": enemy, "dist": global_position.distance_to(enemy.global_position)}
        nearby.append(info)

# Good — reuse pre-sized containers; avoid per-tick allocation in hot paths
var _nearby_buffer: Array[Node] = []

func _physics_process(delta: float) -> void:
    _nearby_buffer.clear()
    for enemy in get_tree().get_nodes_in_group("enemies"):
        _nearby_buffer.append(enemy)
```

**`GEN-GAME-06` (SHOULD)** Reuse buffers/collections across frames instead of
constructing new ones; prefer typed collections (`Array[Type]`, and on Godot
4.4+ `Dictionary[Key, Value]`) and preallocated capacity where the API
supports it — both reduce allocation churn and improve locality (builds on
[performance](06-performance.md) §5).

**`GEN-GAME-32` (CONSIDER)** For the hottest loops over large entity counts,
apply **data-oriented layout**: contiguous packed arrays of plain values
(`PackedFloat32Array`, struct-of-arrays layouts) iterated linearly, instead of
scattered per-object allocations behind references — cache behavior, not
instruction count, usually dominates such loops (Fabian, *Data-Oriented
Design*; Nystrom, "Data Locality"). In Godot terms this is the escalation path
node-per-entity → `MultiMesh` → packed arrays + Servers API
(`GD-PERF-03` in [`languages/gdscript.md`](languages/gdscript.md)).

---

## 4. Object pooling

**`GEN-GAME-07` (SHOULD)** **Pool** objects that are created and destroyed at
high frequency during gameplay — projectiles, hit-VFX, floating damage
numbers, short-lived status-effect icons. Pre-instantiate a fixed (or
growable-with-ceiling) pool at load time; "spawn" means pulling an inactive
instance and resetting its state, "despawn" means deactivating and returning
it to the pool, not destroying the object.

**Why:** Instantiating and freeing a scene/object every time an entity is hit
is exactly the per-frame-allocation problem in §3, amplified — combat-heavy
games produce their highest spawn rates (many hits per second across many
units) at exactly the moment frame budget is already under the most pressure,
so unpooled VFX/damage-numbers are a likely place a frame-budget spike shows
up.

**Pros:** Eliminates allocation/deallocation churn at the exact moment frame
budget is scarcest.
**Cons:** Pool sizing requires tuning (too small → visible pop-in/fallback
allocation under burst load; too large → wasted memory, worse on mobile);
pooled objects must fully reset transient state on reuse or bugs leak between
uses (a pooled damage-number label that doesn't reset its color/fade-timer
shows stale visuals).

**`GEN-GAME-08` (SHOULD)** Size pools from **observed peak concurrency**
(profile a worst-case scene, not a guess), and have a defined fallback
(grow the pool, or drop the lowest-priority effect) for the rare case load
exceeds the pool — silently doing nothing is a worse failure mode than a
capped, deliberate degradation.

---

## 5. Draw calls, batching & shader stutter

**`GEN-GAME-09` (SHOULD)** Minimize draw calls for 2D scenes with many sprites
by batching: share materials/textures across similar sprites (atlas
texturing), use `MultiMeshInstance2D` or the engine's automatic 2D batching
(available on all Godot renderers since 4.4; previously Compatibility-only)
for large counts of visually-identical instances, and avoid unnecessary
per-object unique materials/shaders that break batching.

**`GEN-GAME-10` (CONSIDER)** Profile draw-call count directly (Godot's
Monitor tab / `RenderingServer` frame stats) when frame time is GPU-bound
rather than guessing from sprite count — the number of *state changes*
(material/texture switches), not raw sprite count, is usually the actual
cost driver.

**`GEN-GAME-33` (SHOULD)** Treat **shader/pipeline compilation stutter** as a
frame-budget problem with engine-level fixes: first-use compilation of a
shader pipeline can freeze a frame for tens/hundreds of ms. On Godot 4.4+,
ubershaders precompile fallback pipelines automatically; on 4.5+, enable the
**shader baker** in export settings to precompile shaders for the target
driver at export time (the official docs page "Reducing stutter from shader
compilations" covers the remaining cases). If a hitch appears the first time
an effect/enemy type shows up, suspect pipeline compilation before script
cost.

---

## 6. Mobile-specific constraints

**`GEN-GAME-11` (MUST)** Treat mobile as a **stricter** frame-budget and
memory-budget environment than desktop, not the same budget on smaller
hardware: thermal throttling reduces sustained clock speed the longer a heavy
scene runs (a scene that's fine for the first 10 seconds can degrade when it
runs continuously for minutes — directly relevant to auto-battle/idle games
where combat runs most of the session), battery drain is a user-visible cost
of sustained CPU/GPU load, and available memory is a hard ceiling with no
swap headroom comparable to desktop.

**`GEN-GAME-12` (SHOULD)** Test performance on **median-tier mobile hardware**,
not a flagship device or desktop-in-a-mobile-shell — a scene that holds
60 fps on a developer's current flagship phone can thermal-throttle to
unplayable on the hardware most of the install base actually runs. Budget for
sustained (not burst) performance for long-session games.

**`GEN-GAME-13` (SHOULD)** Reduce simulation/render work when the app is
backgrounded on mobile (the OS will suspend it regardless, but resuming
should recompute via an offline-catch-up batch pass — §2 — not attempt to
"replay" every backgrounded frame).

**`GEN-GAME-34` (SHOULD)** Manage sustained mobile performance **actively**
rather than letting the OS throttle you:

- Cap frame rate deliberately (`Engine.max_fps`; for idle/low-interaction
  scenes also `OS.low_processor_usage_mode`) — running an idle screen at an
  uncapped frame rate burns battery for nothing.
- Choose the renderer for the target tier (Godot's **Mobile** renderer for
  modern devices, **Compatibility** for low-end/GLES3), and rely on the
  automatic frame pacing Godot 4.4+ gets from Android's Swappy library.
- For sustained-load games, adapt quality to **thermal state** using the
  Android Dynamic Performance Framework (ADPF thermal-headroom and
  performance-hint APIs, via a plugin/GDExtension): drop resolution scale,
  effect density, or tick rate in defined tiers as headroom shrinks — a
  controlled quality step-down beats an OS-imposed clock throttle. Google's
  official "Develop with Godot" Android guidance covers renderer and export
  configuration.

---

## 7. Profiling workflow

**`GEN-GAME-14` (SHOULD)** Use the engine's built-in profiler
(Godot's Debugger → Profiler, plus the Monitor tab for draw calls/object
counts/memory) as the primary tool, not guesswork — builds on
[performance](06-performance.md) §3, sharpened: a frame-time profiler that
breaks down time **per subsystem per frame** is required to diagnose which
specific frames blow budget and why, not just aggregate hot functions. Enable
profiler **autostart** (4.4+) so startup/loading frames are captured too.

**`GEN-GAME-15` (SHOULD)** Capture profiles under the game's actual
worst-case scenario — the maximum concurrent entities, effects, and AI/logic
evaluation the design allows (e.g. a full squad-vs-squad fight with every
status effect active), not an idle or single-unit scene. Combat-heavy games
have their worst frame times exactly when the most is happening on screen.

**`GEN-GAME-35` (CONSIDER)** For spikes the built-in profiler can't explain,
use **trace-based profilers** — Godot 4.6+ supports Tracy, Perfetto, and
Instruments instrumentation, and Perfetto is the default tracing path on
Android (4.7+) — and the debugger's **ObjectDB snapshot diffing** (4.6+) to
find object/memory growth between two points in time. Trace timelines show
*when within a frame* time went and what blocked (thread scheduling, GPU
sync), which sampling profilers hide.

---

## 8. Determinism engineering

**`GEN-GAME-16` (MUST)** Treat determinism as a **design constraint enforced
from the first line of simulation code**, not a property to retrofit. A
simulation is deterministic when the same initial state + the same ordered
sequence of inputs always produces bit-identical output, on any machine, any
number of times.

**Why this is load-bearing:** any feature where the *same* simulation must run
live on-screen, be recomputed headless (offline-progress catch-up), or be
replayed for verification (async PvP fairness, replays, lockstep multiplayer)
breaks if two of those paths can diverge even slightly — offline rewards stop
matching what "would have happened," and PvP becomes exploitable (a client
submits a favorable result that a verifying re-simulation disagrees with).
Retrofitting determinism into a simulation that grew non-deterministic habits
(floating-point order-dependence, unseeded randomness,
iteration-order-dependent collections) is a rewrite, not a patch — enforce it
from day one of the simulation kernel. This is the discipline behind the
deterministic-lockstep lineage from *Age of Empires*' "1500 Archers" through
Factorio's inputs-only multiplayer.

**`GEN-GAME-17` (SHOULD)** Prefer **integer/fixed-point math** for values that
must reconcile bit-for-bit across machines; treat cross-platform float
determinism as an unverified risk until proven. Floating-point arithmetic is
**not** associative (`(a + b) + c` can differ from `a + (b + c)` at the bit
level), so the same formula evaluated in a different order — which can happen
silently via compiler/engine optimization differences across platforms, or via
unordered iteration (§10) — can diverge. This is why commercial deterministic
engines (e.g. Photon Quantum) standardize on fixed-point math rather than
floats. Godot-specific precision facts: GDScript's scalar `float` is 64-bit,
but `Vector2/3/4` components are **32-bit** unless the engine is custom-built
with `precision=double` — mixing the two truncates silently. If float math is
used on a reconciliation-critical path, constrain it to a single,
explicitly-ordered evaluation path and verify identical results on every
target platform.

**`GEN-GAME-36` (MUST NOT)** Don't feed **engine-physics results** (contact
points, solver-resolved velocities/positions, physics queries against
solver-owned state) into a simulation that must be reproducible. A fixed tick
makes physics frame-rate-independent, **not** deterministic: neither
GodotPhysics nor Jolt (Godot 4.6+'s default 3D engine) guarantees
bit-identical results across platforms/builds, and results can also change
with engine upgrades. Keep the deterministic kernel's math in your own
integer/ordered code (§11); use engine physics for presentation and for
gameplay that doesn't need to reconcile.

**`GEN-GAME-37` (SHOULD)** Verify competitive/economy-relevant results by
**deterministic re-simulation**, not by trusting reported outcomes: persist
the initial state, seed(s), and the ordered input/command log; a verifier (the
server, or the opponent's client in async PvP) replays the log through the
same kernel and accepts the result only if it matches. This
inputs-plus-determinism model is the industry-standard integrity mechanism of
lockstep RTS multiplayer ("1500 Archers", Factorio) and modern deterministic
engines — it turns "never trust the client" into a cheap equality check
instead of server-side heuristics.

---

## 9. Seeded PRNG discipline

**`GEN-GAME-18` (MUST)** Use an explicit, seeded pseudo-random number
generator for **all** simulation randomness (hit rolls, crit rolls, loot
rolls, procedural generation) — never the engine's global/unseeded RNG for
anything that must be reproducible. Godot's `RandomNumberGenerator` class
supports explicit `seed` — use an instance with a project-controlled seed, not
`randi()`'s global default state, for simulation-affecting rolls.

**`GEN-GAME-19` (MUST)** Keep RNG **decoupled by concern** so one system's
draw count can't shift another's sequence. Two accepted patterns:

- **Split sequential streams:** a distinct seeded stream per independent
  random concern (combat rolls vs loot rolls vs procedural generation). With
  one shared stream, adding a single extra hit-roll silently reseeds every
  subsequent loot roll — *technically* deterministic but unmaintainable, since
  any balance tweak to one system scrambles every other system's results.
- **Stateless noise-based RNG (state of the art):** derive each roll by
  hashing `(seed, tick, entity_id, purpose)` with a strong mixing function
  instead of advancing shared state (Eiserloh, GDC "Noise-Based RNG").
  Order-independent by construction, trivially parallelizable, and
  random-access — any roll can be recomputed in isolation, which also
  simplifies replay debugging.

**`GEN-GAME-20` (SHOULD)** Persist the seed (and enough state to resume the
stream) alongside any result that must be independently reproducible later —
an offline-catch-up recompute or a replay verification needs the *exact* seed
used, not a freshly-drawn one, or "replaying" just produces a different,
unrelated outcome.

---

## 10. Deterministic update ordering

**`GEN-GAME-21` (MUST)** Iterate collections of simulated entities (units,
enemies, active effects) in a **stable, explicit order** every tick — board
position, an assigned index, or an explicit priority — never in
hash-map/dictionary/`Set`-native iteration order, which is not guaranteed
stable across engine versions, platforms, or even separate runs in some
implementations. Two entities resolving a simultaneous action in a different
order can produce a different outcome (e.g. two units at lethal-range killing
each other — who dies "first" can decide whether a passive that triggers
on-death fires).

**`GEN-GAME-22` (SHOULD)** Define an explicit **resolution order** for
simultaneous events within one tick (e.g. "status-effect ticks resolve in
board-position order, then queued actions resolve in priority order, then
death checks") and document it as part of the simulation-kernel contract —
this is a design decision (also relevant to gameplay balance/fairness), not
just an implementation detail, and it must be the same rule on every code path
that runs the sim (live, headless recompute, replay verification).

**`GEN-GAME-23` (SHOULD NOT)** Don't rely on multi-threaded parallel iteration
over simulated entities for the authoritative simulation result unless the
merge/combination step is itself deterministic and order-independent — thread
scheduling is not deterministic, so naively parallelizing "resolve all units'
actions this tick" across threads and merging results in whatever order
threads finish reintroduces exactly the non-determinism this section exists
to prevent. Parallelism for the sim is a performance optimization that must
preserve the ordering contract in `GEN-GAME-21`/`22`, not bypass it. (The
stateless-RNG pattern in `GEN-GAME-19` makes deterministic parallel resolution
substantially easier, since rolls no longer depend on draw order.)

---

## 11. Sim/presentation separation

**`GEN-GAME-24` (SHOULD)** Structure the simulation kernel as a **pure
simulation** — state in, ordered inputs in, new state + an event/result log
out — with no dependency on rendering, node lifecycle, or wall-clock time.
Presentation (sprites, VFX, damage-number popups, camera shake) **consumes**
the simulation's event log and animates it; it never feeds back into
simulation decisions.

**Why:** This is what makes the same kernel usable for on-screen live play,
headless recomputation (no rendering at all), and replay verification
(potentially on a server with no scene tree). A kernel entangled with `Node`
lifecycle or frame-rate-dependent state can only run inside a live, rendering
engine instance — exactly the coupling that blocks offline calculation and
server-side verification. This is "functional core, imperative shell"
(Bernhardt) applied to a game loop, combined with the Command pattern for the
input log (Nystrom).

**`GEN-GAME-25` (SHOULD)** Feed the presentation layer from the simulation's
**event log**, not by having presentation code re-derive "what happened" from
before/after state diffs — an explicit event (`DamageDealt(source, target,
amount, type, was_crit)`) is unambiguous and animatable; a state diff can be
ambiguous about *why* a value changed when multiple effects resolve in the
same tick.

---

## 12. Threading model for real-time engines

Builds on [concurrency](07-concurrency.md). Godot-specific: the scene tree is
main-thread-affine (`GD-CONC-01` in [`languages/gdscript.md`](languages/gdscript.md)).

**`GEN-GAME-26` (SHOULD)** Run the deterministic simulation kernel (§11) so it
*can* execute off the main thread / headless (no scene-tree touch at all) —
this is what makes background `WorkerThreadPool` offline-catch-up computation
and server-side verification possible without duplicating sim logic for a
"server build."

**`GEN-GAME-27` (SHOULD)** Marshal simulation results back to the main thread
explicitly (`call_deferred`/`call_thread_safe`) before touching any node —
never let a background-thread simulation task reach into the scene tree
directly (`GD-CONC-01`).

---

## 13. Save/serialization for simulated state

**`GEN-GAME-28` (SHOULD)** Version every persisted save/state format
explicitly (a `save_version` field) from the first shipped save, and write an
explicit migration path for each version bump — a data-driven, frequently-
patched game (new classes, new items, new mechanics) will change its save
shape repeatedly, and an unversioned save format makes every future content
patch a potential corruption risk for existing players' saves.

**`GEN-GAME-29` (SHOULD)** Validate a loaded save's values before they reach
gameplay math (clamp/reject out-of-range stats, unknown enum values from a
newer format loaded by an older build, etc.) — builds on
[defensive programming](02-defensive-programming.md) and on `GD-SEC-02` in
the GDScript language doc; a save file is an external input even when the
player who edited it is also the one who benefits, which matters wherever
save integrity affects shared fairness (leaderboards, async PvP, trading).

**`GEN-GAME-30` (CONSIDER)** Where save integrity matters for **fairness**
(leaderboards, async PvP, achievement legitimacy) more than for security in
the adversarial-attacker sense, favor **tamper-evidence** (checksum/signature
that detects an edited save) over heavyweight DRM — proportionate to what's
actually at stake (`GD-SEC-03`). For competitive results specifically, replay
verification (`GEN-GAME-37`) is stronger than any client-side tamper seal.

---

## 14. Anti-patterns

- Simulation-affecting math computed from `_process`'s variable render delta
  instead of a fixed timestep.
- Allocating new collections/objects inside a per-frame or per-physics-tick
  hot path.
- Spawning and freeing scenes/objects per gameplay event instead of pooling.
- Assuming desktop-tier sustained performance on mobile without testing on
  median-tier, thermally-throttled hardware.
- Running uncapped frame rates in idle/low-interaction scenes on battery-
  powered devices.
- Using the engine's global/unseeded RNG for any roll that must be
  reproducible (loot, crit, procedural generation).
- One shared RNG stream across unrelated systems, so a balance change to one
  system silently reseeds another.
- Iterating simulated entities in hash-map/native collection order instead of
  an explicit, stable order.
- Feeding engine-physics solver results into a simulation that must
  reconcile bit-for-bit across machines.
- Parallelizing per-tick entity resolution without a deterministic merge step.
- Kernel code that reads `Node`/scene-tree state directly, coupling it to a
  live rendering instance and blocking headless/offline/server reuse.
- Presentation code re-deriving "what happened" from state diffs instead of
  consuming an explicit event log.
- Trusting client-reported competitive results instead of verifying by
  deterministic re-simulation.
- An unversioned save/state format with no migration path.

---

## Quick checklist

- [ ] Frame-budget-critical code measured against the worst-case frame, not
      the average; worst-case metrics tracked in production (`GEN-GAME-01/02/31`).
- [ ] Simulation/gameplay math runs on a fixed timestep, decoupled from
      render-frame `delta` (`GEN-GAME-03`).
- [ ] Catch-up after a stall is bounded per live frame (Max Physics Steps per
      Frame); large offline gaps run as an explicit batch/headless pass
      (`GEN-GAME-04`).
- [ ] No allocation inside per-frame/per-tick hot paths; buffers reused; typed
      collections; data-oriented layout considered for the hottest loops
      (`GEN-GAME-05/06/32`).
- [ ] High-frequency spawns (VFX, damage numbers, projectiles) pooled and
      sized from observed peak concurrency (`GEN-GAME-07/08`).
- [ ] Draw calls batched (shared materials/atlases); profiled, not guessed;
      shader-compilation stutter mitigated (ubershaders/shader baker)
      (`GEN-GAME-09/10/33`).
- [ ] Performance validated on median-tier mobile hardware under sustained
      load; frame rate capped and quality adapted to thermal state
      (`GEN-GAME-11/12/34`).
- [ ] Worst-case scenario (max entities, effects, AI evaluation) profiled
      explicitly; trace-based profilers used for unexplained spikes
      (`GEN-GAME-15/35`).
- [ ] All simulation randomness uses an explicit, seeded RNG — never the
      engine's global RNG (`GEN-GAME-18`).
- [ ] RNG decoupled by concern (split streams or stateless noise-based);
      seeds persisted for reproducibility (`GEN-GAME-19/20`).
- [ ] Entity iteration order is explicit and stable every tick; simultaneous-
      event resolution order is defined and documented (`GEN-GAME-21/22`).
- [ ] Any parallelism over per-tick entity resolution preserves deterministic
      ordering via its merge step (`GEN-GAME-23`).
- [ ] Reconciliation-critical math is integer/fixed-point (or verified
      single-path float); no engine-physics results feed the deterministic
      kernel (`GEN-GAME-17/36`).
- [ ] Simulation kernel is a pure sim (state + ordered inputs → state + event
      log) with no scene-tree/render dependency (`GEN-GAME-24/25`).
- [ ] Competitive/economy results verified by deterministic re-simulation of
      the input log, not trusted from the client (`GEN-GAME-37`).
- [ ] Background-thread sim work marshals results to the main thread before
      touching nodes (`GEN-GAME-26/27`).
- [ ] Save/state format is versioned with a migration path; loaded values are
      validated before reaching gameplay math (`GEN-GAME-28/29`).

---

## References

- Glenn Fiedler, "Fix Your Timestep!" — https://gafferongames.com/post/fix_your_timestep/
- Glenn Fiedler, "Deterministic Lockstep" — https://gafferongames.com/post/deterministic_lockstep/
- Glenn Fiedler, "Floating Point Determinism" — https://gafferongames.com/post/floating_point_determinism/
- Paul Bettner & Mark Terrano, "1500 Archers on a 28.8: Network Programming in Age of Empires and Beyond" — https://www.gamedeveloper.com/programming/1500-archers-on-a-28-8-network-programming-in-age-of-empires-and-beyond
- Factorio, "Friday Facts #302 — The multiplayer megapacket" (deterministic lockstep at scale) — https://www.factorio.com/blog/post/fff-302
- Photon Quantum (fixed-point deterministic engine; industry reference) — https://doc.photonengine.com/quantum/current/getting-started/quantum-intro
- Squirrel Eiserloh, GDC "Math for Game Programmers: Noise-Based RNG" — https://gdcvault.com/play/1024365/Math-for-Game-Programmers-Noise
- Robert Nystrom, *Game Programming Patterns* — Game Loop, Object Pool, Update
  Method, Data Locality, Command chapters — https://gameprogrammingpatterns.com/
- Richard Fabian, *Data-Oriented Design* — https://www.dataorienteddesign.com/dodmain/
- Gary Bernhardt, "Functional Core, Imperative Shell" — https://www.destroyallsoftware.com/screencasts/catalog/functional-core-imperative-shell
- Alan Wolfe (demofox), "Demystifying Floating Point Precision" — https://blog.demofox.org/2017/11/21/floating-point-precision/
- Godot Docs, "Optimization: General" and "GPU optimization" — https://docs.godotengine.org/en/stable/tutorials/performance/general_optimization.html
- Godot Docs, "Physics Interpolation" — https://docs.godotengine.org/en/stable/tutorials/physics/interpolation/index.html
- Godot Docs, "Troubleshooting physics issues" (spiral of death, tick rate) — https://docs.godotengine.org/en/stable/tutorials/physics/troubleshooting_physics_issues.html
- Godot Docs, "Using Jolt Physics" — https://docs.godotengine.org/en/stable/tutorials/physics/using_jolt_physics.html
- Godot Docs, "Large world coordinates" (float precision facts) — https://docs.godotengine.org/en/stable/tutorials/physics/large_world_coordinates.html
- Godot Docs, "Reducing stutter from shader (pipeline) compilations" — https://docs.godotengine.org/en/stable/tutorials/performance/pipeline_compilations.html
- Godot Docs, "Using multiple threads" — https://docs.godotengine.org/en/stable/tutorials/performance/using_multiple_threads.html
- Android vitals, "Slow rendering" (16 ms target; frozen frames > 700 ms) — https://developer.android.com/topic/performance/vitals/render
- Android Dynamic Performance Framework (thermal & performance-hint APIs) — https://developer.android.com/games/optimize/adpf
- Google, "Develop with Godot" (Android renderer/export guidance) — https://developer.android.com/games/engines/godot/godot-configure
- IEEE 754 floating-point standard (non-associativity of FP addition).
