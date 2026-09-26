# Changelog

## [Unreleased]

- Added `Nodes.Canvas3D` for drawing a panel in the 3D world with a frame, raster density, world size, material, and shadows. It automatically activates Scene3D and uses its existing ECS components and surface cache.

- Mounted cameras, sprites, and Canvas2D nodes automatically activate their scene and graphical dependencies, including from `on_ready` and during execution. Asset stores are available when Nodes is prepared. Nonvisual nodes and physics remain usable without a window.
