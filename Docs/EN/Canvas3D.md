# Draw a panel in a 3D scene

`Nodes.Canvas3D` places a `GFX.Canvas` drawing on a rectangle in the 3D world.
It inherits from `Node3D`: position, rotation, and scale follow its spatial
parent. Use it for a sign, screen, or label in the scene.

## Create a panel

This program displays an unlit panel. The camera looks toward the panel from
the positive Z axis; closing the window stops the application.

```sx
use GFX.Application
use GFX.Canvas
use GFX.Color
use GFX.Nodes
use GFX.Plugins
use GFX.Scene3D
use STD.Math

func main() {
    var panel = Nodes.Canvas3D(
        "Sign",
        frame:Canvas.Rect(Math.Vec2(), Math.Vec2(320.0, 180.0)),
        density:2.0,
        world_size:Math.Vec2(3.2, 1.8),
        material:Scene3D.CanvasPanelMaterial.unlit()
    )
    panel.paint(func(painter:&Canvas.Painter) {
        painter.fill(
            Canvas.Rect(Math.Vec2(), Math.Vec2(320.0, 180.0), 16.0),
            Canvas.Fill.solid(Color.slate_800())
        )
        painter.text("Silex", Math.Vec2(32.0, 90.0), Canvas.TextStyle(size:36.0))
    })
    var root = Nodes.Node("Scene")
    root.add_child(panel)
    root.add_child(Nodes.Camera3D()..position = Math.Vec3(0.0, 0.0, 5.0))
    Application()..add_plugin(Plugins.Nodes(root))..run()
}
```

Mounting the panel automatically activates Scene3D and its graphical
dependencies, including when the panel is added during execution.
The Scene2D plugin is not required.

## Configure the drawing and its surface

- `frame` selects the drawing rectangle to display, defaulting to `(0, 0)`
  through `(320, 180)` in Canvas coordinates. The drawing keeps its downward Y axis.
- `density` sets surface pixels per Canvas unit, defaulting to `1`. The texture
  measures `ceil(frame.size * density)`; the example produces a 640 × 360 surface.
- `world_size` sets the panel dimensions in the world, defaulting to
  `(3.2, 1.8)`. The rectangle is centered on the local origin in the XY plane.
- `text_mode` uses `Scene2D.Canvas.TextMode`, defaulting to `automatic`.
- `visible` and `casts_shadows` both default to `true`.

The frame, density, and world size can change after construction. Their
dimensions must be finite and strictly positive; the frame origin must be
finite. Invalid values produce an explicit error at construction or assignment.

`paint(callback)`, `clear()`, `command_count()`, and `canvas()` follow the
`Canvas2D` usage. `canvas()` returns the same shared drawing: changes through
that reference are visible to the panel. `clear()` erases the drawing without
removing the Node.

The renderer retains the surface when only the transform, world size, or
material changes. Changing the drawing, frame, density, or text mode lets the
renderer refresh the surface on its next visible render. `visible = false`
hides the panel; `enable(false)` deactivates its branch according to the usual
Nodes contract.

## Choose a material and use ECS

`material` directly uses `Scene3D.CanvasPanelMaterial`. The default material
is lit, double-sided, and uses `AlphaMode.blend` transparency.
`CanvasPanelMaterial.unlit()` suits panels that should remain readable without
lights. `AlphaMode.mask` with `alpha_cutoff` lets shadows follow the drawing's
coverage. Blend mode retains the limitations of 3D transparency sorting.

Each Node creates its own drawing and material by default. An explicitly
provided material can be shared by several panels. Its properties are shared
with ECS; replacing `panel.material` publishes the new reference on the next
Nodes synchronization cycle. Other placement properties follow that cycle too.

A mounted entity contains `Scene3D.Transform`, `Scene3D.CanvasPanel`, and
`Scene3D.CanvasPanelMaterial`. Rendering, shadows, and caching remain owned by
Scene3D. An ECS consumer can assemble these three components directly without
installing Nodes. Node properties drive its components; direct ECS writes do
not update object properties. The shared drawing and material remain usable
after unmounting.

`Canvas3D` displays a flat surface. To turn a drawing into geometry, use
`Scene3D.Meshes.add_canvas_geometry`. Pointer picking and routing events to a
Canvas interface are outside this Node's scope.

[Back to GFX.Nodes](README.md)
