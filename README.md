# GFX.Nodes

`GFX.Nodes` provides an object-oriented scene tree for Silex applications while
keeping GFX.Application, GFX.ECS, GFX.Scene2D, and GFX.Scene3D as the runtime
owners of scheduling, entities, and scene components.

```text
silex install GFX.Nodes
```

The generic `Node` owns tree structure and lifecycle only. `Node2D` and
`Node3D` add their respective scene transforms. `Camera2D`, `Camera3D`,
`Sprite2D`, and `Sprite3D` project object properties into the corresponding
scene components, while `Canvas2D` owns vector drawing commands backed by
`GFX.Canvas`.

Children can be added, removed and reparented at runtime, including from
lifecycle callbacks. Object links change immediately; ECS mounting and teardown
are applied at safe points around node processing. Subclasses read the current
application input through `self.input()` in lifecycle callbacks. They access
application and game resources through `self.resources()`, including
`Application.Controller` to request shutdown with `stop()`.

`enable(false)` deactivates an entire branch, including processing, rendering,
animation and physics participation, while retaining its entities and settings.
`enable(true)` restores the branch while preserving each child’s own choice.
Subclasses can respond with `on_enable` and `on_disable`.

Built-in nodes are static by default. Consumer subclasses participate in
`on_process` automatically and can opt out with `set_process(false)`. Property
changes are dirty-tracked: external changes synchronize on the next update,
while changes made by one `on_process` callback are coalesced into one ECS
write per affected component.

## Documentation

`CharacterBody2D` provides controller-driven capsule motion with
`move(displacement, response)` (slide or stop) and non-mutating `test_move`;
`StaticBody2D` provides fixed obstacles. Install `Plugins.Physics2D` to run their
simulation and use `on_physics_process` for fixed-step movement. Physics is a
package dependency; starting its simulation remains explicit.

- [French documentation](Docs/FR/README.md)
- [English documentation](Docs/EN/README.md)

The package requires Silex 0.44.0 or newer.
