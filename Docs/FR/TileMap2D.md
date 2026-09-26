# Construire une couche de tuiles

`Nodes.TileMap2D` place une grille de tuiles dans une scène 2D. Il hérite de
`Node2D` et utilise directement le composant ECS `Scene2D.TileMap` et son
renderer. Une carte montée possède une seule entité, quel que soit son nombre
de cellules.

## Créer et modifier la grille

Le constructeur `Nodes.TileMap2D(atlas, columns, rows, name)` reçoit un
`Scene2D.TileAtlas`, un nombre de colonnes et un nombre de lignes strictement
positifs. Le nom optionnel vaut `"TileMap2D"`. La grille est initialement vide.
L’atlas et les dimensions sont fixés à la construction.

Le `TileAtlas` décrit une image de `GFX.Assets.Image.Store`, découpée en tuiles
rectangulaires régulières, sans marge ni espacement. Les dimensions de l’image
doivent être des multiples exacts de celles d’une tuile. Les indices de tuiles
avancent de gauche à droite, puis du haut vers le bas dans l’image.

- `set(column, row, tile)` place une tuile dans une cellule.
- `tile(column, row)` retourne son indice, ou `null` si elle est vide.
- `clear(column, row)` vide une cellule ; `clear()` vide toute la carte.
- `fill(tile)` remplit la carte avec une même tuile.
- `contains(column, row)` indique si une coordonnée appartient à la grille.
- `atlas()`, `columns()`, `rows()` et `size()` décrivent la carte. `size()`
  donne ses dimensions locales avant application du transform.

Une coordonnée hors grille provoque une erreur pour la lecture ou l’écriture
d’une cellule. Un indice de tuile négatif est refusé dès l’écriture ; un indice
qui dépasse l’atlas est détecté par le renderer lorsqu’il prépare cette tuile.
Les opérations sans effet ne déclenchent aucune synchronisation du Node.

## Placer et afficher une couche

Les propriétés héritées `position`, `rotation` et `scale` composent le
placement avec celui du parent `Node2D` immédiat. Les réglages `color`, `layer`,
`depth` et `visible` sont modifiables et peuvent être fournis au constructeur
après `name`. Ils valent respectivement blanc, `0`, `0.0` et `true` par défaut.
`sampling()` lit le mode d’échantillonnage ; `sampling(value)` choisit
`Scene2D.Sampling.pixelated` ou `smooth`. Le mode initial est `pixelated`.

Chaque Node représente une couche. Plusieurs Nodes peuvent partager le même
atlas tout en conservant des cellules et des réglages indépendants. Le montage
d’une carte active automatiquement Scene2D et ses dépendances graphiques,
y compris lors d’un ajout pendant l’exécution. Les Nodes fournis sont statiques
par défaut ; une sous-classe suit les règles de traitement habituelles.

Les éditions sont immédiatement lisibles sur le Node et sont publiées dans
l’ECS au prochain point de synchronisation. Plusieurs éditions dans un même
callback sont regroupées. Les écritures ECS directes ne remontent pas dans le
Node : utiliser ses méthodes pour modifier une carte qu’il possède.

`visible = false` masque la couche. `enable(false)` désactive sa branche sans
perdre les cellules. Retirer puis remonter le Node recrée son entité avec les
cellules et les réglages conservés. Les cellules vides ne produisent aucune
instance ; le renderer existant limite la préparation à la zone visible.

## Retrouver une cellule

Les lignes avancent vers le haut dans le monde Scene2D. `cell_origin(column,
row)` retourne l’origine locale d’une cellule ; `cell_center(column, row)`
retourne son centre. `coordinate_at(point)` convertit un point **local à la
carte** en `Scene2D.TileCoordinate?`, ou retourne `null` hors de la grille.
Une position de pointeur doit d’abord être convertie depuis le viewport vers
le monde, puis vers l’espace local de la carte.

## Surfaces de neuf tuiles et collisions

`fill(column, row, columns, rows, patch)` accepte `Scene2D.TilePatch`, avec les
mêmes coordonnées et le même contrat que le composant ECS. Les neuf indices
peuvent être explicites ou provenir de `Scene2D.TilePatch.atlas(top_left, columns)`.
Le rectangle doit mesurer au moins 2 × 2 cellules ; les bords et le centre se
répètent, les coins restent fixes.

Activer `collision_enabled = true` et installer `Plugins.Physics2D` rend toutes
les cellules occupées solides. La valeur initiale est `false`. `collision_filter` configure les catégories,
le masque et le groupe de collision de cette couche. Le Node ajoute
`Physics.TileMapCollider2D` et laisse le plugin physique posséder le corps et
fusionner les rectangles. Une édition du Node met donc à jour les collisions
sans code de reconstruction dans le jeu. La visibilité ne change pas la
physique ; la désactivation de la branche suspend les collisions. L'échelle
globale doit rester unitaire. Les formes utilisent les dimensions des cellules
et le `scene_units_per_meter` du plugin.

Le renderer regroupe les cellules visibles en un dessin instancié par couche,
conserve leur préparation et évite les transferts GPU inchangés. Le coût des
frames stables dépend de la partie visible, pas de la surface totale du niveau.

## Utiliser directement l’ECS

Le Node monte `Scene2D.Transform` et `Scene2D.TileMap`. Un consommateur ECS
peut employer les mêmes opérations sur `Scene2D.TileMap`, également disponible
sous l’alias `Components.TileMap2D`, sans installer Nodes.

Cette version couvre les grilles carrées et rectangulaires de dimensions fixes.
Les terrains automatiques, le dual grid, les dispositions hexagonales ou
isométriques restent à ajouter au socle ECS.

[Retour à GFX.Nodes](README.md)
