# Game Runtime Performance & Determinism

> Real-time game loops run under a hard per-frame deadline and, for simulations
> that must be replayed, resumed offline, or compared across machines
> (netcode, offline-catch-up, save/replay), must also produce **bit-identical
> results** given the same inputs. This file covers both: the frame-budget
> performance discipline all real-time engines share, and the determinism
> engineering that turns "runs fast" into "runs the same everywhere." Engine
> examples use Godot/GDScript (see [`languages/gdscript.md`](languages/gdscript.md)
> for the language-level rules); the reasoning generalizes to any real-time
> engine.

**Rule ID prefix:** `GEN-GAME`

---

## Table of contents

1. [The game loop and the frame budget](#1-the-game-loop-and-the-frame-budget)
2. [Fixed timestep vs render frame](#2-fixed-timestep-vs-render-frame)
3. [Avoiding per-frame allocation](#3-avoiding-per-frame-allocation)
4. [Object pooling](#4-object-pooling)
5. [Draw calls & batching](#5-draw-calls--batching)
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
between a live run and an offline-catch-up recompute that isn't rendering
frames at all. Decoupling the two — a classic "fix your timestep" pattern —
makes simulation results reproducible regardless of display frame rate, and
lets rendering **interpolate** between simulation states for smoothness
without touching simulation correctness.

**`GEN-GAME-04` (SHOULD)** When the fixed step must "catch up" after a stall
(loading hitch, OS scheduling pause, or a large offline-elapsed-time
recompute), cap the number of catch-up steps per real frame ("spiral of
death" guard) rather than iterating an unbounded number of fixed steps
synchronously — an unbounded catch-up loop can itself blow the frame budget
trying to repay simulation debt in one frame. For a genuinely large gap (the
game's own offline-progress calculation, e.g. "player was away 8 hours"), run
that as an explicit **batch/headless simulation pass** decoupled from the live
frame loop entirely, not as an oversized catch-up inside `_physics_process`.

---

## 3. Avoiding per-frame allocation

**`GEN-GAME-05` (SHOULD)** Avoid allocating (heap objects, arrays, strings)
inside code that runs every frame or every physics tick. Builds on
[performance](06-performance.md) §4 (memory & allocation), sharpened for
real-time: a managed-runtime garbage collector (GDScript, C#, most scripting
VMs) that triggers mid-frame is a **frame-budget spike**, not just aggregate
overhead — a single GC pause can blow the 16.6 ms budget and read as a visible
stutter even if average allocation rate is low.

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
constructing new ones; prefer typed arrays (`Array[Type]`) and preallocated
capacity where the API supports it — both reduce allocation churn and improve
locality (builds on [performance](06-performance.md) §5).

---

## 4. Object pooling

**`GEN-GAME-07` (SHOULD)** **Pool** objects that are created and destroyed at
high frequency during gameplay — projectiles, hit-VFX, floating damage
numbers, short-lived status-effect icons. Pre-instantiate a fixed (or
growable-with-ceiling) pool at load time; "spawn" means pulling an inactive
instance and resetting its state, "despawn" means deactivating and returning
it to the pool, not destroying the object.

**Why:** Instantiating and freeing a scene/object every time a monster is hit
is exactly the per-frame-allocation problem in §3, amplified — combat is the
highest-frequency spawn point in this game's design (many hits per second
across a full squad fight), so unpooled VFX/damage-numbers is the most likely
place a frame-budget spike shows up.

**Pros:** Eliminates allocation/deallocation churn at the exact moment
(intense combat) frame budget is already under the most pressure.
**Cons:** Pool sizing requires tuning (too small → visible pop-in/fallback
allocation under burst load; too large → wasted memory, worse on mobile);
pooled objects must fully reset transient state on reuse or bugs leak between
uses (a pooled damage-number label that doesn't reset its color/fade-timer
shows stale visuals).

**`GEN-GAME-08` (SHOULD)** Size pools from **observed peak concurrency**
(profile a worst-case fight, not a guess), and have a defined fallback
(grow the pool, or drop the lowest-priority effect) for the rare case load
exceeds the pool — silently doing nothing is a worse failure mode than a
capped, deliberate degradation.

---

## 5. Draw calls & batching

**`GEN-GAME-09` (SHOULD)** Minimize draw calls for a 2D stylized/iconographic
combat scene by batching: share materials/textures across similar sprites
(atlas texturing), use `MultiMeshInstance2D` or Godot's automatic 2D batching
for large counts of visually-identical instances (e.g. many small icon-style
enemies, particle-like effects), and avoid unnecessary per-object unique
materials/shaders that break batching.

**`GEN-GAME-10` (CONSIDER)** Profile draw-call count directly (Godot's
Monitor tab / `RenderingServer` frame stats) when frame time is GPU-bound
rather than guessing from sprite count — the number of *state changes*
(material/texture switches), not raw sprite count, is usually the actual
cost driver.

---

## 6. Mobile-specific constraints

**`GEN-GAME-11` (MUST)** Treat mobile as a **stricter** frame-budget and
memory-budget environment than desktop, not the same budget on smaller
hardware: thermal throttling reduces sustained clock speed the longer a heavy
scene runs (a fight that's fine for the first 10 seconds can degrade if
combat runs continuously for minutes — directly relevant to an AFK/auto-battle
game), battery drain is a user-visible cost of sustained CPU/GPU load, and
available memory is a hard ceiling with no swap headroom comparable to
desktop.

**`GEN-GAME-12` (SHOULD)** Test performance on **median-tier mobile hardware**,
not a flagship device or desktop-in-a-mobile-shell — a combat scene that holds
60 fps on a developer's current flagship phone can thermal-throttle to
unplayable on the hardware most of the install base actually runs. Budget for
sustained (not burst) performance given this game's AFK/long-session
real-time-combat model.

**`GEN-GAME-13` (SHOULD)** Reduce simulation/render work when the app is
backgrounded on mobile (the OS will suspend it regardless, but resuming
should recompute via the offline-catch-up batch pass — §2 — not attempt to
"replay" every backgrounded frame).

---

## 7. Profiling workflow

**`GEN-GAME-14` (SHOULD)** Use the engine's built-in profiler
(Godot's Debugger → Profiler, plus the Monitor tab for draw calls/object
counts/memory) as the primary tool, not guesswork — builds on
[performance](06-performance.md) §3, sharpened: a frame-time profiler that
breaks down time **per subsystem per frame** is required to diagnose which
specific frames blow budget and why, not just aggregate hot functions.

**`GEN-GAME-15` (SHOULD)** Capture profiles under the actual worst-case
scenario this game produces — a full squad (grid-max heroes) fighting a full
enemy squad with multiple simultaneous status effects, AoE skills, and an
active Tactics/automation rule evaluation pass — not an idle or single-unit
scene. Combat-heavy idle games have their worst frame times exactly when the
most is happening on screen, which for an "always auto-battling" game can be
most of the session.

---

## 8. Determinism engineering

**`GEN-GAME-16` (MUST)** Treat determinism as a **design constraint enforced
from the first line of simulation code**, not a property to retrofit. A
simulation is deterministic when the same initial state + the same ordered
sequence of inputs always produces bit-identical output, on any machine, any
number of times.

**Why this is load-bearing for this project specifically:** the feature spec
requires the *same* combat sim to run live on-screen, recomputed for
offline-progress catch-up, and replayed for async PvP fairness/anti-cheat.
If two of those three paths can diverge even slightly, offline rewards stop
matching what "would have happened," and PvP becomes exploitable (an attacker
replays a favorable-looking client-side result that the server/opponent's
replay disagrees with). Retrofitting determinism into a simulation that grew
non-deterministic habits (floating-point order-dependence, unseeded
randomness, iteration-order-dependent collections) is a rewrite, not a patch —
enforce it from day one of the combat kernel.

**`GEN-GAME-17` (SHOULD)** Prefer **fixed-point or carefully-ordered
double/float arithmetic** for anything whose result must match bit-for-bit
across machines. Floating-point arithmetic is **not** associative
(`(a + b) + c` can differ from `a + (b + c)` at the bit level), so the same
formula evaluated in a different order — which can happen silently via
compiler/engine optimization differences across platforms, or via unordered
iteration (§10) — can diverge. Where cross-platform bit-identical results are
required (this project's offline-catch-up + async-PvP requirement), either:
use fixed-point (integer) math for the values that must reconcile exactly, or
constrain float math to a single, explicitly-ordered evaluation path and
accept/verify that the target platforms' floating-point behavior is
consistent enough for the required tolerance.

---

## 9. Seeded PRNG discipline

**`GEN-GAME-18` (MUST)** Use an explicit, seeded pseudo-random number
generator for **all** simulation randomness (hit rolls, crit rolls, loot
rolls, affix rolls, procedural rift generation, procedural hero recruitment)
— never the engine's global/unseeded RNG for anything that must be
reproducible. Godot's `RandomNumberGenerator` class supports explicit
`seed` — use an instance with a project-controlled seed, not `randi()`'s
global default state, for simulation-affecting rolls.

**`GEN-GAME-19` (MUST)** Keep RNG **streams separated by concern** — a
distinct seeded stream (or clearly-defined draw order from one stream) per
independent random concern (combat rolls vs loot rolls vs procedural
generation) so that changing one system's random draw count doesn't shift the
sequence every other system consumes. A single shared stream where, say,
adding one more hit-roll shifts every subsequent loot roll makes the sim
*technically* deterministic but practically unmaintainable — any balance
tweak to one system silently reseeds every other system's results for that
run.

**`GEN-GAME-20` (SHOULD)** Persist the seed (and enough state to resume the
stream) alongside any result that must be independently reproducible later —
an offline-catch-up recompute or an async-PvP replay needs the *exact* seed
used, not a freshly-drawn one, or "replaying" the fight just produces a
different, unrelated outcome.

---

## 10. Deterministic update ordering

**`GEN-GAME-21` (MUST)** Iterate collections of simulated entities (heroes,
enemies, active effects) in a **stable, explicit order** every tick — grid
position, an assigned index, or an explicit priority — never in
hash-map/dictionary/`Set`-native iteration order, which is not guaranteed
stable across engine versions, platforms, or even separate runs in some
implementations. Two entities resolving a simultaneous action in a different
order can produce a different outcome (e.g. two units at lethal-range killing
each other — who dies "first" can decide whether a passive that triggers
on-death fires).

**`GEN-GAME-22` (SHOULD)** Define an explicit **resolution order** for
simultaneous events within one tick (e.g. "status-effect ticks resolve in
grid-position order, then queued skill casts resolve in cast-priority order,
then death checks") and document it as part of the combat-kernel contract —
this is a design decision (also relevant to gameplay balance/fairness), not
just an implementation detail, and it must be the same rule on every code path
that runs the sim (live, offline-catch-up, PvP replay).

**`GEN-GAME-23` (SHOULD NOT)** Don't rely on multi-threaded parallel iteration
over simulated entities for the authoritative simulation result unless the
merge/combination step is itself deterministic and order-independent — thread
scheduling is not deterministic, so naively parallelizing "resolve all units'
actions this tick" across threads and merging results in whatever order
threads finish reintroduces exactly the non-determinism this section exists
to prevent. Parallelism for the sim is a performance optimization that must
preserve the ordering contract in `GEN-GAME-21`/`22`, not bypass it.

---

## 11. Sim/presentation separation

**`GEN-GAME-24` (SHOULD)** Structure the combat kernel as a **pure
simulation** — state in, ordered inputs in, new state + an event/result log
out — with no dependency on rendering, node lifecycle, or wall-clock time.
Presentation (sprites, VFX, damage-number popups, camera shake) **consumes**
the simulation's event log and animates it; it never feeds back into
simulation decisions.

**Why:** This is what makes the same kernel usable for on-screen live combat,
headless offline-catch-up (no rendering at all), and async-PvP replay
(potentially on a server with no scene tree). A kernel entangled with `Node`
lifecycle or frame-rate-dependent state can only run inside a live, rendering
Godot instance — exactly the coupling that would block offline calc and
server-side PvP verification. See the tech-agnostic architectural framing
(fixed-tick pipeline, command pattern for replay) in the Design library's
[`10-game-architecture.md`](../Design/10-game-architecture.md).

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

**`GEN-GAME-26` (SHOULD)** Run the deterministic combat kernel (§11) so it
*can* execute off the main thread / headless (no scene-tree touch at all) —
this is what makes background `WorkerThreadPool` offline-catch-up computation
and, later, server-side PvP verification possible without duplicating sim
logic for a "server build."

**`GEN-GAME-27` (SHOULD)** Marshal simulation results back to the main thread
explicitly (`call_deferred`/`call_thread_safe`) before touching any node —
never let a background-thread simulation task reach into the scene tree
directly (`GD-CONC-01`).

---

## 13. Save/serialization for simulated state

**`GEN-GAME-28` (SHOULD)** Version every persisted save/state format
explicitly (a `save_version` field) from the first shipped save, and write an
explicit migration path for each version bump — a data-driven, frequently-
patched game (new classes, new affixes, new item slots) will change its save
shape repeatedly, and an unversioned save format makes every future content
patch a potential corruption risk for existing players' saves.

**`GEN-GAME-29` (SHOULD)** Validate a loaded save's values before they reach
gameplay math (clamp/reject out-of-range stats, unknown enum values from a
newer format loaded by an older build, etc.) — builds on
[defensive programming](02-defensive-programming.md) and on `GD-SEC-02` in
the GDScript language doc; a save file is an external input even though the
player who edited it is also the one who benefits, which matters for the
no-pay-to-win economy's integrity goals.

**`GEN-GAME-30` (CONSIDER)** For a no-P2W economy where save integrity matters
for fairness (async PvP leaderboards, achievement legitimacy) more than for
security in the adversarial-attacker sense, favor **tamper-evidence**
(checksum/signature that detects an edited save) over heavyweight DRM —
proportionate to what's actually at stake (`GD-SEC-03`).

---

## 14. Anti-patterns

- Simulation-affecting math computed from `_process`'s variable render delta
  instead of a fixed timestep.
- Allocating new collections/objects inside a per-frame or per-physics-tick
  hot path.
- Spawning and freeing scenes/objects per combat event instead of pooling.
- Assuming desktop-tier sustained performance on mobile without testing on
  median-tier, thermally-throttled hardware.
- Using the engine's global/unseeded RNG for any roll that must be
  reproducible (loot, crit, procedural generation).
- One shared RNG stream across unrelated systems, so a balance change to one
  system silently reseeds another.
- Iterating simulated entities in hash-map/native collection order instead of
  an explicit, stable order.
- Parallelizing per-tick entity resolution without a deterministic merge step.
- Combat-kernel code that reads `Node`/scene-tree state directly, coupling it
  to a live rendering instance and blocking headless/offline/server reuse.
- Presentation code re-deriving "what happened" from state diffs instead of
  consuming an explicit event log.
- An unversioned save/state format with no migration path.

---

## Quick checklist

- [ ] Frame-budget-critical code measured against the worst-case frame, not
      the average (`GEN-GAME-01/02`).
- [ ] Simulation/gameplay math runs on a fixed timestep, decoupled from
      render-frame `delta` (`GEN-GAME-03`).
- [ ] Catch-up after a stall is bounded per live frame; large offline gaps run
      as an explicit batch/headless pass (`GEN-GAME-04`).
- [ ] No allocation inside per-frame/per-tick hot paths; buffers reused
      (`GEN-GAME-05/06`).
- [ ] High-frequency combat spawns (VFX, damage numbers, projectiles) pooled
      and sized from observed peak concurrency (`GEN-GAME-07/08`).
- [ ] Draw calls batched (shared materials/atlases); profiled, not guessed
      (`GEN-GAME-09/10`).
- [ ] Performance validated on median-tier mobile hardware under sustained
      combat load, not just desktop/flagship (`GEN-GAME-11/12`).
- [ ] Worst-case combat scenario (full squad, full effects, automation rules
      active) profiled explicitly (`GEN-GAME-15`).
- [ ] All simulation randomness uses an explicit, seeded RNG — never the
      engine's global RNG (`GEN-GAME-18`).
- [ ] RNG streams separated by concern; seeds persisted for reproducibility
      (`GEN-GAME-19/20`).
- [ ] Entity iteration order is explicit and stable every tick; simultaneous-
      event resolution order is defined and documented (`GEN-GAME-21/22`).
- [ ] Any parallelism over per-tick entity resolution preserves deterministic
      ordering via its merge step (`GEN-GAME-23`).
- [ ] Combat kernel is a pure sim (state + ordered inputs → state + event log)
      with no scene-tree/render dependency (`GEN-GAME-24/25`).
- [ ] Background-thread sim work marshals results to the main thread before
      touching nodes (`GEN-GAME-26/27`).
- [ ] Save/state format is versioned with a migration path; loaded values are
      validated before reaching gameplay math (`GEN-GAME-28/29`).

---

## References

- Glenn Fiedler, "Fix Your Timestep!" — https://gafferongames.com/post/fix_your_timestep/
- Robert Nystrom, *Game Programming Patterns* — Game Loop, Object Pool, Update
  Method, Double Buffer chapters — https://gameprogrammingpatterns.com/
- Godot Docs, "Optimization: General" and "GPU optimization" —
  https://docs.godotengine.org/en/stable/tutorials/performance/general_optimization.html
- Godot Docs, "Physics interpolation" — https://docs.godotengine.org/en/stable/tutorials/physics/physics_interpolation.html
- Godot Docs, "Using multiple threads" — https://docs.godotengine.org/en/stable/tutorials/performance/using_multiple_threads.html
- Glenn Fiedler, "Deterministic Lockstep" — https://gafferongames.com/post/deterministic_lockstep/
- Glenn Fiedler, "Floating Point Determinism" — https://gafferongames.com/post/floating_point_determinism/
- Gaffer On Games / GDC, networked physics and rollback-netcode determinism
  discussions — https://gafferongames.com/
- IEEE 754 floating-point standard (non-associativity of FP addition).
