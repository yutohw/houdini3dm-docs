# Houdini3DM

Reads and writes Rhino `.3dm` files in Houdini. Two SOP assets and one Rhino-side
Python script.

[Buy on Gumroad](https://yutohw.gumroad.com/l/houdini3dm).

| | |
| --- | --- |
| Version | 1.1.0 |
| Assets | `rhino_3dm_import`, `rhino_3dm_export` |
| Houdini | 22.0, 21.0, 20.5 — Indie or Apprentice, Windows |
| Rhino | 7 or later, for `HouRebuild.py` only |
| Bundled | rhino3dm 8.35.0 (MIT) |

Houdini and Rhino do not share a geometry kernel. Rhino is NURBS, Houdini is
polygonal. Import samples Rhino geometry into polygons or NURBS curves; export
writes polygons back out as planar surfaces, fitted curves and points, and a
Rhino-side script does the trimming and joining that only the Rhino kernel can
do. The round trip is therefore two steps in each direction, not one.

---

## Contents

- [Requirements](#requirements)
- [Installation](#installation)
- [The Rhino alias](#the-rhino-alias)
- [Workflow](#workflow)
- [Rhino 3DM Import](#rhino-3dm-import)
  - [Outputs](#outputs)
  - [Parameters](#import-parameters)
  - [Attributes created](#attributes-created-by-the-importer)
  - [Groups created](#groups-created-by-the-importer)
- [Rhino 3DM Export](#rhino-3dm-export)
  - [Inputs](#inputs)
  - [How pieces are formed](#how-pieces-are-formed)
  - [Parameters](#export-parameters)
  - [Attributes read](#attributes-read-by-the-exporter)
  - [User strings written](#user-strings-written-to-the-3dm)
  - [The report](#the-report)
- [HouRebuild.py](#hourebuildpy)
- [Attribute reference](#attribute-reference)
- [Troubleshooting](#troubleshooting)
- [Licensing](#licensing)

---

## Requirements

Houdini **Indie** or **Apprentice**, version 22.0, 21.0 or 20.5, on Windows. The
assets are `.hdalc` files. A commercial Houdini licence cannot load them. That
restriction is set by SideFX and is outside the control of this package.

Rhino 7 or later is needed only to run `HouRebuild.py`. Import and export work
without Rhino installed.

No admin rights, no `pip install`, no network access. rhino3dm is bundled for
Python 3.9 through 3.13; Houdini loads the copy matching its own interpreter.

---

## Installation

### 1. Unzip

Unzip to `C:\tools\Houdini3DM-1.1.0`. The package file already points there, so
nothing needs editing. Houdini reads this folder on every launch — do not leave
it in Downloads.

The folder should contain `Houdini3DM.json`, the documentation files, and the
`Houdini3DM`, `rhino` and `samples` folders directly.

### 2. Copy the package file

Copy `Houdini3DM.json` into the Houdini packages folder:

| Houdini | Folder |
| --- | --- |
| 22.0 | `C:\Users\<you>\Documents\houdini22.0\packages\` |
| 21.0 | `C:\Users\<you>\Documents\houdini21.0\packages\` |
| 20.5 | `C:\Users\<you>\Documents\houdini20.5\packages\` |

Create `packages` if it does not exist.

### 3. Restart Houdini

Create a Geometry object, dive inside, press Tab and type `rhino`. Both assets
appear. Point **Rhino 3DM Import** at `samples/Rhino_Example_Base_File_01.3dm`.

### Installing elsewhere

Put the folder anywhere and edit one line in `Houdini3DM.json` before copying it
into `packages`. Set `HOUDINI3DM` to the full path of the **inner**
`Houdini3DM` folder — the one containing `otls` and the `pythonX.Ylibs`
folders — with forward slashes:

```json
{ "HOUDINI3DM": "D:/wherever/Houdini3DM-1.1.0/Houdini3DM" }
```

### Verifying

The **Houdini3DM** shelf has a **Check Install** tool. It reports the Houdini
version, Python version, licence category, whether rhino3dm loaded and from
where, and whether each asset is installed. Include its output in any support
request.

### Upgrading

The install folder is named after the version, so a new release lands beside the
old one. After installing:

1. Replace `Houdini3DM.json` in the packages folder — the old one points at the
   old install folder.
2. Re-point the Rhino alias at the new `HouRebuild.py`.
3. Delete the old install folder.

Saved `.hip` files are unaffected. The asset definition version has not changed
since 1.0.0, so existing nodes pick up new behaviour with no parameters to reset.

---

## The Rhino alias

`HouRebuild.py` runs often. Picking it out of a file dialog every time is slow.
Make it a one-word command instead.

In Rhino: **Tools › Options › Aliases**. Click **+** and fill in two fields:

| Field | Value |
| --- | --- |
| Alias | `hou` |
| Command Macro | `! _-RunPythonScript "C:\tools\Houdini3DM-1.1.0\rhino\HouRebuild.py"` |

Typing `hou` now runs the rebuild on whatever is open.

The macro reads as:

| Token | Meaning |
| --- | --- |
| `!` | Cancel any command still running |
| `_` | Force the English command name, so the alias works on a localised Rhino |
| `-` | Run the command without its dialog |
| `"…"` | Path to the script. Keep the quotes |

Any alias name works as long as it is not already a Rhino command. The alias is
stored per user and survives restarts, so it is a one-off on each machine.

**As a toolbar button:** right-click an empty part of any toolbar, choose **New
Button**, paste the same macro into *Left mouse button command*.

---

## Workflow

### Rhino to Houdini

One step. Point the importer at a `.3dm` and work with the result.

### Houdini to Rhino

Two steps, and the second is not optional.

1. Export from Houdini with **Rhino 3DM Export**.
2. Open the result in Rhino. Solids arrive as loose face outlines, not joined
   polysurfaces. This is the expected halfway state.
3. Run `HouRebuild.py`. It builds trimmed surfaces from the outlines, joins each
   piece into a polysurface, recreates block definitions and places instances.
4. Read the report printed to the Rhino command line.

The exporter writes outlines rather than surfaces because trimming and joining
require the NURBS kernel, which lives in Rhino.

### Round trip

Import, modify, export, rebuild. Layer paths, object names, materials, colours
and block structure survive the trip, carried as attributes in Houdini and as
user strings in the `.3dm`.

To put geometry back where it started after **Transform to Centroid** moved it,
set the exporter's **Translate** to *Using xform* and point **Position
Reference** at the importer. See [Translate](#translate).

---

# Rhino 3DM Import

Reads a `.3dm` file and outputs Houdini geometry. The file is cached by path and
modification time; only the current file is held in memory. Press **Reload** to
force a re-read.

## Outputs

| # | Name | Contents |
| --- | --- | --- |
| 1 | Scene (Non-Block) | Everything not inside a block |
| 2 | Block Definitions | Block contents, at the definition origin |
| 3 | Block Instance Points | One point per placed block, with transform attributes |
| 4 | Exploded Blocks (For View) | Block contents baked at their placed positions |

Outputs 2 and 3 are the pair to feed **Copy to Points**. Output 4 is the same
result pre-baked, for viewing and for checking that output 3's transforms are
correct; it is not needed to reconstruct the scene.

Output 4 receives the same **Transform to Centroid** offset as output 1, so the
two stay aligned at any scale.

Nested blocks are flattened. A block placed inside another block is walked
through to its leaf geometry, to a depth limit of 32.

## Import Parameters

### Import File

#### 3dm File

Path to the `.3dm` file.

#### Reload

Clears the file cache and re-cooks the four internal readers. Use after editing
the file outside Houdini.

### Position and Scale

#### Scale

Multiplier applied to all geometry. Rhino files are commonly in millimetres;
`0.001` brings a millimetre file into metres.

The value is also written to output 1 as a detail attribute named `scale`, which
is what the exporter's **Reference Importer Scale** reads.

Cannot be 0.

#### Transform to Centroid

Moves the imported geometry so it is centred in X and Z and sits on the ground
plane in Y. Useful for files modelled at a site coordinate far from the origin,
where single-precision viewport maths breaks down.

The offset is stored as a 16-float detail attribute named `xform` on outputs 1,
3 and 4. The exporter can read it back to restore the original position.

### Options

#### Z-up to Y-up

Converts Rhino's Z-up coordinate system to Houdini's Y-up. Rhino `(x, y, z)`
becomes Houdini `(x, z, -y)`.

Turn off to keep Rhino coordinates unchanged.

#### Layer Filter

Pattern matched against the full layer path, using `fnmatch` syntax. `*` imports
everything.

Layer paths use `::` as the separator, matching Rhino. A pattern of
`Structure::*` imports that layer and its children.

Applies to top-level objects only. Objects inside a block definition follow their
block.

#### Include Hidden

Imports objects whose own visibility or whose layer visibility is off. By default
both must be visible.

#### Colour

| Value | Meaning |
| --- | --- |
| None | No `Cd` attribute is created |
| Layer Colour | The object's layer colour |
| Display Colour | The object's display colour, resolving by-material and falling back to the layer |

Pure black is read as white, since a Rhino object with no colour set reports
black.

`Cd` is created as a primitive attribute and then promoted to points by averaging
before welding, so on the outputs it is a **point** attribute on solids and
meshes, and a point attribute on points. On curves it is a primitive attribute.

#### Import Solids & Meshes / Import Curves / Import Points

Switches for the three geometry kinds. Turning one off leaves its attributes in
place but produces no geometry, so downstream nodes do not error.

### Solids & Meshes

Disabled when **Import Solids & Meshes** is off.

#### Breps & Extrusions

Imports Breps and Extrusions by taking the render mesh of each face. One mesh per
Brep face.

#### Meshes

Imports Rhino mesh objects.

#### Weld Faces

Fuses coincident points, so a Brep arrives as one connected mesh rather than a
pile of disconnected faces.

Required by **Dissolve Flat Faces** — a dissolve needs shared edges to work with.

#### Dissolve Flat Faces

Merges adjacent triangles that lie in the same plane back into single polygons.
A Rhino planar face that arrived as a triangulated render mesh becomes one
n-sided polygon again.

This is the parameter that makes imported geometry workable. Without it a simple
box arrives as 12 triangles.

#### Flat Angle

Maximum angle between face normals for two faces to count as coplanar. Default
`1` degree. Raise it to merge across slight curvature; lower it to be strict.

#### Keep Rhino Face Boundaries

Prevents the dissolve from crossing from one Rhino Brep face into another, using
the per-face `face_id` attribute. Two coplanar Rhino faces that happen to be
adjacent stay separate.

Turn off to merge them into one polygon.

#### Skip Faces With Holes

Leaves regions containing holes triangulated rather than dissolving them. A
dissolved region with a hole produces a polygon with a bridging slit, which some
downstream operations dislike.

#### Remove Inline Points

Deletes points that sit on a straight run along a dissolved region's boundary,
leaving only real corners.

#### Dissolve Layers

Space-separated patterns limiting the dissolve to certain layers. `*` dissolves
everywhere. Use this to keep a heavy curved layer triangulated while cleaning up
the rest.

### Curves

Disabled when **Import Curves** is off.

#### Curve Output

| Value | Meaning |
| --- | --- |
| Polygon | Sample the curve into a polyline |
| NURBS | Rebuild as a Houdini NURBS curve where possible |

NURBS output is verified before it is kept: the rebuilt curve is sampled at seven
parameters and compared against the Rhino curve. If any sample is further apart
than **Chord Tolerance**, the curve falls back to polyline sampling. Degree must
be between 1 and 10.

Rational curves — circles, arcs, conics — are handled, including builds of
rhino3dm that store control points in homogeneous coordinates.

#### Chord Tolerance

Maximum distance between the sampled polyline and the true curve, in Houdini
units. Smaller gives more points.

Sampling is exact at every knot and adaptive in between, to a recursion depth of
12.

#### Max Segment Length

Forces subdivision of any segment longer than this, in Houdini units, even where
the curve is straight enough not to need it. `0` disables.

#### Min Divisions per Span

Minimum samples per knot span for curves of degree 2 and above. Degree 1 curves
are never subdivided beyond their knots.

#### Even Resample

Resamples polyline curves to a uniform segment length after sampling. Discards
the adaptive point distribution in favour of regular spacing.

#### Even Segment Length

Segment length for **Even Resample**, in Houdini units.

#### Remove Inline Points

Removes points along straight runs of a polyline.

#### Inline Angle Threshold

Maximum turn angle, in degrees, for a point to count as inline and be removed.

#### Unroll Closed Curves

Opens closed polyline curves by duplicating the first point at the end. Some
downstream operations expect an explicit closing point.

## Attributes created by the importer

### Solids and meshes (polygons)

| Name | Class | Type | Description |
| --- | --- | --- | --- |
| `layer` | primitive | string | Full Rhino layer path, `::` separated |
| `material` | primitive | string | Render material name, resolved from object or layer |
| `objname` | primitive | string | Rhino object name, empty if unnamed |
| `name` | primitive | string | `piece_<n>`, one per source Rhino object |
| `rhino_id` | primitive | string | Rhino object GUID |
| `block` | primitive | string | Block definition name, empty outside blocks |
| `face_id` | primitive | int | Index of the source Brep face |
| `instance_id` | primitive | int | Index of the block instance this came from, `-1` outside blocks |
| `uv` | vertex | vector3 | Texture coordinates, `(0,0,0)` where the mesh has none |
| `Cd` | point | vector3 | Colour, only when **Colour** is not *None* |

`name` is a stable per-object identifier suitable for **Copy to Points**,
packing, or `name`-based piece workflows. It is also the exporter's fallback for
the Rhino object name when `objname` is empty.

### Curves

| Name | Class | Type | Description |
| --- | --- | --- | --- |
| `layer` | primitive | string | Full Rhino layer path |
| `objname` | primitive | string | Rhino object name |
| `rhino_id` | primitive | string | Rhino object GUID |
| `block` | primitive | string | Block definition name |
| `curve_type` | primitive | string | Rhino class name: `LineCurve`, `ArcCurve`, `NurbsCurve`, `PolyCurve`, … |
| `degree` | primitive | int | Degree of the source Rhino curve |
| `instance_id` | primitive | int | Block instance index, `-1` outside blocks |
| `Cd` | primitive | vector3 | Colour, only when **Colour** is not *None* |

`curve_type` records what the curve was in Rhino even when it has been sampled to
a polyline. The exporter's `kind` tagging uses its presence to recognise curves.

### Points

| Name | Class | Type | Description |
| --- | --- | --- | --- |
| `layer` | point | string | Full Rhino layer path |
| `objname` | point | string | Rhino object name |
| `rhino_id` | point | string | Rhino object GUID |
| `block` | point | string | Block definition name |
| `instance_id` | point | int | Block instance index, `-1` outside blocks |
| `Cd` | point | vector3 | Colour, only when **Colour** is not *None* |

Rhino point clouds are expanded to one Houdini point each.

### Block instance points (output 3)

| Name | Class | Type | Description |
| --- | --- | --- | --- |
| `block` | point | string | Block definition name — matches `block` on output 2 |
| `layer` | point | string | Layer path of the instance reference |
| `objname` | point | string | Rhino object name of the instance |
| `rhino_id` | point | string | Rhino object GUID |
| `instance_id` | point | int | Index matching `instance_id` on output 4 |
| `orient` | point | vector4 | Rotation quaternion `(x, y, z, w)` |
| `scale` | point | vector3 | Non-uniform scale, per local axis |
| `rhino_xform` | point | float[16] | The full instance transform, row-major, for cases decomposition cannot express |

`P`, `orient` and `scale` are exactly the attribute set **Copy to Points** reads.
Wiring output 2 into the first input and output 3 into the second reconstructs
the placed blocks.

Mirrored instances are detected by a negative determinant and expressed as a
negative X scale. Shear cannot be expressed in `orient` and `scale`; use
`rhino_xform` for those.

### Detail

| Name | Outputs | Type | Description |
| --- | --- | --- | --- |
| `scale` | 1 | float | The **Scale** parameter value |
| `xform` | 1, 3, 4 | float[16] | Offset applied by **Transform to Centroid** |

## Groups created by the importer

| Group | Class | Contents |
| --- | --- | --- |
| `layer_<name>` | primitive, point | One group per Rhino layer |
| `block_<name>` | primitive, point | One group per block definition, on outputs 2 and 3 |
| `poly_curves` | primitive | All curves sampled to polylines, excluding NURBS output |

Group names are sanitised: any character outside `A-Z a-z 0-9 _` becomes `_`. A
layer called `Level 01::Walls` produces `layer_Level_01__Walls`.

`poly_curves` always exists, even when empty, so a downstream Resample never
errors.

---

# Rhino 3DM Export

Writes `.3dm` from Houdini geometry. Writing happens when the **Export** button
is pressed, not on cook. The node's own output is a preview of what will be
written.

Solids are written as loose face outlines. Run [`HouRebuild.py`](#hourebuildpy)
in Rhino afterwards to turn them into joined polysurfaces.

## Inputs

| # | Name | Expected contents |
| --- | --- | --- |
| 1 | Scene (Non-Block) | Polygons, curves and points, mixed |
| 2 | Block Definitions | Geometry to become block definitions, at the origin |
| 3 | Block Instance Points | Points carrying `block` and transform attributes |
| 4 | — | Unused |

Input 1 is sorted internally. Each primitive is tagged `face` or `curve`:
anything that is not a closed polygon with three or more vertices is a curve, as
is anything carrying a `curve_type` attribute. Points belonging to no primitive
become exported Rhino points.

Input 2 is merged with input 1 and tagged `is_block_def = 1`. Geometry so tagged
is scaled but never moved, because block definitions live at the definition
origin.

Input 3 is normalised before use — see [Block instance
points](#block-instance-points).

## How pieces are formed

A *piece* is one Rhino object. The exporter derives pieces from connectivity, not
from any incoming attribute:

1. **Optional weld.** A Fuse merges points closer than **Weld Distance**, so
   faces that only touch become connected.
2. **Connectivity.** Connected primitives are given a shared `piece_id`.
3. **Classify.** Each face is tested for planarity against **Planar Tolerance**.
   A piece is written as surfaces when every face is planar and the piece has no
   more than **Max Faces Per Surface Piece** faces. Otherwise the whole piece is
   written as a single Rhino mesh.

Any `piece_id` on incoming geometry is overwritten. To control what becomes one
Rhino object, control what is connected — weld or split geometry upstream.

Within a surface piece, the exporter then:

- Groups coplanar adjacent faces into flat regions and writes each region as one
  outline.
- Detects faces with holes and writes the outer loop plus each inner loop,
  tagged with a shared `region` user string so Rhino can build one trimmed
  surface from them.
- Detects rows of facets approximating a cylinder, cone or fillet and fits a
  single NURBS surface, when **Curved Surfaces** is on.

## Export Parameters

### Export File

#### File

Output path. Written when **Export** is pressed.

#### Export

Writes the file. Errors are raised as node errors; the summary is printed as a
node message and to the Python shell.

### Position and Scale

#### Reference Importer Scale

Takes the scale from a Rhino 3DM Import node instead of the **Scale** parameter,
and inverts it, so a file imported at `0.001` is written back at `1000`.

Reads the detail attribute `scale` from the node named in **Scale Importer
Path**. Errors if that node has no `scale` parameter, or if it is 0.

#### Scale

Multiplier applied on the way out. Disabled when **Reference Importer Scale** is
on. Cannot be 0.

The node's output geometry reflects whichever of the two is active, so what
travels downstream matches what is written.

#### Scale Importer Path

Path to the Rhino 3DM Import node whose scale is to be inverted. Hidden unless
**Reference Importer Scale** is on.

#### Translate

| Value | Meaning |
| --- | --- |
| None | Write geometry where it is |
| Using xform | Undo a stored transform before writing |

*Using xform* reads a 16-float detail attribute named `xform` from the node in
**Position Reference**, inverts it, and applies it to all geometry. Its purpose
is to undo the importer's **Transform to Centroid** so exported geometry lands
back at the original Rhino coordinates.

Errors if the referenced node has no geometry, no `xform` detail attribute, or an
`xform` that is not 16 floats.

#### Position Reference

Path to the node carrying `xform`. Normally the Rhino 3DM Import node that moved
the geometry in the first place.

### Options

#### Y-up to Z-up

Converts Houdini's Y-up back to Rhino's Z-up. The inverse of the importer's
**Z-up to Y-up**. Leave both on, or both off.

#### Export Solids & Meshes / Export Curves / Export Points

Switches for the three geometry kinds.

#### Export Vertex Colours

Writes object colours into the `.3dm`, and per-vertex colours on meshes where the
rhino3dm build supports them.

#### Colour Attribute

Name of the colour attribute to read. Default `Cd`.

The attribute is looked for on vertices, then points, then primitives, then the
detail, which is the order Houdini itself resolves colour for display. The first
class that carries it wins. The export report names the class that was used.

Vertex and point colours can vary within a piece and so can produce a gradient;
primitive and detail colours cannot.

#### Multiple Colour Forces Mesh

When a piece carries more than one colour, writes it as a mesh with vertex
colours rather than as surfaces, because a Rhino surface or polysurface holds
only one colour.

Turn off to keep surfaces and accept that the piece takes a single colour.

Colour variation is measured against a threshold of 2/255 per channel.

### Solids

Disabled when **Export Solids & Meshes** is off.

#### Weld Before Connectivity

Fuses coincident points before pieces are found. Without it, faces that touch but
do not share points become separate Rhino objects.

#### Weld Distance

Fuse radius, in Houdini units, before scaling.

#### Planar Tolerance

Maximum deviation from the face plane, in Houdini units, for a face to count as
planar. A piece containing any non-planar face is written as a mesh.

Raise it to push borderline geometry down the surface path; lower it if slightly
warped faces are producing bad surfaces.

#### Max Faces Per Surface Piece

Pieces with more faces than this are written as a single mesh instead of as
surfaces. Default `5000`.

This is the control for heavy tessellated geometry. A sphere with 20,000 facets
has no useful surface representation and would otherwise produce 20,000 Rhino
objects. Lower the value to send more geometry down the mesh path.

#### Triangulate

Divides all faces into triangles before export.

#### Curved Surfaces

Detects rows of coplanar facets that approximate a cylinder, cone or fillet, and
fits one NURBS surface instead of writing each facet. Turns a tessellated fillet
back into a single clean surface.

#### Smooth Angle

Maximum angle between adjacent face normals, in degrees, for the join to be
considered smooth and therefore part of a curved run rather than a real edge.

#### Arc Tolerance

Maximum deviation, in Houdini units, between a fitted arc and the facets it
replaces. A run needs at least three facets to become an arc.

### Curves

Disabled when **Export Curves** is off.

#### Curve Mode

| Value | Meaning |
| --- | --- |
| Exact Polyline | Write every point as a polyline vertex |
| Smart | Fit lines, arcs and splines to the points |

*Smart* produces curves a Rhino user can work with. *Exact Polyline* produces
exactly the points that were in Houdini.

#### Fit Tolerance

Maximum deviation between a fitted segment and the source points, in Houdini
units. Disabled in *Exact Polyline* mode.

#### Corner Angle

Turn angle, in degrees, above which a point is treated as a hard corner and
splits the curve into separate fitted segments.

#### Max Segments Before Spline

A run that would need more than this many line or arc segments is written as a
single spline instead.

#### Max Control Points

Upper limit on the control points of a fitted spline. A fit needing more falls
back to a polyline.

#### Use Arcs

Allows arc fitting. Turn off to restrict fitting to lines and splines.

#### Use Splines

Allows spline fitting. Turn off to restrict fitting to lines and arcs.

#### Join Into Single NURBS

Writes each curve as one joined NURBS curve rather than as separate segments.
Disabled in *Exact Polyline* mode.

When off, segments of one Houdini curve are tagged with a shared `curve_join`
user string, and `HouRebuild.py` joins them in Rhino.

## Attributes read by the exporter

None are required. Each is used if present and ignored if not.

### On input 1 and input 2

| Name | Class | Type | Use |
| --- | --- | --- | --- |
| `layer` | primitive, point | string | Rhino layer path. Nested layers are created by splitting on `::`. Empty or missing becomes `Default` |
| `objname` | primitive, point | string | Rhino object name |
| `name` | primitive, point | string | Fallback object name when `objname` is empty |
| `material` | primitive, point | string | Written as a `material` user string, not as a Rhino render material |
| `block` | primitive, point | string | Block definition name. Read only where `is_block_def` is 1 |
| `is_block_def` | primitive, point | int | `1` marks block definition geometry. Set automatically on input 2 |
| `curve_type` | primitive | string | Forces a primitive to be treated as a curve |
| *colour* | vertex, point, primitive, detail | vector3 | Named by **Colour Attribute** |

`layer` accepts any string. `Site::Buildings::Tower` creates three nested Rhino
layers. There is no need for the layer to have come from an import.

### Block instance points

Input 3 is normalised through the same attribute set **Copy to Points** uses, so
a point stream built for Copy to Points works unchanged:

| Name | Type | Use |
| --- | --- | --- |
| `block` | string | Block definition name. **Required** — points without it are skipped |
| `P` | vector3 | Position |
| `orient` | vector4 | Rotation quaternion. Takes priority over `N`/`v`/`up` |
| `N` | vector3 | Direction, used when `orient` is absent |
| `v` | vector3 | Direction, used when neither `orient` nor `N` is present |
| `up` | vector3 | Up vector, combined with `N` or `v` |
| `rot` | vector4 | Extra rotation, multiplied onto the result |
| `scale` | vector3 | Non-uniform scale |
| `pscale` | float | Uniform scale, multiplied into `scale` |
| `trans` | vector3 | Extra translation |
| `pivot` | vector3 | Pivot offset, applied before rotation |
| `layer` | string | Layer for the instance |

`pscale` and `scale` combine: a point with `scale = (2, 1, 1)` and
`pscale = 3` is placed at `(6, 3, 3)`.

Transform order matches **Copy to Points**: scale in local axes, then rotate,
then translate.

Instance points are only read when the input carries a `block` attribute.

## User strings written to the `.3dm`

These are how information survives in the file and how `HouRebuild.py` knows what
to rebuild. They are visible in Rhino under **Properties › Details**.

| Key | On | Meaning |
| --- | --- | --- |
| `piece` | Face outlines | Which Rhino object the outline belongs to. Outlines sharing a value are joined into one polysurface |
| `region` | Face outlines | Loops of one face with holes. All loops sharing a value build one trimmed surface |
| `material` | All | Material name from the `material` attribute |
| `block` | Block definition geometry | Definition name to create |
| `block_instance` | Instance points | Present and set to `1` on every instance point |
| `orient` | Instance points | Quaternion as `x,y,z,w` |
| `scale` | Instance points | Scale as `x,y,z` |
| `curve_join` | Curve segments | Segments sharing a value are joined into one curve |
| `color` | All, when colour is exported | `r,g,b` as 0–255 integers |

`color` is written in addition to the Rhino object colour. Reading an object's
colour back in Rhino resolves through its layer, which cannot distinguish "no
colour set" from black. The user string is the unambiguous record, and the
rebuild script reads it in preference.

Objects are also given their Rhino layer, name and display colour directly.

## The report

The **Export** button prints a single line summarising the write. It is the
fastest diagnosis available.

```
-> Export_03.3dm (scale 100, undoing xform from /obj/Demo/rhino_3dm_import1)
 | solids: 0 mesh, 0 mesh for colour, 15 surface pieces (8 curved, 0 arc
 outlines, 3171 flat, 5 holed, 0 non-planar) | curves: 1 smart, 0 NURBS, 0
 polyline, 1 fell back | 9 points | 0 block instances | colour: 'Cd' on vertex
```

| Field | Meaning |
| --- | --- |
| `mesh` | Pieces written as meshes, because of planarity or face count |
| `mesh for colour` | Pieces written as meshes to preserve a colour gradient |
| `surface pieces` | Pieces written as surfaces |
| `curved` | Facet runs fitted to NURBS surfaces |
| `arc outlines` | Outlines containing fitted arcs |
| `flat` | Planar region outlines |
| `holed` | Faces with holes, written as multiple loops |
| `non-planar` | Faces that failed the planarity test |
| `smart` / `NURBS` / `polyline` / `fell back` | How each curve was written |
| `colour` | Which attribute class the colour came from, or that none was found |
| `smooth facet joins not converted to arcs` | Appears when curved detection left smooth joins unconverted |

`colour: 'Cd' on NOTHING - no colour exported` means the attribute named in
**Colour Attribute** is not on the geometry in any class.

---

# HouRebuild.py

Runs **inside Rhino**, not Houdini. It converts an exported `.3dm` from loose
outlines into finished geometry.

## Running it

Either `RunPythonScript` and pick the file, or set up [the
alias](#the-rhino-alias) and type one word.

## What it does

In order:

1. **Trimmed surfaces with holes.** Outlines sharing a `region` user string are
   passed to `AddPlanarSrf` together, producing one trimmed surface.
2. **Ordinary face outlines.** Three- and four-corner outlines become surfaces
   directly; larger outlines and outlines containing arcs go through
   `AddPlanarSrf`. Source curves are deleted.
3. **Joining.** Surfaces sharing a `piece` user string are joined into a
   polysurface. The script reports how many came out as closed solids, how many
   as open polysurfaces, and how many fragmented into more than one object.
4. **Curve joining.** Segments sharing a `curve_join` user string are joined.
   Only relevant when the export could not write a single NURBS curve.
5. **Block definitions.** Objects sharing a `block` user string become one block
   definition.
6. **Block instances.** Each point carrying `block_instance` is replaced by a
   placed block, using its `orient` and `scale` user strings. Duplicate
   placements of the same definition at the same transform are removed.

## Colour

Colour is taken from the `color` user string the exporter writes, not from the
object's displayed colour: reading that back resolves through the layer, which
would stamp every rebuilt surface with the layer colour.

A polysurface carries one colour. When the faces of a piece disagree, the script
keeps one and reports how many pieces lost a gradient. Turn on **Multiple Colour
Forces Mesh** in the exporter to keep those gradients as meshes instead.

## Reading the report

```
Tolerance: 0.001   Units: Millimeters
Trimmed surfaces with holes: 5
Surfaces built: 3184   failed: 2
   2 failed at 4 corners
Joined: 14 solids, 1 open polysurfaces, 0 fragmented
Block definitions created: 3
Block instances placed: 42, skipped: 0, duplicates removed: 0
```

A few failures on messy input are normal. The breakdown says whether they failed
as holed regions, as outlines containing arcs, or by corner count.

`open polysurfaces` means a piece did not close into a solid — usually a missing
or failed face. `fragmented` means a piece joined into more than one object,
which indicates faces that do not meet within the document tolerance.

The script uses the document's absolute tolerance, printed at the top. If joining
fails widely, check that tolerance against the scale of the geometry.

---

# Attribute reference

Summary of every attribute name this package reads or writes.

| Name | Written by import | Read by export |
| --- | --- | --- |
| `layer` | ✓ | ✓ |
| `material` | ✓ | ✓ |
| `objname` | ✓ | ✓ |
| `name` | ✓ | ✓ (fallback for `objname`) |
| `block` | ✓ | ✓ |
| `rhino_id` | ✓ | — |
| `face_id` | ✓ | — |
| `instance_id` | ✓ | — |
| `curve_type` | ✓ | ✓ |
| `degree` | ✓ | — |
| `uv` | ✓ | — |
| `Cd` | ✓ | ✓ (name configurable) |
| `orient` | ✓ | ✓ |
| `scale` (point) | ✓ | ✓ |
| `rhino_xform` | ✓ | — |
| `scale` (detail) | ✓ | ✓ (via **Reference Importer Scale**) |
| `xform` (detail) | ✓ | ✓ (via **Translate**) |
| `is_block_def` | — | ✓ |
| `pscale`, `rot`, `trans`, `pivot`, `N`, `v`, `up` | — | ✓ |
| `piece_id`, `piece_kind` | — | internal to the exporter |

`piece_id` and `piece_kind` are created inside the export node and are not read
from incoming geometry.

---

# Troubleshooting

**Tab search finds no Rhino 3DM nodes.**
Open Windows › Package Browser and look for Houdini3DM. If absent,
`Houdini3DM.json` is not in the packages folder. If present, the path inside it
is wrong.

**"No module named rhino3dm", or "'module' object has no attribute 'r3'".**
The same fault. The asset's shared Python module fails on its import line and
everything downstream reports a missing attribute on a module that never
finished loading. Run **Check Install** and read the Python version. The package
needs a `pythonX.Ylibs` folder matching it exactly and ships 3.9 through 3.13.

**"Limited commercial (Indie) assets not allowed with commercial licenses".**
This build is Indie only and cannot be loaded by a commercial Houdini licence.

**Export produced loose surfaces, not solids.**
Expected. Run the rebuild — see [HouRebuild.py](#hourebuildpy).

**Nothing comes out coloured.**
Read the `colour:` field of the export report. If it says no colour was
exported, either **Export Vertex Colours** is off or **Colour Attribute** names
an attribute that is not on the geometry in any class.

**Everything comes out black in Rhino.**
Fixed in 1.1.0, and both halves had to change. Confirm the Rhino alias points at
the 1.1.0 `HouRebuild.py`, not an older copy.

**A piece came out as one mesh instead of surfaces.**
Either a face failed the planarity test or the piece exceeds **Max Faces Per
Surface Piece**. The report counts both. Raise **Planar Tolerance**, or raise the
face limit.

**One Rhino object arrived as many, or many as one.**
Pieces follow connectivity. Turn on **Weld Before Connectivity** and raise
**Weld Distance** to merge; split the geometry upstream to separate.

**Blocks are placed at the wrong position or scale.**
Check that input 3 carries `orient` and `scale`, and that any `pscale` is
intended — it multiplies into `scale`. Sheared instances cannot be expressed in
`orient` and `scale`; use `rhino_xform` from the importer.

**`hou` does nothing in Rhino.**
The path in the alias does not match the install folder. Correct it in Tools ›
Options › Aliases, keeping the quotes.

---

# Licensing

Licence terms are in `EULA.txt`.

Includes [rhino3dm](https://github.com/mcneel/rhino3dm) by Robert McNeel &
Associates, MIT licence — see `Houdini3DM/rhino3dm-LICENSE.txt`.

Rhino and Rhinoceros are trademarks of Robert McNeel & Associates. This package
is not affiliated with or endorsed by Robert McNeel & Associates.
