# Historique

## [Unreleased]

- Les caméras, sprites et Canvas2D montés dans un arbre activent automatiquement leur scène et ses dépendances graphiques, y compris depuis `on_ready` ou pendant l’exécution. Les magasins d’assets sont disponibles dès la préparation de Nodes. Les nœuds sans affichage et la physique restent utilisables sans fenêtre.
