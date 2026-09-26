# Build a tile layer

`Nodes.TileMap2D` places a tile grid in a 2D scene. It inherits from `Node2D`
and directly uses the `Scene2D.TileMap` ECS component and renderer. A mounted
map owns one entity regardless of its cell count.

## Create and edit the grid

The `Nodes.TileMap2D(atlas, columns, rows, name)` constructor takes a
`Scene2D.TileAtlas` and strictly positive column and row counts. The optional
name defaults to `"TileMap2D"`. The grid starts empty. The atlas and dimensions
are fixed at construction.

A `TileAtlas` describes an image from `GFX.Assets.Image.Store`, split into
regular rectangular tiles with no margins or spacing. The image dimensions
must be exact multiples of the tile dimensions. Tile indices advance from
left to right, then from top to bottom in the image.

- `set(column, row, tile)` places a tile in a cell.
- `tile(column, row)` returns its index, or `null` for an empty cell.
- `clear(column, row)` empties a cell; `clear()` empties the whole map.
- `fill(tile)` fills the map with one tile.
- `contains(column, row)` checks whether a coordinate belongs to the grid.
- `atlas()`, `columns()`, `rows()`, and `size()` describe the map. `size()`
  gives its local dimensions before applying the transform.

Reading or writing a cell outside the grid produces an error. A negative tile
index is rejected on write; an index beyond the atlas is detected by the
renderer when preparing that tile. Operations that change nothing do not
trigger Node synchronization.

## Place and display a layer

The inherited `position`, `rotation`, and `scale` properties compose placement
with the immediate `Node2D` parent. The mutable `color`, `layer`, `depth`, and
`visible` settings can also be supplied to the constructor after `name`. They
default to white, `0`, `0.0`, and `true` respectively. `sampling()` reads the
sampling mode; `sampling(value)` selects `Scene2D.Sampling.pixelated` or
`smooth`. The initial mode is `pixelated`.

Each Node represents one layer. Several Nodes can share an atlas while keeping
independent cells and settings. Mounting a map automatically activates Scene2D
and its graphical dependencies, including when added during execution. Built-in
Nodes are static by default; subclasses follow the usual processing rules.

Edits can be read immediately on the Node and are published to ECS at the next
synchronization point. Multiple edits in one callback are coalesced. Direct
ECS writes do not update the Node: use its methods to edit a map it owns.

`visible = false` hides the layer. `enable(false)` deactivates its branch
without losing cells. Removing and remounting the Node recreates its entity
with the retained cells and settings. Empty cells produce no instances; the
existing renderer limits preparation to the visible region.

## Locate a cell

Rows advance upward in the Scene2D world. `cell_origin(column, row)` returns
a cell's local origin; `cell_center(column, row)` returns its center.
`coordinate_at(point)` converts a point **local to the map** into
`Scene2D.TileCoordinate?`, or returns `null` outside the grid. A pointer position
must first be converted from viewport to world coordinates, then into the map's
local space.
## Nine-tile surfaces and collisions

`fill(column, row, columns, rows, patch)` accepts `Scene2D.TilePatch`, with the
same coordinates and contract as the ECS component. Supply nine indices or use
`Scene2D.TilePatch.atlas(top_left, columns)`. The rectangle must contain at least
2 × 2 cells; edges and center repeat while corners remain fixed.

Set `collision_enabled = true` and install `Plugins.Physics2D` to make every
occupied cell solid. The initial value is `false`. `collision_filter` configures this layer's
categories, mask and collision group. The Node adds
`Physics.TileMapCollider2D`; the physics plugin owns the body and merges solid
rectangles. Editing the Node updates collisions without reconstruction code in
the game. Visibility does not affect physics; disabling the branch suspends
collisions. Global scale must remain one. Shapes use cell dimensions and the
plugin's `scene_units_per_meter`.

The renderer batches visible cells into one instanced draw per layer, retains
their preparation and skips unchanged GPU transfers. Stable frame cost depends
on the visible portion, rather than total level area.


## Use ECS directly

The Node mounts `Scene2D.Transform` and `Scene2D.TileMap`. An ECS consumer can
use the same operations on `Scene2D.TileMap`, also available through the
`Components.TileMap2D` alias, without installing Nodes.

This version covers square and rectangular grids with fixed dimensions.
Automatic terrains, dual grids, hexagonal and isometric layouts remain future
additions to the ECS foundation.

[Back to GFX.Nodes](README.md)
