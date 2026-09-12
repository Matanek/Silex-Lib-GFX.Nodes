# Move a character with collisions

`Nodes.CharacterBody2D` moves a capsule against bodies in the physics world.
The controller chooses the velocity; `move` applies the permitted displacement.
By default it slides and removes the velocity component blocked by contacts.
`Nodes.StaticBody2D` represents walls and fixed obstacles.

Install `Plugins.Physics2D()` and `Plugins.Nodes(root)` in the same Application
or Bundle, in either order. The physics plugin owns fixed stepping; nodes do
not create another clock. Mounting a physics node without `Plugins.Physics2D`
produces a diagnostic.

## Write the controller

This fragment defines a character moving horizontally. Add an instance to the
root passed to `Plugins.Nodes`, and attach a `Canvas2D` or `Sprite2D` child for
its appearance.

```sx
use GFX.Nodes
use GFX.Physics
use STD.Math

class Player:Nodes.CharacterBody2D {
    init():super(Physics.CharacterMover2DSettings()
        ..capsule = Physics.Capsule2D(
            Math.Vec2(0.0, -0.06), Math.Vec2(0.0, 0.06), 0.12
        ), "Player"
    ) {}

    override func on_physics_process(delta:float) {
        self.velocity = Math.Vec2(360.0, 0.0)
        self.move()
    }
}
```

`velocity` is a global velocity in scene units per second. Do not multiply it
by `delta` before the no-argument call: that call already applies the fixed
delta. Input and AI can choose this velocity in the same way. Gravity,
acceleration and braking belong to the controller.

A controller integrating its own acceleration can call
`move(displacement)` with a global translation in scene units. This
overload preserves that integration without multiplying by the fixed delta
again. Both forms return a `Nodes.MotionResult2D`. Call them only from
`on_physics_process`.

The `response` parameter selects the collision response:

- `Nodes.CollisionResponse.slide` (the default) continues along obstacles and
  removes blocked components from `velocity`.
- `Nodes.CollisionResponse.stop` stops at the first impact. It preserves
  `velocity`: the controller chooses whether to stop, apply damage or bounce.

```sx
let result = self.move(displacement, Nodes.CollisionResponse.stop)
if result.collided() {
    let normal = result.collisions()[0].normal
    // The remaining path is available for a custom response.
    let remaining = result.remaining_translation
}
```

`move()` always slides. To select `stop` from velocity, call
`move(self.velocity.multiply(delta), Nodes.CollisionResponse.stop)`.
Both responses recover initial overlap before applying the requested motion.
Impacts respect the physics solver's tolerances.

`test_move(displacement, response)` predicts the same result from the current
pose, also defaulting to sliding. It changes neither the node pose, body,
velocity nor `motion()`, and does not advance the simulation. It works during lifecycle
callbacks of a mounted node, including `on_process`, outside physics steps and
while paused. Obstacles use their
currently synchronized world poses; the test does not predict their future motion.

## Give obstacles a shape

This fragment creates a wall measuring 24 by 360 scene units when the physics
plugin uses `scene_units_per_meter = 100.0`:

```sx
var wall = Nodes.StaticBody2D(
    Physics.Shape2D.box(Physics.Box2D(Math.Vec2(0.24, 3.6))), "Wall"
)..position = Math.Vec2(240.0, 0.0)
```

Node positions remain in scene units. `GFX.Physics` shapes and tolerances use
meters. At this scale the controller's capsule is 24 scene units wide and 36
high. Drawing never creates a collider automatically: draw a child with matching
dimensions to align the appearance.

The character filter comes from `CharacterMover2DSettings.query_filter` and also
applies to its own collider. `StaticBody2D` accepts a `collision_filter`
parameter. The character excludes its own body from queries, while other
characters and bodies can detect it. Sensors are ignored as obstacles by default.

## Read contacts and manage lifecycle

`move` returns a `Nodes.MotionResult2D`; `motion()` stores the last applied
result, or `null` before the first call and after remounting.

- `applied_translation` is the completed movement in scene units.
- `remaining_translation` is the requested minus completed movement, including
  any initial overlap recovery.
- `started_overlapping` reports initial overlap.
- `collided()` reports contacts; `collisions()` provides `Nodes.MotionCollision2D`
  values with a global `point` in scene units, a unit `normal`, and `collider()`.

Contacts can originate at the start, along the path or at the endpoint. A contact
therefore does not necessarily mean blocked movement. Collider handles follow
the world's lifetime rules. While a body awaits world binding, the result reports
zero completed movement, the entire requested movement as remaining, and no contacts.

`CharacterBody2D` enables physics callbacks automatically. Other nodes can use
`set_physics_process(true)` and override `on_physics_process(delta)`.
`set_process` controls frame callbacks separately. Physics steps never run while
the simulation is paused, even for an `always` node. Node modes still filter
callbacks while the simulation advances.

`enable(false)` keeps the body and its pose but suspends branch callbacks and
collisions. Detaching destroys the body; remounting creates a new physics
identity. A node added during a physics callback receives its body on the next
frame. Its movement remains zero until that body is bound. Visual children
follow the resolved pose within the current step, and Bundles retain separate
worlds and callbacks.

## Limits of this first version

The character uses an upright capsule with zero global rotation and unit global
scale. Spatial parents are supported while these constraints hold.
`StaticBody2D` supports rotation but also requires unit global scale; resize the
shape to change its dimensions.

The character is controller-driven: solver impulses do not move it, and it does
not apply gameplay pushing to dynamic obstacles. Platform following,
floor/wall/ceiling classification, slope snapping and step climbing are not
provided. Assigning `position` directly remains a repositioning operation
without testing the path; use `move` for constrained motion.

The [consumer tests](../../Tests/Consumer/Tests/CharacterBody2D.sx) cover sliding,
corrected velocity, filters, parent transforms, pause, mutations during a step
and Bundle isolation.

[Back to GFX.Nodes](README.md)
