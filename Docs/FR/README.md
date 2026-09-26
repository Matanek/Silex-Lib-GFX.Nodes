# GFX.Nodes

`GFX.Nodes` propose un arbre de scène orienté objet sans remplacer l’ECS de
GFX. Chaque nœud monté possède une entité ECS ; les scènes 2D et 3D continuent
de posséder leurs composants et leurs renderers.

Voir aussi [Communiquer entre Nodes](Events.md) et
[Détecter avec Area2D et Raycast2D](Sensors2D.md), ainsi que
[Dessiner un panneau en 3D](Canvas3D.md).

## Principe

Pour les personnages et les obstacles, suivre
[Déplacer un personnage avec des collisions](CharacterBody2D.md).

Chaque classe de nœud représente une fonctionnalité :

- `Node` possède un nom, un parent, des enfants et un cycle de vie ;
- `Node2D` ajoute un `GFX.Scene2D.Transform` local ;
- `Node3D` ajoute un `GFX.Scene3D.Transform` local ;
- `Camera2D` et `Camera3D` ajoutent la caméra de leur scène ;
- `Sprite2D` et `Sprite3D` ajoutent un sprite utilisant une image de
  `GFX.Assets` ;
- `Canvas2D` ajoute un dessin vectoriel `GFX.Canvas` rendu par Scene2D ;
- `Canvas3D` place un dessin Canvas sur un panneau rendu par Scene3D ;
- `CharacterBody2D` et `StaticBody2D` ajoutent le déplacement avec collisions et
  les obstacles fixes, avec `Plugins.Physics2D` ;
- `Area2D` détecte les chevauchements sans bloquer les corps ;
- `Raycast2D` recherche le premier obstacle sur un segment.

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
        ..add_plugin(Plugins.Nodes(root))
        ..run()
}
```

`Plugins.Nodes` installe le monde ECS, le temps et les magasins d’assets.
Monter un `Camera2D`, `Sprite2D` ou `Canvas2D` active `Scene2D` ; monter un
`Camera3D`, `Sprite3D` ou `Canvas3D` active `Scene3D`. Chaque scène apporte
ses dépendances Rendering, GPU, Window et Input. Les simples `Node`, `Node2D`
et `Node3D` restent utilisables sans fenêtre ; la simulation `Physics2D` reste
explicite.

Cela fonctionne pour l’arbre initial, les enfants créés dans `on_ready` et
ceux ajoutés pendant l’exécution. Les plugins manquants démarrent à la fin de
l’étape courante, avant le rendu suivant. Leur activation ne relance ni les
callbacks `on_ready` ni les services déjà démarrés. Leurs ressources sont
disponibles après cette étape, pas dans le callback qui vient de les demander.

Pour personnaliser la fenêtre ou le rendu, ajoutez explicitement `Plugins.Window`
ou `Plugins.Rendering` avant `prepare()` ou `run()`, dans l’ordre de votre choix.
Les réglages explicites sont conservés. Les dépendances graphiques restent
actives jusqu’à l’arrêt du contexte, même après le retrait du dernier nœud visuel.

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
automatiquement à `on_process` ; `set_process(false)` suspend ce callback,
et `set_process(true)` le rétablit. Les propriétés modifiées pendant un
même `on_process` sont regroupées avant leur publication dans l’ECS. Une
modification effectuée hors callback est publiée au prochain cycle de mise à
jour. Seuls les transforms descendants réellement affectés sont propagés.

## Activer ou désactiver une branche

`node.enable(false)` rend le nœud et ses descendants inactifs : leurs
`on_process`, leurs animations et leur rendu s’arrêtent, et leurs corps
physiques ne participent plus à la simulation ni aux requêtes spatiales.
`node.enable(true)` réactive la branche. Un enfant explicitement désactivé
reste désactivé, même lorsque son parent est réactivé.

Ce fragment distingue l’objet entier de sa représentation visuelle :

```sx
var enemy = Nodes.Node("Enemy")
var visual = Nodes.Canvas2D("Visual")
enemy.add_child(visual)
visual.enable(false) // Le comportement du parent reste actif.
enemy.enable(false)  // Toute la branche devient inactive.
enemy.enable(true)   // Visual reste désactivé.
```

Les objets restent dans l’arbre et conservent leurs entités, composants et
réglages. La réactivation ne rappelle pas `on_ready`. La visibilité locale,
`process_mode` et le réglage `set_process` restent inchangés. Un mode `always`
ne contourne pas un ancêtre désactivé. La propriété `visible` des nœuds de
rendu permet toujours de masquer leur affichage tout en laissant évoluer leurs
animations et leur comportement.

`is_enabled()` lit le choix local. `is_enabled_in_tree()` indique si le nœud
est monté et actif compte tenu de ses ancêtres. L’héritage change immédiatement
lors d’un appel à `enable` ou d’un changement de parent ; les composants restent
modifiables pendant l’inactivité.

Une sous-classe peut redéfinir `on_enable()` et `on_disable()`. Un nœud monté
actif reçoit son premier `on_enable` après son `on_ready`. Un nœud monté inactif
reçoit `on_ready`, puis attend sa première activation pour recevoir
`on_enable`. Un nœud actif reçoit `on_disable` avant `on_exit_tree` au démontage.
Ces callbacks disposent de `resources()`, `input()` et des raccourcis
applicatifs selon les mêmes règles que les autres callbacks. Omettre `super`
n’empêche pas la désactivation interne.

Les callbacks d’activation et l’état ECS sont synchronisés aux points sûrs du
cycle des Nodes. Plusieurs appels avant une même synchronisation sont regroupés
selon l’état final. Désactiver une branche pendant `on_process` empêche ses
nœuds de recevoir de nouveaux `on_process` dans ce passage. Un appel entre deux
mises à jour est publié à la suivante, même si l’application est en pause.

Les entités inactives sont exclues des `ECS.Query` et de `World.entities()`.
Les systèmes personnalisés utilisant ces requêtes suivent donc l’activation.
L’accès direct à une entité connue reste possible ; un système qui choisit de
parcourir aussi les entités désactivées doit respecter explicitement leur état.

## Suspendre les nœuds du jeu

La propriété `process_mode` utilise `Application.ProcessMode`. Par défaut,
`inherit` reprend le mode effectif du parent ; une racine utilise `pausable`.
Mettre l’application en pause suspend donc les `on_process` du jeu tout
en conservant les entités et leur rendu.

`self.pause_application()` et `self.resume_application()` mettent en pause et
reprennent l’application courante ; `self.is_application_paused()` lit son état.
Dans un Bundle, ils ciblent uniquement ce Bundle. Ces raccourcis protégés
s’utilisent dans `on_ready`, `on_process` et
`on_exit_tree`, avec les mêmes règles de disponibilité que `resources()`.
Ils ne suspendent pas seulement le sous-arbre du nœud appelant.

`set_process_mode(mode)` règle le mode du Node et actualise l’héritage de ses
descendants comme l’affectation à `process_mode`. Cette méthode publique peut
être appelée avant le montage ou pendant l’exécution, et utilisée en cascade.

Dans cet exemple complet, Échap suspend le joueur et peut aussi le réactiver :

```sx
use GFX.Application
use GFX.Input
use GFX.Nodes
use GFX.Plugins

class Player:Nodes.Node {
    override func on_process(delta:float) {
        // Faire avancer le jeu ici.
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

Un enfant `always` continue même sous un parent suspendu ou `disabled`.
`when_paused` convient à un menu actif seulement en pause. Un mode explicite
devient à son tour la valeur héritée par ses descendants ; changer de parent ou
de mode actualise cet héritage. `disabled` suspend les callbacks récurrents,
sans démonter le sous-arbre ni empêcher un enfant d’avoir son propre mode.
`set_process(false)` désactive seulement le callback du nœud concerné,
indépendamment de son mode. `on_ready` et `on_exit_tree` restent exécutés.

L’état de pause est lu une fois par passage des callbacks ; un changement
pendant ce passage s’applique aux nœuds au passage suivant. Les callbacks qui
restent actifs reçoivent le delta réel de la frame. Les ajouts, retraits et
synchronisations de propriétés continuent pendant la pause.

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

## Lire les entrées depuis un nœud

Dans `on_process(delta)`, `self.input()` donne un accès en lecture à
`GFX.Input.State`, actualisé par l’application pendant `pre_update`. La méthode
protégée est aussi disponible dans `on_ready` et `on_exit_tree`. Installer
`Plugins.Input()` avant `Plugins.Nodes`, ou utiliser `Plugins.Window()` qui
installe déjà les entrées. Un arbre sans input reste utilisable tant qu’aucun
nœud n’appelle `input()`.

Ce fragment déplace un nœud 2D tant que la touche droite est maintenue :

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

`is_down` teste une touche maintenue ; `is_pressed` et `is_released` testent
les transitions de la frame. `Input.Key` représente les positions physiques du
clavier : `w`, `a`, `s`, `d` correspondent aux touches ZQSD d’un clavier AZERTY.

Le résultat `@Input.State` est un emprunt en lecture : ne pas le conserver
au-delà du callback. Pour lire plusieurs touches dans une variable `input`,
limiter son emprunt à un bloc, puis modifier le nœud après ce bloc. Aucun appel
à `Input.update()` n’est nécessaire. Après un changement d’arbre, `input()`
lit automatiquement l’état de l’application de destination. Un appel sur un
nœud démonté ou sans état d’input disponible produit une erreur explicite.

## Accéder aux ressources et quitter le jeu

`self.resources()` donne accès aux ressources du contexte applicatif courant
depuis `on_ready`, `on_process` et `on_exit_tree`. Utiliser `get<T>()` pour
emprunter une ressource en lecture et `get_mut<T>()` pour la modifier. Les
ressources propres au jeu sont accessibles de la même manière que celles de
GFX.

Ce programme ferme la fenêtre à l’appui sur Échap :

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

`self.quit_application()` est un raccourci protégé vers `Application.Controller.stop()`.
Il s’utilise depuis les mêmes callbacks que `resources()`. Il demande l’arrêt
de la boucle ; les callbacks de sortie des nœuds sont
ensuite exécutés. Dans un Bundle, le nœud accède aux ressources locales de ce
Bundle, avec le repli habituel vers les ressources parentes. Son contrôleur
arrête uniquement le Bundle. Après un changement d’arbre, les callbacks
utilisent les ressources du contexte de destination.

Récupérer les ressources dans chaque callback qui en a besoin et garder les
emprunts dans ce callback. Un appel à `resources()` hors d’un passage de cycle
de vie, ou sur un nœud démonté, produit une erreur explicite. `get` et
`get_mut` conservent leur erreur habituelle lorsqu’une ressource manque ;
`try_get` et `try_get_mut` permettent de traiter son absence.

Les callbacks s’exécutent sans chevauchement avec les systèmes parallèles du
même contexte, puisque les ressources auxquelles ils accèdent sont choisies
dynamiquement. `SceneTree` et `ECS.World` sont déjà utilisés par le traitement
des nœuds : ne pas les remplacer, les retirer ou vider le magasin pendant un
callback. Pour modifier l’arbre, utiliser les méthodes des nœuds.
