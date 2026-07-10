# GDScript / Godot Review Checklist

> Use with [`../languages/gdscript.md`](../languages/gdscript.md),
> [`../13-game-runtime-and-determinism.md`](../13-game-runtime-and-determinism.md),
> and [`general.md`](general.md).

- [ ] Static typing on variables/parameters/returns you control, including
      typed `Array`/`Dictionary` collections; no unnecessary
      `Variant`/untyped members (`GD-TYPE-*`).
- [ ] Frame-rate-independent gameplay logic in `_physics_process`, not
      `_process`; determinism-critical code doesn't consume engine-physics
      results (`GD-LIFE-02`, `GEN-GAME-36`).
- [ ] Child-node references cached in `@onready`, not re-resolved via
      `get_node()`/`$Path` inside per-frame callbacks (`GD-LIFE-03`).
- [ ] Node references held across a frame/`await`/signal are checked with
      `is_instance_valid()` before use (`GD-ERR-03`).
- [ ] Signal hygiene: alive-but-detached (e.g. pooled) receivers disconnect;
      lambda/bound callables tied to an owner's lifetime (`GD-SIG-02`).
- [ ] Presentation animation via `Tween`/`AnimationPlayer`, not `_process`
      lerps (`GD-SIG-05`).
- [ ] `queue_free()` used to remove nodes (never `free()` inside a
      signal/physics callback) (`GD-RES-01`).
- [ ] High-frequency spawn/destroy (VFX, damage numbers, projectiles) pooled,
      not instanced-and-freed per event (`GD-RES-03`).
- [ ] Data-driven content modeled as `Resource` subclasses treated as
      immutable templates, not mutated at runtime (`GD-EXP-02/03`).
- [ ] Scenes self-contained; "call down, signal up"; no `get_parent()`/
      absolute-path reach-ups; folders grouped by feature (`GD-ORG-*`).
- [ ] Autoloads reserved for genuine global singletons, not a dumping ground
      for convenience state (`GD-AUTO-*`).
- [ ] Input via InputMap actions and the correct `_input`/`_unhandled_input`
      callback; no hardcoded keys (`GD-INPUT-*`).
- [ ] Hot-path identifiers use `StringName`; heavyweight loads run via
      `ResourceLoader.load_threaded_request()` (`GD-PERF-01/02`).
- [ ] No `Node`/scene-tree API touched from a background thread or C#
      thread-pool continuation without `call_deferred`/`call_thread_safe`
      (`GD-CONC-01/03`).
- [ ] Untrusted/mod/imported content loaded only as restricted plain data,
      never as an executable script/arbitrary `Resource` type (`GD-SEC-01`).
- [ ] External/tampered values entering gameplay math validated and clamped
      (`GD-SEC-02`).
- [ ] Sim-affecting code paths free of frame-rate-dependent timing and
      free of `await`-introduced ordering hazards (`GD-LIFE-02`, `GD-SIG-04`).
- [ ] Deterministic simulation has headless, seeded, repeatable test coverage
      running in CI (`GD-TEST-02`, `GD-CI-01`).
- [ ] Release builds ship error telemetry (custom `Logger`, backtraces, crash
      reporting) (`GD-OBS-*`).
- [ ] `.godot/` ignored; `.uid` files committed; text `.tscn`/`.tres`; LFS for
      binaries; mass file upgrades in dedicated commits (`GD-VCS-*`).
- [ ] Controls remappable, UI text scalable, color not the sole information
      channel; accessible names on custom controls (`GD-ACC-01`).
