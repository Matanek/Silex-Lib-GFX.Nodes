# Communicate between Nodes

A method requests an action from a known object. An event announces a fact
without knowing its recipients: a character entered a zone, a door opened, or
a score changed. Use typed events for these notifications and ordinary methods
for commands.

## Publish an event

The producer keeps a private `EventEmitter<T>` and exposes only its
`Event<T>`. Consumers can listen to that event; they cannot emit it.

```sx
use GFX.Nodes

class Score:Nodes.Node {
    private var updates:Nodes.EventEmitter<int> = Nodes.EventEmitter<int>()
    var score_changed:Nodes.Event<int> { get { return self.updates.event } }
    private var value:int = 0

    func add(points:int) {
        self.value += points
        self.updates.emit(self.value)
    }
}

class ScoreDisplay:Nodes.Node {
    private var score:Score
    init(score:Score):super("ScoreDisplay") { self.score = score }

    override func on_ready() {
        self.listen(self.score.score_changed, self.show_score)
    }

    func show_score(value:int) { print("Score: $(value)") }
}
```

`listen` infers the message type. Its callback can be a free function or a
method bound to an owned class: the latter retains its receiver object. A
closure borrowing local variables or a method bound to a structure remains
lexical and is unsuitable for a lasting subscription.

## Lifetime

`node.listen(event, callback, delivery)` belongs to the Node calling `listen`,
even if the callback targets another object. The Node must be mounted.
`on_ready` is the usual subscription point.

The return value is a `Connection`: `is_connected()` reads its state and
`disconnect()` disconnects immediately, including during emission. Repeating
`disconnect()` has no effect. Dropping the variable holding the connection
does not disconnect the subscription.

Removing a Node from its tree disconnects its subscriptions as soon as removal
is requested, before its exit callbacks. Tree shutdown follows the same rule.
Remounting calls `on_ready` again, which can create fresh connections. Pause,
`set_process(false)` and `enable(false)` preserve subscriptions: notifications
received in these states are still delivered.

## Order and delivery

The default `EventDelivery.immediate` calls recipients during `emit`, in
subscription order. Each emission snapshots its connections: a subscription
added during a callback waits for the next emission; a connection disconnected
before its turn is skipped. Recursive emission is immediate too.

To defer a handler:

```sx
self.listen(area.body_entered, self.on_body_entered, Nodes.EventDelivery.deferred)
```

Deferred messages join the recipient's tree. That tree delivers one batch at
the end of startup, each Node update pass, and each completed physics step.
Messages queued during that batch wait for the next boundary; disconnecting
also cancels pending messages. Batch order follows queue insertion order.

Messages follow ordinary Silex value semantics: their class objects remain
shared. Delivery does not deep-copy object state. An event explicitly shared
between two trees uses a queue in each recipient tree. No global bus is installed.

An immediate callback runs in its emitter's context. The recipient Node's
`resources()` access requires an active pass of its own tree. Use deferred
delivery for an emitter outside that tree when the handler needs its resources.
A scene's physics events are emitted inside its tree's physics pass.

Executable coverage is in
[`Events.sx`](../../Tests/Consumer/Tests/Events.sx).

A complete example runs a score, an area and a ray without a window in
[`EventsAndSensors.sx`](../../Tests/Consumer/Smokes/EventsAndSensors.sx).
