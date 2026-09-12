# Detect with Area2D and Raycast2D

`Area2D` detects bodies occupying a zone without blocking them. `Raycast2D`
finds the first obstacle along a segment. Both inherit from `Node2D` and
require `Plugins.Physics2D()` alongside `Plugins.Nodes(root)`.

## Trigger an action in a zone

```sx
use GFX.Nodes
use GFX.Physics
use STD.Math

class Checkpoint:Nodes.Area2D {
    init():super(Physics.Shape2D.box(Physics.Box2D(Math.Vec2(2.0, 1.0)))) {}

    override func on_ready() {
        self.listen(self.body_entered, self.activate)
    }

    func activate(contact:Nodes.BodyContact2D) {
        if let node = contact.node { print("Entered: $(node.name())") }
    }
}
```

The constructor accepts `shape`, `name` and `collision_filter`. The shape uses
meters; the Node position uses scene units. Global scale must remain `(1, 1)`:
resize the shape to change the zone.

`body_entered` and `body_exited` carry a `BodyContact2D`. Its `body` field is
the physics handle; `node` identifies the corresponding `GFX.Nodes` physics Node when one exists.
A body created directly in the physics world or through an ECS component can
be detected with `node == null`. To compare two handles for the same body, use
`body.same_body(other)`; comparison remains valid after destruction, but
physical properties require `is_valid()`.

`overlapping_bodies()` returns a snapshot of currently known bodies. Multiple
colliders on one body produce only one entry and one exit. Collision filters
apply; other sensors do not trigger these body events. This version does not
expose events between areas.

Transitions are calculated after every actual simulation step, including each
catch-up step. No new overlap events are calculated while paused. Disabling or
removing the area clears its overlaps and emits exits at lifecycle
synchronization. Disabling or destroying a visitor produces its exit on the
next physics step. Handlers may change the tree; removals are reconciled before
the next step.

## Find an obstacle

```sx
var sight = Nodes.Raycast2D(Math.Vec2(300.0, 0.0), "Sight")
player.add_child(sight)
```

`target_position` is the local endpoint relative to the Node, in scene units.
Global rotation and scale transform the segment. The ray updates after every
physics step. Its `hit` property is null when nothing is hit; otherwise the
`RaycastHit2D` provides:

- `point`, in scene units, and `normal`, in global coordinates;
- `fraction`, the traveled fraction of the segment within engine tolerances;
- `collider`, the hit collider, and `body`, its `BodyContact2D`.

`query_filter` controls detected categories. `exclude_parent` defaults to
`true` and ignores the nearest ancestor carrying a physics body; `exceptions`
is a list of additional bodies to ignore. `include_sensors` defaults to
`false`; enable it to hit areas too. The selected result is the nearest hit
after applying these exclusions.

`cast()` performs the query immediately and updates `hit`. Call it during a
tree callback, for example after changing direction or from a Node active while
paused. It reads the already bound world: new bodies may not yet be present
in `on_ready`. A zero segment hits nothing. A query moves no bodies.

Disabling or removing the ray clears its result at lifecycle synchronization.
While paused, the last result persists until a manual query or resumed
simulation. A retained result may therefore contain a handle invalidated since
the query.

Executable coverage is in
[`Sensors2D.sx`](../../Tests/Consumer/Tests/Sensors2D.sx).

A complete example runs a score, an area and a ray without a window in
[`EventsAndSensors.sx`](../../Tests/Consumer/Smokes/EventsAndSensors.sx).
