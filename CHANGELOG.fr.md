# Historique

## [Unreleased]

- Ajout de `Nodes.Canvas3D` pour dessiner un panneau dans le monde 3D avec cadre, densité, taille monde, matériau et ombres. Il active automatiquement Scene3D et utilise ses composants ECS et son cache de surfaces existants.

- Les caméras, sprites et Canvas2D montés dans un arbre activent automatiquement leur scène et ses dépendances graphiques, y compris depuis `on_ready` ou pendant l’exécution. Les magasins d’assets sont disponibles dès la préparation de Nodes. Les nœuds sans affichage et la physique restent utilisables sans fenêtre.
