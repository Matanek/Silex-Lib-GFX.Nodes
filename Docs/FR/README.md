# GFX.Nodes

`GFX.Nodes` propose un arbre de scène orienté objet sans remplacer l’ECS de
GFX. Chaque nœud monté possède une entité ECS ; les scènes 2D et 3D continuent
de posséder leurs composants et leurs renderers.

## Principe

Chaque classe de nœud représente une fonctionnalité :

- `Node` possède un nom, un parent, des enfants et un cycle de vie ;
- `Node2D` ajoute un `GFX.Scene2D.Transform` local ;
- `Node3D` ajoute un `GFX.Scene3D.Transform` local ;
- `Camera2D` et `Camera3D` ajoutent la caméra de leur scène ;
- `Sprite2D` et `Sprite3D` ajoutent un sprite utilisant une image de
  `GFX.Assets` ;
- `Canvas2D` ajoute un dessin vectoriel `GFX.Canvas` rendu par Scene2D.

Le `Node` générique ne connaît aucune dimension et ne transporte aucun
transform. Un nœud spatial compose son transform avec celui de son parent
immédiat lorsque celui-ci appartient au même domaine. Un `Node` générique entre
deux nœuds spatiaux démarre donc naturellement une nouvelle racine spatiale.

## Utilisation

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

`Plugins.Nodes` installe l’intégration Application, le monde ECS et le temps.
Le renderer reste un choix explicite : ajouter `Plugins.Scene2D()` pour les
nœuds 2D, `Plugins.Scene3D()` pour les nœuds 3D, ou les deux.

## Caméras et sprites

Les spécialisations de scène gardent une API objet directe tout en montant les
composants ECS correspondants. Il n’existe pas de méthode générique
`add_component` : chaque Node porte seulement sa fonctionnalité.

```silex
var camera = Nodes.Camera2D(zoom:2.0)
var sprite = Nodes.Sprite2D(image, name:"Player", layer:1)

root.add_child(camera)
root.add_child(sprite)

sprite.visible = false
camera.zoom = 3.0
```

`Camera2D : Node2D`, `Sprite2D : Node2D`, `Camera3D : Node3D` et
`Sprite3D : Node3D`. Lorsque leurs propriétés changent, elles sont
synchronisées vers
`GFX.Scene2D.Camera`, `GFX.Scene2D.Sprite`, `GFX.Scene3D.Camera` et
`GFX.Scene3D.Sprite`. Scene2D et Scene3D demeurent donc seuls propriétaires du
rendu.

## Cycle de vie

Une classe spécialisée redéfinit `on_ready`, `on_process` ou `on_exit_tree`.
Les vrais appels de cycle de vie restent privés, de sorte qu’un oubli de
`super` ne puisse pas casser la bookkeeping interne ni la chaîne d’appel.

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

Les Nodes fournis par le package sont statiques par défaut et ne reçoivent
donc aucun callback par frame. Une classe définie par l’application participe
automatiquement à `on_process` ; `set_process(false)` permet de la désactiver,
et `set_process(true)` de la réactiver. Les propriétés modifiées pendant un
même `on_process` sont regroupées avant leur publication dans l’ECS. Une
modification effectuée hors callback est publiée au prochain cycle de mise à
jour. Seuls les transforms descendants réellement affectés sont propagés.

La première base privilégie les arbres construits avant le démarrage. L’ajout
et le retrait d’enfants pendant que l’arbre est monté ne sont pas encore pris
en charge.
