# Create a parallax background

`Nodes.Parallax2D` passes parallax settings to sprites and tile maps in its
spatial branch. `GFX.Scene2D` continues to own rendering.

This fragment assumes an existing `background:Nodes.TileMap2D`:

```sx
var distance = Nodes.Parallax2D(
    factor:Math.Vec2(0.2, 0.0),
    origin:Math.Vec2(160.0, 90.0),
    repeat_size:Math.Vec2(96.0, 0.0))
distance.add_child(background)
```

Scenery moves horizontally at 20% of world speed and keeps a stable vertical
horizon. Its motif repeats every 96 world units along X. `factor`, `origin`,
and `repeat_size` remain editable after mounting. A nested `Parallax2D`
overrides inherited settings for its own branch.

Inheritance follows immediate `Node2D` parents, like transforms: a generic
`Node` interrupts the chain. Remounting under another parent recomputes the
settings. `Parallax2D` alone starts no display; graphical children activate
their usual renderer.

Positions and collisions retain their world coordinates. In ECS, add
`Scene2D.Parallax` directly to each graphical entity. See the
[Scene2D contract](../../../GFX.Scene2D/Docs/EN/README.md#scroll-scenery-with-parallax)
for factors, repetition in world coordinates, and rendering rules.
