# GDScript / Godot Review Checklist

> Use with [`../languages/gdscript.md`](../languages/gdscript.md),
> [`../13-game-runtime-and-determinism.md`](../13-game-runtime-and-determinism.md),
> and [`general.md`](general.md).

- [ ] Static typing on variables/parameters/returns you control; no unnecessary
      `Variant`/untyped members (`GD-TYPE-*`).
- [ ] Deterministic/frame-rate-independent gameplay logic in
      `_physics_process`, not `_process` (`GD-LIFE-02`).
- [ ] Child-node references cached in `@onready`, not re-resolved via
      `get_node()`/`$Path` inside per-frame callbacks (`GD-LIFE-03`).
- [ ] Node references held across a frame/`await`/signal are checked with
      `is_instance_valid()` before use (`GD-ERR-03`).
- [ ] Manually-connected signals are disconnected in `_exit_tree` (or use
      auto-managed editor connections) (`GD-SIG-02`).
- [ ] `queue_free()` used to remove nodes (never `free()` inside a
      signal/physics callback) (`GD-RES-01`).
- [ ] High-frequency spawn/destroy (VFX, damage numbers, projectiles) pooled,
      not instanced-and-freed per event (`GD-RES-03`).
- [ ] Data-driven content (classes/skills/items/affixes/enemies) modeled as
      `Resource` subclasses treated as immutable templates, not mutated at
      runtime (`GD-EXP-02/03`).
- [ ] Autoloads reserved for genuine global singletons, not a dumping ground
      for convenience state (`GD-AUTO-*`).
- [ ] No `Node`/scene-tree API touched from a background thread without
      `call_deferred`/`call_thread_safe` (`GD-CONC-01`).
- [ ] Untrusted/mod/imported content loaded only as restricted plain data,
      never as an executable script/arbitrary `Resource` type (`GD-SEC-01`).
- [ ] External/tampered values entering gameplay math validated and clamped
      (`GD-SEC-02`).
- [ ] Combat/sim-affecting code paths free of frame-rate-dependent timing and
      free of `await`-introduced ordering hazards (`GD-LIFE-02`, `GD-SIG-04`).
- [ ] Deterministic combat sim has headless, seeded, repeatable test coverage
      (`GD-TEST-02`).
