# Dessiner un panneau dans une scène 3D

`Nodes.Canvas3D` place un dessin `GFX.Canvas` sur un rectangle du monde 3D.
Il hérite de `Node3D` : position, rotation et échelle suivent son parent spatial.
Utilisez-le pour une enseigne, un écran ou une étiquette dans la scène.

## Créer un panneau

Ce programme affiche un panneau sans éclairage. La caméra regarde vers le
panneau depuis l’axe Z positif ; fermer la fenêtre arrête l’application.

```sx
use GFX.Application
use GFX.Canvas
use GFX.Color
use GFX.Nodes
use GFX.Plugins
use GFX.Scene3D
use STD.Math

func main() {
    var panel = Nodes.Canvas3D(
        "Sign",
        frame:Canvas.Rect(Math.Vec2(), Math.Vec2(320.0, 180.0)),
        density:2.0,
        world_size:Math.Vec2(3.2, 1.8),
        material:Scene3D.CanvasPanelMaterial.unlit()
    )
    panel.paint(func(painter:&Canvas.Painter) {
        painter.fill(
            Canvas.Rect(Math.Vec2(), Math.Vec2(320.0, 180.0), 16.0),
            Canvas.Fill.solid(Color.slate_800())
        )
        painter.text("Silex", Math.Vec2(32.0, 90.0), Canvas.TextStyle(size:36.0))
    })
    var root = Nodes.Node("Scene")
    root.add_child(panel)
    root.add_child(Nodes.Camera3D()..position = Math.Vec3(0.0, 0.0, 5.0))
    Application()..add_plugin(Plugins.Nodes(root))..run()
}
```

Le montage du panneau active automatiquement Scene3D et ses dépendances
graphiques, y compris lorsque le panneau est ajouté pendant l’exécution.
Le plugin Scene2D n’est pas nécessaire.

## Régler le dessin et sa surface

- `frame` choisit le rectangle du dessin à afficher, par défaut `(0, 0)` à
  `(320, 180)` en coordonnées Canvas. Le dessin conserve son axe Y vers le bas.
- `density` règle le nombre de pixels de surface par unité Canvas, par défaut
  `1`. La texture mesure `ceil(frame.size * density)` ; l’exemple produit
  une surface de 640 × 360 pixels.
- `world_size` fixe les dimensions du panneau dans le monde, par défaut
  `(3.2, 1.8)`. Le rectangle est centré sur l’origine locale, dans le plan XY.
- `text_mode` reprend `Scene2D.Canvas.TextMode`, avec `automatic` par défaut.
- `visible` et `casts_shadows` valent `true` par défaut.

Le cadre, la densité et la taille monde sont modifiables après construction.
Leurs dimensions doivent être finies et strictement positives ; l’origine du
cadre doit être finie. Une valeur invalide produit une erreur explicite dès
la construction ou l’affectation.

`paint(callback)`, `clear()`, `command_count()` et `canvas()` reprennent les
usages de `Canvas2D`. `canvas()` retourne le même dessin partagé : une
modification par cette référence est visible par le panneau. `clear()` efface
le dessin sans retirer le Node.

Le renderer conserve la surface lorsque seuls le transform, la taille monde
ou le matériau changent. Modifier le dessin, le cadre, la densité ou le mode de
texte permet au renderer d’actualiser la surface au prochain rendu visible.
`visible = false` masque le panneau ; `enable(false)` désactive sa branche
selon le contrat habituel des Nodes.

## Choisir le matériau et utiliser l’ECS

`material` utilise directement `Scene3D.CanvasPanelMaterial`. Par défaut, le
matériau est éclairé, à double face, avec transparence `AlphaMode.blend`.
`CanvasPanelMaterial.unlit()` convient aux panneaux qui doivent rester lisibles
sans lumière. `AlphaMode.mask` avec `alpha_cutoff` permet notamment aux ombres
de suivre la couverture du dessin. Le mode blend conserve les limites du tri
des transparences 3D.

Chaque Node construit son propre dessin et son propre matériau par défaut.
Un matériau fourni explicitement peut être partagé entre plusieurs panneaux.
Ses propriétés sont partagées avec l’ECS ; remplacer `panel.material` publie
la nouvelle référence au prochain cycle de synchronisation des Nodes. Les
autres propriétés de placement suivent aussi ce cycle.

Une entité montée contient `Scene3D.Transform`, `Scene3D.CanvasPanel` et
`Scene3D.CanvasPanelMaterial`. Le rendu, les ombres et le cache restent ceux
de Scene3D. Un consommateur ECS peut assembler directement ces trois
composants, sans installer Nodes. Les propriétés du Node pilotent ses
composants ; une écriture ECS directe ne remonte pas dans les propriétés objet.
Le dessin et le matériau partagés restent utilisables après le démontage.

`Canvas3D` affiche une surface plane. Pour transformer un dessin en géométrie,
utiliser `Scene3D.Meshes.add_canvas_geometry`. La sélection au pointeur et le
routage d’événements vers une interface Canvas ne font pas partie de ce Node.

[Retour à GFX.Nodes](README.md)
