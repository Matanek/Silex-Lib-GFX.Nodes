# GFX.Nodes

`GFX.Nodes` provides an object-oriented scene tree without replacing the GFX
ECS. Every mounted node owns an ECS entity; the 2D and 3D scene packages remain
the owners of their components and renderers.

## Principle

Each node class represents one capability:

- `Node` owns a name, parent, children, and lifecycle;
- `Node2D` adds a local `GFX.Scene2D.Transform`;
- `Node3D` adds a local `GFX.Scene3D.Transform`;
- `Camera2D` and `Camera3D` add the camera for their scene;
- `Sprite2D` and `Sprite3D` add a sprite using a `GFX.Assets` image;
- `Canvas2D` adds a `GFX.Canvas` vector drawing rendered by Scene2D.

The generic `Node` has no dimension and carries no transform. A spatial node
composes its transform with its immediate parent when that parent belongs to
the same domain. A generic `Node` between two spatial nodes therefore starts a
new spatial root.

## Usage

```silex
use GFX.Application
use GFX.Canvas
use GFX.Color
use GFX.Nodes
use GFX.Plugins
use STD.Math

class Body:Nodes.Canvas2D {
    init():super("Body") {
        self.paint(func(painter:&Canvas.Painter) {
            painter.fill(
                Canvas.Circle(Math.Vec2(), 12.0),
                Canvas.Fill.solid(Color.cyan_400())
            )
        })
    }

    override func on_ready() {
        print("ready $(self.path())")
    }
}

func main() {
    var player = Nodes.Node2D("Player", Math.Vec2(40.0, 20.0))
    player.add_child(Body())

    var root = Nodes.Node("Level")
    root.add_child(player)

    Application()
        ..add_plugin(Plugins.Window())
        ..add_plugin(Plugins.Rendering())
        ..add_plugin(Plugins.Scene2D())
        ..add_plugin(Plugins.Nodes(root))
        ..run()
}
```

`Plugins.Nodes` installs Application integration, the ECS world, and time.
Rendering remains explicit: add `Plugins.Scene2D()` for 2D nodes,
`Plugins.Scene3D()` for 3D nodes, or both.

## Cameras and sprites

Scene specializations keep a direct object API while mounting their
corresponding ECS components. There is no generic `add_component` method: each
Node carries only its own capability.

```silex
var camera = Nodes.Camera2D(zoom:2.0)
var sprite = Nodes.Sprite2D(image, name:"Player", layer:1)

root.add_child(camera)
root.add_child(sprite)

sprite.visible = false
camera.zoom = 3.0
```

`Camera2D : Node2D`, `Sprite2D : Node2D`, `Camera3D : Node3D`, and
`Sprite3D : Node3D`. On each cycle, their properties synchronize to
`GFX.Scene2D.Camera`, `GFX.Scene2D.Sprite`, `GFX.Scene3D.Camera`, and
`GFX.Scene3D.Sprite`. Scene2D and Scene3D therefore remain the sole rendering
owners.

## Lifecycle

A specialized class overrides `on_ready`, `on_process`, or `on_exit_tree`.
The actual lifecycle dispatch remains private so forgetting `super` cannot
break internal bookkeeping or the call chain.

```silex
class Player:Nodes.Node2D {
    override func on_process(delta:float) {
        self.position = Math.Vec2(
            self.position.x + 80.0 * delta,
            self.position.y
        )
    }
}
```

This first base focuses on trees built before startup. Adding or removing
children while a tree is mounted is not supported yet.
