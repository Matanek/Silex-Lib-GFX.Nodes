# Place a fixed 3D obstacle

`Nodes.StaticBody3D` adds a fixed `PhysicsBody3D` to a `Node3D`. Install
`Plugins.Physics3D()` and `Plugins.Nodes(root)` in the same Application. The
node does not start the renderer, so the scene can run without a window.

```silex
use GFX.Application
use GFX.Nodes
use GFX.Physics
use GFX.Plugins
use STD.Math

var parent = Nodes.Node3D("Parent", Math.Vec3(5.0, 0.0, 0.0))
parent.add_child(Nodes.StaticBody3D(
    Physics.Shape3D.box(Physics.Box3D(Math.Vec3(1.0)))
)..position = Math.Vec3(2.0, 0.0, 0.0))
var root = Nodes.Node()..add_child(parent)
var application = Application()
    ..add_plugin(Plugins.Physics3D())
    ..add_plugin(Plugins.Nodes(root))
```

The local pose follows `Node3D` composition. The physical body receives the
global pose when mounted and after a parent move, at the next fixed step.
Global scale must remain one: size the `Shape3D`, not the node. Mounting the
node without `Plugins.Physics3D`, or using an invalid physical scale, produces
a diagnostic. Its entity and body are removed together when unmounted.

The [consumer test](../../Tests/Consumer/Tests/StaticBody3D.sx) verifies
composition under a moved and rotated parent.
