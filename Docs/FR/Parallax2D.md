# Créer un fond en parallaxe

`Nodes.Parallax2D` transmet des réglages de parallaxe aux sprites et tilemaps
de sa branche spatiale. Le rendu reste assuré par `GFX.Scene2D`.

Ce fragment suppose un `background:Nodes.TileMap2D` existant :

```sx
var distance = Nodes.Parallax2D(
    factor:Math.Vec2(0.2, 0.0),
    origin:Math.Vec2(160.0, 90.0),
    repeat_size:Math.Vec2(96.0, 0.0))
distance.add_child(background)
```

Le décor se déplace horizontalement à 20 % de la vitesse du monde et garde un
horizon vertical stable. Le motif se répète tous les 96 unités monde en X.
`factor`, `origin` et `repeat_size` sont modifiables après montage. Un
`Parallax2D` imbriqué remplace les réglages hérités pour sa propre branche.

L’héritage suit les parents `Node2D` immédiats, comme les transforms : un
`Node` générique interrompt cette chaîne. Le remontage sous un autre parent
recalcule les réglages. `Parallax2D` seul ne démarre aucun affichage ; ses
enfants graphiques activent leur renderer habituel.

Les positions et collisions gardent leurs coordonnées monde. En ECS, ajouter
`Scene2D.Parallax` directement à chaque entité graphique. Consulter le
[contrat Scene2D](../../../GFX.Scene2D/Docs/FR/README.md#faire-défiler-un-décor-en-parallaxe)
pour les facteurs, les périodes en coordonnées monde et les règles de rendu.
