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

Built-in nodes are static by default. Consumer subclasses participate in
`on_process` automatically and can opt out with `set_process(false)`. Property
changes are dirty-tracked: external changes synchronize on the next update,
while changes made by one `on_process` callback are coalesced into one ECS
write per affected component.

## Documentation

- [French documentation](Docs/FR/README.md)
- [English documentation](Docs/EN/README.md)

The package requires Silex 0.44.0 or newer.
