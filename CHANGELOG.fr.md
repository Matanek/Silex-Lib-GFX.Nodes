# Historique

## [Unreleased]

- TileMap2D accepte les motifs TilePatch et collision_enabled ; le rendu et les collisions restent portés par les composants ECS Scene2D et Physics.

- Ajout de `Nodes.TileMap2D` : grille rectangulaire, édition des cellules, conversions locales et réglages de rendu. La couche active automatiquement Scene2D et synchronise les éditions regroupées vers son composant ECS existant.

- Ajout de `Nodes.Canvas3D` pour dessiner un panneau dans le monde 3D avec cadre, densité, taille monde, matériau et ombres. Il active automatiquement Scene3D et utilise ses composants ECS et son cache de surfaces existants.

- Les caméras, sprites et Canvas2D montés dans un arbre activent automatiquement leur scène et ses dépendances graphiques, y compris depuis `on_ready` ou pendant l’exécution. Les magasins d’assets sont disponibles dès la préparation de Nodes. Les nœuds sans affichage et la physique restent utilisables sans fenêtre.
