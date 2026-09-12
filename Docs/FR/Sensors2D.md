# Détecter avec Area2D et Raycast2D

`Area2D` détecte les corps qui occupent une zone sans les bloquer.
`Raycast2D` cherche le premier obstacle le long d'un segment. Ces deux Nodes
héritent de `Node2D` et nécessitent `Plugins.Physics2D()` en plus de
`Plugins.Nodes(root)`.

## Déclencher une action dans une zone

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
        if let node = contact.node { print("Entrée de $(node.name())") }
    }
}
```

Le constructeur accepte `shape`, `name` et `collision_filter`. La forme est en
mètres ; la position du Node est en unités de scène. L'échelle globale doit
rester `(1, 1)` : redimensionnez la forme pour changer la zone.

`body_entered` et `body_exited` portent un `BodyContact2D`. Son champ `body`
est le handle physique ; `node` identifie le Node physique de `GFX.Nodes`
correspondant lorsqu'il existe. Un corps créé directement dans le monde physique ou par un composant
ECS peut être détecté avec `node == null`. Pour comparer deux handles d'un même
corps, utilisez `body.same_body(other)` ; la comparaison reste possible après
leur destruction, mais leurs propriétés physiques exigent `is_valid()`.

`overlapping_bodies()` retourne l'instantané des corps actuellement connus.
Plusieurs colliders d'un même corps ne produisent qu'une entrée et une sortie.
Les filtres de collision s'appliquent ; les autres capteurs ne déclenchent pas
ces événements de corps. Cette version n'expose pas d'événements entre zones.

Les transitions sont calculées après chaque pas réel de simulation, y compris
chaque pas de rattrapage. Aucun nouvel événement de chevauchement n'est calculé
pendant la pause. Désactiver ou retirer la zone vide ses chevauchements et émet
les sorties à la synchronisation de son cycle de vie. Désactiver ou détruire
un visiteur produit sa sortie au prochain pas physique. Les callbacks peuvent
modifier l'arbre ; les retraits sont réconciliés avant le pas suivant.

## Chercher un obstacle

```sx
var sight = Nodes.Raycast2D(Math.Vec2(300.0, 0.0), "Sight")
player.add_child(sight)
```

`target_position` est l'extrémité locale relative au Node, en unités de scène.
La rotation et l'échelle globales transforment ce segment. Le rayon s'actualise
après chaque pas physique. Sa propriété `hit` est nulle en cas d'absence de
collision ; sinon le `RaycastHit2D` donne :

- `point`, en unités de scène, et `normal`, dans le repère global ;
- `fraction`, la fraction parcourue du segment selon les tolérances du moteur ;
- `collider`, le collider touché, et `body`, son `BodyContact2D`.

`query_filter` contrôle les catégories détectées. `exclude_parent` vaut
`true` et ignore le plus proche ancêtre portant un corps physique ;
`exceptions` est une liste de corps supplémentaires à ignorer.
`include_sensors` vaut `false` ; l'activer permet de toucher aussi les zones.
Le résultat retenu est le plus proche après ces exclusions.

`cast()` effectue immédiatement la requête et actualise `hit`. Appelez-le
pendant un callback de l'arbre, par exemple après avoir changé la direction
ou depuis un Node actif pendant la pause. Il lit le monde déjà lié : dans
`on_ready`, les nouveaux corps peuvent ne pas encore y être présents.
Un segment nul ne touche rien. Une requête ne déplace aucun corps.

La désactivation ou le retrait du rayon efface son résultat à la synchronisation
du cycle de vie. Pendant une pause, le dernier résultat est conservé jusqu'à
une requête manuelle ou une nouvelle simulation. Un résultat conservé peut
donc contenir un handle devenu invalide depuis la requête.

Les preuves exécutables se trouvent dans
[`Sensors2D.sx`](../../Tests/Consumer/Tests/Sensors2D.sx).

Un exemple complet, exécutable sans fenêtre, assemble un score, une zone et
un rayon dans [`EventsAndSensors.sx`](../../Tests/Consumer/Smokes/EventsAndSensors.sx).
