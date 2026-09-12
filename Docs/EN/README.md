# GFX.Nodes

`GFX.Nodes` provides an object-oriented scene tree without replacing the GFX
ECS. Every mounted node owns an ECS entity; the 2D and 3D scene packages remain
the owners of their components and renderers.

## Principle

For characters and obstacles, see
[Move a character with collisions](CharacterBody2D.md).

Each node class represents one capability:

- `Node` owns a name, parent, children, and lifecycle;
- `Node2D` adds a local `GFX.Scene2D.Transform`;
- `Node3D` adds a local `GFX.Scene3D.Transform`;
- `Camera2D` and `Camera3D` add the camera for their scene;
- `Sprite2D` and `Sprite3D` add a sprite using a `GFX.Assets` image;
- `Canvas2D` adds a `GFX.Canvas` vector drawing rendered by Scene2D;
- `CharacterBody2D` and `StaticBody2D` add collision-aware motion and fixed
  obstacles, using `Plugins.Physics2D`.

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
`Sprite3D : Node3D`. When their properties change, they synchronize to
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

Nodes supplied by the package are static by default and therefore receive no
per-frame callback. A class authored by the application participates in
`on_process` automatically; `set_process(false)` disables it and
`set_process(true)` enables it again. Properties changed by one `on_process`
callback are coalesced before publication to ECS. A change made outside a
callback is published on the next update cycle. Only affected descendant
transforms are propagated.

## Enable or disable a branch

`node.enable(false)` makes the node and its descendants inactive: their
`on_process`, animations and rendering stop, and their physical bodies no
longer participate in simulation or spatial queries. `node.enable(true)`
reactivates the branch. An explicitly disabled child stays disabled when its
parent is enabled again.

This fragment separates the whole object from its visual representation:

```sx
var enemy = Nodes.Node("Enemy")
var visual = Nodes.Canvas2D("Visual")
enemy.add_child(visual)
visual.enable(false) // The parent behavior remains active.
enemy.enable(false)  // The whole branch becomes inactive.
enemy.enable(true)   // Visual remains disabled.
```

Objects stay in the tree and retain their entities, components and settings.
Reactivation does not call `on_ready` again. Local visibility, `process_mode`
and the `set_process` setting remain unchanged. An `always` mode cannot bypass
a disabled ancestor. Render nodes still expose `visible` to hide their output
while their animations and behavior keep advancing.

`is_enabled()` reads the local choice. `is_enabled_in_tree()` reports whether
the node is mounted and active given its ancestors. Inheritance changes
immediately on `enable` or reparenting; component properties remain editable
while inactive.

A subclass can override `on_enable()` and `on_disable()`. An active node
receives its first `on_enable` after `on_ready`. A node mounted inactive receives
`on_ready`, then waits for its first activation before receiving `on_enable`.
An active node receives `on_disable` before `on_exit_tree` during unmounting.
These callbacks can use `resources()`, `input()` and application shortcuts under
the same rules as other callbacks. Omitting `super` does not prevent internal
deactivation.

Activation callbacks and ECS state synchronize at safe points in the Nodes
lifecycle. Multiple calls before a synchronization are coalesced to the final
state. Disabling a branch during `on_process` prevents further `on_process`
calls in that branch during the same pass. A call between updates is published
at the next update, including while the application is paused.

Inactive entities are excluded from `ECS.Query` and `World.entities()`. Custom
systems using those queries therefore respect activation. Direct access to a
known entity remains available; a system that deliberately includes disabled
entities must explicitly respect their state.

## Pause game nodes

The `process_mode` property uses `Application.ProcessMode`. By default,
`inherit` takes the parent's effective mode; a root uses `pausable`.
Pausing the application therefore suspends game `on_process` callbacks
while preserving entities and their rendering.

`self.pause_application()` and `self.resume_application()` pause and resume the
current application; `self.is_application_paused()` reads its state. In a
Bundle, they target only that Bundle.
These protected shortcuts are available in `on_ready`, `on_process` and
`on_exit_tree`, under the same availability rules as `resources()`. They do
not pause only the calling node's subtree.

`set_process_mode(mode)` sets the Node's mode and updates inheritance for its
descendants just like assigning `process_mode`. This public method is available
before mounting and at runtime, and can be used in a cascade.

In this complete example, Escape pauses the player and can resume it:

```sx
use GFX.Application
use GFX.Input
use GFX.Nodes
use GFX.Plugins

class Player:Nodes.Node {
    override func on_process(delta:float) {
        // Advance gameplay here.
    }
}

class Root:Nodes.Node {
    override func on_ready() {
        self.set_process_mode(Application.ProcessMode.always)
        self.add_child(Player()..set_process_mode(Application.ProcessMode.pausable))
    }

    override func on_process(delta:float) {
        let toggle = self.input().is_pressed(Input.Key.escape)
        if toggle {
            if self.is_application_paused() { self.resume_application() }
            else { self.pause_application() }
        }
    }
}

func main() {
    Application()
        ..add_plugin(Plugins.Window())
        ..add_plugin(Plugins.Nodes(Root()))
        ..run()
}
```

An `always` child continues even under a paused or `disabled` parent.
`when_paused` suits a menu active only during a pause. An explicit mode becomes
the value inherited by its descendants; changing the parent or mode updates
this inheritance. `disabled` suspends recurring callbacks without unmounting
the subtree or preventing a child from choosing its own mode.
`set_process(false)` disables only that node's callback, independently of its
mode. `on_ready` and `on_exit_tree` still execute.

The pause state is read once per callback pass; a change during that pass
applies to nodes on the next pass. Callbacks that remain active receive the
real frame delta. Additions, removals and property synchronization continue
while paused.

## Change the tree at runtime

`add_child` and `remove_child` work before startup, from `on_ready`,
`on_process` and `on_exit_tree`, and between updates. Node properties remain
editable at runtime.

This fragment creates children when their parent becomes ready:

```sx
class Player:Nodes.Node2D {
    override func on_ready() {
        self.add_child(Nodes.Canvas2D("Body"))
    }
}

class Root:Nodes.Node {
    override func on_ready() {
        self.add_child(Player())
    }
}
```

The links observed through `parent`, `children`, `child_count` and `path`
change immediately. ECS mounting and callbacks wait for a safe point: after
startup callbacks, then before and after the `on_process` pass. A child added
from `on_ready` is therefore ready by the end of startup; a node added during
`on_process` receives its first process callback on the next update. Existing
children mount before their parent's `on_ready`; additions requested by a
callback mount after it returns.

`remove_child` returns the detached node, or `null` if it was not a child.
The subtree immediately stops receiving new `on_process` calls. At the next
safe point, children receive `on_exit_tree` before their parent and their
entities are destroyed. `is_inside_tree` and `entity` describe ECS mounting:
they remain set until unmounting, including inside `on_exit_tree`, then become
`false` and `null`. The detached object and its properties remain usable while
the application retains a reference.

To change parents, call `remove_child`, then `add_child` on the new parent.
The subtree receives new entities and another `on_ready`; `ready_count` counts
these mounts. Local transforms are preserved and composed with the new spatial
parent. When moving between trees, mounting in the new tree waits for the old
tree to unmount the subtree in its own ECS world; both trees must therefore
continue updating.

An addition cancelled before mounting receives no callbacks. Shutdown completes
pending removals and unmounts all remaining mounted nodes. Additions targeting
the stopping tree change the object hierarchy but create no new entities.

Cycles, duplicate sibling names and adding a node that already has a parent
remain forbidden. A mounted tree root cannot become another node's child.

## Read input from a node

Inside `on_process(delta)`, `self.input()` provides read access to
`GFX.Input.State`, updated by the application during `pre_update`. This
protected method is also available in `on_ready` and `on_exit_tree`. Install
`Plugins.Input()` before `Plugins.Nodes`, or use `Plugins.Window()`, which
already installs input. Trees without input remain usable as long as no node
calls `input()`.

This fragment moves a 2D node while the rightward key is held:

```sx
use GFX.Input
use GFX.Nodes
use STD.Math

class Player:Nodes.Node2D {
    override func on_process(delta:float) {
        let right = self.input().is_down(Input.Key.d)
        if right {
            self.position = self.position.add(Math.Vec2(260.0 * delta, 0.0))
        }
    }
}
```

`is_down` tests a held key; `is_pressed` and `is_released` test transitions
within the current frame. `Input.Key` represents physical keyboard positions:
`w`, `a`, `s`, `d` correspond to the ZQSD keys on an AZERTY keyboard.

The `@Input.State` result is a read borrow: do not retain it beyond the
callback. To read several keys through an `input` variable, limit its borrow
to a block, then mutate the node after that block. No call to `Input.update()`
is needed. After moving to another tree, `input()` automatically reads the
destination application's state. Calling it on an unmounted node or without
an available input state produces an explicit error.

## Access resources and quit the game

`self.resources()` provides the current application context's resources from
`on_ready`, `on_process`, and `on_exit_tree`. Use `get<T>()` to borrow a
resource for reading and `get_mut<T>()` to modify it. Game resources are
available in the same way as GFX resources.

This program closes the window when Escape is pressed:

```sx
use GFX.Application
use GFX.Input
use GFX.Nodes
use GFX.Plugins

class Root:Nodes.Node {
    override func on_process(delta:float) {
        let quit = self.input().is_pressed(Input.Key.escape)
        if quit {
            self.quit_application()
        }
    }
}

func main() {
    Application()
        ..add_plugin(Plugins.Window())
        ..add_plugin(Plugins.Nodes(Root()))
        ..run()
}
```

`self.quit_application()` is a protected shortcut to `Application.Controller.stop()`.
It is available in the same callbacks as `resources()`. It requests loop
termination; node exit callbacks then run. Inside a
Bundle, the node accesses that Bundle's local resources, with the usual
fallback to parent resources. Its controller stops only the Bundle. After a
node moves to another tree, callbacks use the destination context's resources.

Retrieve resources in each callback that needs them and keep borrows within
that callback. Calling `resources()` outside a lifecycle pass or on an
unmounted node produces an explicit error. `get` and `get_mut` retain their
usual error when a resource is missing; `try_get` and `try_get_mut` allow
handling its absence.

Callbacks do not overlap parallel systems in the same context because their
resource accesses are chosen dynamically. `SceneTree` and `ECS.World` are
already in use by node processing: do not replace them, remove them, or clear
the store during a callback. Use node methods to modify the tree.
