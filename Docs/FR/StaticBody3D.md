# Placer un obstacle fixe en 3D

`Nodes.StaticBody3D` ajoute un `PhysicsBody3D` fixe à un `Node3D`. Installer
`Plugins.Physics3D()` et `Plugins.Nodes(root)` dans la même Application. Le
nœud ne démarre pas le renderer : une scène sans fenêtre reste possible.

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

La pose locale suit la composition `Node3D`; le corps physique reçoit la pose
globale au montage et après tout déplacement du parent, au prochain pas fixe.
L'échelle globale doit rester unitaire : dimensionner la `Shape3D` plutôt que
le nœud. Monter ce nœud sans `Plugins.Physics3D` ou avec une échelle physique
invalide produit un diagnostic. Son entité et son corps sont retirés ensemble
au démontage.

Le [test consommateur](../../Tests/Consumer/Tests/StaticBody3D.sx) vérifie la
composition sous un parent déplacé et tourné.
