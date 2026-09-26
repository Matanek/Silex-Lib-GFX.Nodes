# Changelog

## [Unreleased]

- Mounted cameras, sprites, and Canvas2D nodes automatically activate their scene and graphical dependencies, including from `on_ready` and during execution. Asset stores are available when Nodes is prepared. Nonvisual nodes and physics remain usable without a window.
