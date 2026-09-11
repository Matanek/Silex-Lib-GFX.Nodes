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

## Modifier l’arbre pendant l’exécution

`add_child` et `remove_child` fonctionnent avant le démarrage, depuis
`on_ready`, `on_process` et `on_exit_tree`, ainsi qu’entre les mises à jour.
Les propriétés des nœuds restent modifiables à chaud.

Ce fragment construit les enfants quand leur parent devient prêt :

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

Les liens observés par `parent`, `children`, `child_count` et `path` changent
dès l’appel. Le montage ECS et les callbacks attendent un point sûr : après
les callbacks de démarrage, puis avant et après le passage des `on_process`.
Un enfant ajouté dans `on_ready` est donc prêt à la fin du démarrage ; un
nœud ajouté pendant `on_process` reçoit son premier traitement à la mise à
jour suivante. Les enfants déjà présents sont montés avant le `on_ready` de
leur parent ; les ajouts demandés par un callback sont montés après son retour.

`remove_child` renvoie le nœud détaché, ou `null` s’il n’était pas un enfant.
Le sous-arbre cesse immédiatement de recevoir de nouveaux `on_process`. Au
point sûr suivant, les enfants puis leur parent reçoivent `on_exit_tree` et
leurs entités sont détruites. `is_inside_tree` et `entity` décrivent ce montage
ECS : ils restent renseignés jusqu’au démontage, y compris dans `on_exit_tree`,
puis deviennent respectivement `false` et `null`. L’objet détaché et ses
propriétés restent utilisables tant que l’application en conserve une référence.

Pour changer de parent, appeler `remove_child`, puis `add_child` sur le nouveau
parent. Le sous-arbre reçoit de nouvelles entités et un nouveau `on_ready` ;
`ready_count` compte ces montages. Ses transforms locaux sont conservés et
recomposés avec le nouveau parent spatial. Entre deux arbres, le montage dans
le nouvel arbre attend que l’ancien ait démonté le sous-arbre dans son propre
monde ECS ; chaque arbre doit donc continuer à être mis à jour.

Un ajout annulé avant son montage ne reçoit aucun callback. À l’arrêt, les
retraits en attente sont terminés et tous les nœuds encore montés sont démontés.
Les ajouts effectués vers l’arbre qui s’arrête modifient la hiérarchie objet,
mais ne créent pas de nouvelles entités.

Les cycles, les noms identiques parmi les enfants d’un même parent et l’ajout
d’un nœud ayant déjà un parent restent interdits. La racine d’un arbre monté
ne peut pas devenir l’enfant d’un autre nœud.
