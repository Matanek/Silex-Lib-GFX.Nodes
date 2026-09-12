# Déplacer un personnage avec des collisions

`Nodes.CharacterBody2D` déplace une capsule contre les corps du monde physique.
Le contrôleur choisit la vitesse ; `move_and_slide` applique le déplacement
autorisé et retire de la vitesse la composante bloquée par les contacts.
`Nodes.StaticBody2D` représente les murs et les obstacles fixes.

Installer `Plugins.Physics2D()` et `Plugins.Nodes(root)` dans la même
Application ou le même Bundle. Leur ordre d'installation est libre. Le plugin
physique assure les pas fixes ; les nœuds ne créent pas une deuxième horloge.
Un nœud physique sans `Plugins.Physics2D` produit un diagnostic au montage.

## Écrire le contrôleur

Ce fragment définit un personnage qui avance horizontalement. Ajouter une
instance à la racine passée à `Plugins.Nodes` et lui donner un enfant
`Canvas2D` ou `Sprite2D` pour son apparence.

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
        self.move_and_slide()
    }
}
```

`velocity` est une vitesse globale en unités de scène par seconde. Ne pas la
multiplier par `delta` avant l'appel sans argument : celui-ci applique déjà le
pas fixe. Les entrées et l'IA peuvent décider de cette vitesse de la même façon.
La gravité, l'accélération et le freinage appartiennent au contrôleur.

Un contrôleur qui intègre lui-même son accélération peut appeler
`move_and_slide(displacement)` avec une translation globale en unités de scène.
Cette surcharge conserve son intégration, sans remultiplier par le pas fixe.
Les deux formes renvoient `true` si le mouvement rencontre un collider.
Elles s'appellent uniquement depuis `on_physics_process`.

## Donner une forme aux obstacles

Ce fragment crée un mur de 24 unités sur 360 lorsque le plugin physique utilise
`scene_units_per_meter = 100.0` :

```sx
var wall = Nodes.StaticBody2D(
    Physics.Shape2D.box(Physics.Box2D(Math.Vec2(0.24, 3.6))), "Wall"
)..position = Math.Vec2(240.0, 0.0)
```

Les positions des nœuds restent en unités de scène. Les formes et les tolérances
de `GFX.Physics` sont en mètres. À cette échelle, la capsule du contrôleur mesure
24 unités de large et 36 de haut. Un dessin ne crée jamais automatiquement un
collider : dessiner un enfant de même taille pour aligner l'apparence.

Le filtre du personnage vient de `CharacterMover2DSettings.query_filter` ; il
s'applique aussi à son propre collider. `StaticBody2D` accepte un paramètre
`collision_filter`. Le personnage ignore son propre corps dans les requêtes,
mais les autres personnages et corps peuvent le détecter. Les capteurs sont
ignorés comme obstacles par défaut.

## Lire les contacts et gérer le cycle de vie

`motion()` renvoie le dernier `Physics.CharacterMoveResult2D`, ou `null` avant
le premier déplacement lié au monde. Ses translations sont en mètres ; il
contient les colliders et les plans rencontrés ainsi que l'indication d'un
recouvrement initial. Les handles de colliders suivent la durée de vie du monde.

`CharacterBody2D` active automatiquement les callbacks physiques. Les autres
nœuds peuvent employer `set_physics_process(true)` et surcharger
`on_physics_process(delta)`. `set_process` contrôle séparément les callbacks de
frame. Un pas physique ne s'exécute jamais pendant la pause de la simulation,
même pour un nœud de mode `always`. Les modes des nœuds continuent de filtrer
les callbacks lorsque la simulation avance.

`enable(false)` conserve le corps et sa pose, mais suspend les callbacks et les
collisions de la branche. Le retrait de l'arbre détruit le corps ; un nouveau
montage crée une nouvelle identité physique. Un nœud ajouté pendant un callback
physique reçoit son corps à la prochaine frame. Son déplacement reste nul tant
que ce corps n'est pas lié. Les enfants visuels suivent la pose résolue dès le
pas courant, et les Bundles gardent leurs propres mondes et callbacks.

## Limites de cette première version

Le personnage utilise une capsule droite, sans rotation globale ni mise à
l'échelle globale. Un parent spatial est possible si ces contraintes restent
respectées. `StaticBody2D` accepte la rotation, mais exige aussi une échelle
globale unitaire ; modifier la forme pour changer sa taille.

Le personnage est piloté par son contrôleur : il n'est pas déplacé par les
impulsions du solveur et n'applique pas de poussée de gameplay aux obstacles
dynamiques. Le suivi des plateformes, la classification sol/mur/plafond,
l'adhérence aux pentes et le franchissement des marches ne sont pas fournis.
Une modification directe de `position` reste un repositionnement sans test du
trajet ; employer `move_and_slide` pour le mouvement contraint.

Les [tests consommateurs](../../Tests/Consumer/Tests/CharacterBody2D.sx) couvrent
le glissement, la vitesse corrigée, les filtres, les transformations parentales,
la pause, les mutations pendant un pas et l'isolation des Bundles.

[Retour à GFX.Nodes](README.md)
