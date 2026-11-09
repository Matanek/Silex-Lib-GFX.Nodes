# Changelog

## [0.2.0] - 2026-11-09

- Added `StaticBody3D`: a fixed obstacle below a `Node3D` parent, with its
  global pose synchronized by `Plugins.Physics3D` and unit scale checked.

- Added `Parallax2D`: `Node2D` descendants inherit the Scene2D parallax component. Nested groups can override settings; edits and remounts update ECS components.

- TileMap2D now accepts TilePatch fills and collision_enabled; Scene2D and Physics ECS components continue to own rendering and collisions.

- Added `Nodes.TileMap2D`: rectangular grids, cell editing, local coordinate conversion, and rendering settings. The layer automatically activates Scene2D and synchronizes coalesced edits to its existing ECS component.

- Added `Nodes.Canvas3D` for drawing a panel in the 3D world with a frame, raster density, world size, material, and shadows. It automatically activates Scene3D and uses its existing ECS components and surface cache.

- Mounted cameras, sprites, and Canvas2D nodes automatically activate their scene and graphical dependencies, including from `on_ready` and during execution. Asset stores are available when Nodes is prepared. Nonvisual nodes and physics remain usable without a window.
