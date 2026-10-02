# TechDraw / BIM technical drawings

Covers building a 2D technical/classification drawing in FreeCAD — walls, coloured
zones, an ISO title block, a spreadsheet-driven schedule — as a reusable pattern.
Built out validating this against a real job: `09 - Okracandle/Property/Unit 01 Zone
Layout.FCStd`. Read this before starting a new drawing of this kind; copy the pattern
rather than re-deriving the APIs from scratch.

There is no dedicated Draft/Arch/TechDraw MCP tool — everything here runs through
`execute_python` scripting FreeCAD's own `Draft`, `Part`, `TechDraw`, `TechDrawGui`
Python modules.

## The template

`11 - Arkoatelier/FreeCAD Templates/Technical-Drawing-A3-Landscape.svg` — a generic,
white-label ISO 5457 A3-landscape title block (adapted from FreeCAD's own shipped
`A3_Landscape_ISO5457_advanced.svg`). Fields: Owner, Site/Location, Title +
supplementary titles, Created by / Approved by, Document type, Document status,
Drawing number, Revision, Sheet, Language, Issue date, Scale, Classification. Not
Okracandle-branded — blank/generic so it serves any future paying job.

Attach it to a page:

```python
tmpl = doc.addObject("TechDraw::DrawSVGTemplate", "Template")
tmpl.Template = "11 - Arkoatelier/FreeCAD Templates/Technical-Drawing-A3-Landscape.svg"
page = doc.addObject("TechDraw::DrawPage", "Page")
page.Template = tmpl
```

Fields are populated by writing the whole dict at once — you cannot set one key in
isolation via property access, read-modify-write the dict:

```python
et = dict(tmpl.EditableTexts)
et["revision_index"] = "K"
tmpl.EditableTexts = et       # must reassign the whole dict, not mutate in place
```

Known field keys on this template: `legal_owner_1`, `part_material` (relabelled
"Site / Location"), `title`, `supplementary_title_1/2`, `creator`, `approval_person`,
`document_type`, `document_status`, `drawing_number`, `revision_index`,
`sheet_number`, `language_code`, `date_of_issue`, `scale`, `general_tolerances`
(relabelled "Classification"), `responsible_department`.

**Gotcha — text overflow.** Fields have a fixed box width with no wrapping. A long
value (a full site address, a long classification phrase) overlaps the neighbouring
field's text with no error or warning — it just renders garbled. Keep title-block
values short; abbreviate ("Marcam Ind. Complex" not "Marcam Industrial Complex") and
verify the export by eye every time a value changes.

**Not yet solved: live expression binding into EditableTexts.** `EditableTexts` is a
Map property, and `setExpression` does not bind into individual keys of a map the way
it does an ordinary string/quantity property. In the current build the title block is
populated by *pushing* Spreadsheet values into `EditableTexts` explicitly, once, at
build time — not through a live FreeCAD expression that recomputes automatically when
the Spreadsheet cell changes. If you edit a cell after the fact (e.g. bump the
Revision), you must re-run the push-to-EditableTexts step yourself before re-exporting
— see the worked macro's `sync_titleblock()` step. Don't assume changing the
Spreadsheet alone updates the title block; it doesn't, yet.

## Data model: one Spreadsheet per drawing

`Spreadsheet::Sheet` object named `Parameters`. Two conventions in one sheet:

- **Identity block** (rows 1–8 or so): `Client`/`Site`/`Drafter`/`DrawingNo`/`Revision`
  and any driving dimensions (`UnitWidth_mm`, `UnitDepth_mm`).
- **Schedule table** (a fixed cell range, e.g. `A10:C15`): headers in row 1 of the
  range, one data row per zone/item after.

Bind a `TechDraw::DrawViewSpreadsheet` to the schedule range — this one genuinely is
live: it re-renders from the sheet on every recompute.

```python
tbl = doc.addObject("TechDraw::DrawViewSpreadsheet", "ZoneSchedule")
tbl.Source = sheet
tbl.CellStart = "A10"
tbl.CellEnd = "C15"
tbl.Font = "osifont"
tbl.TextSize = 4        # NOT FontSize - that property doesn't exist on this type
tbl.Scale = 0.6          # default column widths (100mm!) and row heights (30mm!) are
                         # enormous - always set an explicit small Scale or it renders
                         # either off-page or illegibly huge
tbl.X = 320
tbl.Y = 90
page.addView(tbl)        # REQUIRED - addObject() alone does NOT add it to page.Views.
                         # Missing this step is silent: no error, the table Symbol
                         # generates correctly, it simply never appears in the export.
                         # Always verify with `tbl in page.Views` / `page.Views`.
doc.recompute()
page.ViewObject.doubleClicked()   # see "first-render gotcha" below
doc.recompute()
```

## Floor plan geometry — SUPERSEDED 2026-09-05: use `Sketcher::SketchObject`, not solids

**The section below (solids-only) was the original working method and is kept for the
historical record of why it was tried, but it has been superseded.** The real problem
it was solving was never "solids vs wires" as such — it was that a **static,
one-shot-scripted** shape (a Draft rectangle turned into a `Part::Feature` blob with a
baked `Shape`) has no editable parameters: resizing a zone meant re-running a build
script, and `Unit 01 Zone Layout.FCStd`'s own "Editing this drawing interactively"
section (below) found this was the single biggest gap against "true parametric,
GUI-editable" — confirmed directly against Erik's actual complaint that everything was
one-shot `execute_python` pushes with no normal FreeCAD drag/resize workflow.

**What actually works and is now the standing method, validated by direct GUI
round-trip on `Unit 01 Zone Layout.FCStd` 2026-09-05 (phase 1 of a staged
solids→Sketcher migration):** build each zone and the wall frame as its own
**fully-constrained `Sketcher::SketchObject`** (closed 4-line rectangle, named
`DistanceX`/`DistanceY` constraints for position and size), add the sketches directly
to a `TechDraw::DrawViewPart.Source` list — **no extrusion, no `Part::Feature`
wrapper, the sketch itself projects.** This is a real, meaningful change from the
solids-only guidance below: a plain `Sketcher::SketchObject` (not a bare Draft
wire/rectangle) survived `Source`-list mutation, `Direction` changes and repeated
recompute pixel-identical in direct testing — the flat-wire fragility documented below
was specific to Draft geometry, not to Sketcher geometry generally.

**Worked pattern** (rectangle with position + size as independently named,
GUI-editable constraints):

```python
import Part, Sketcher
sk = doc.addObject("Sketcher::SketchObject", "Sketch_H")
sk.addGeometry(Part.LineSegment(App.Vector(0,0,0), App.Vector(1,0,0)), False)
sk.addGeometry(Part.LineSegment(App.Vector(1,0,0), App.Vector(1,1,0)), False)
sk.addGeometry(Part.LineSegment(App.Vector(1,1,0), App.Vector(0,1,0)), False)
sk.addGeometry(Part.LineSegment(App.Vector(0,1,0), App.Vector(0,0,0)), False)
sk.addConstraint(Sketcher.Constraint('Coincident', 0, 2, 1, 1))
sk.addConstraint(Sketcher.Constraint('Coincident', 1, 2, 2, 1))
sk.addConstraint(Sketcher.Constraint('Coincident', 2, 2, 3, 1))
sk.addConstraint(Sketcher.Constraint('Coincident', 3, 2, 0, 1))
sk.addConstraint(Sketcher.Constraint('Horizontal', 0))
sk.addConstraint(Sketcher.Constraint('Horizontal', 2))
sk.addConstraint(Sketcher.Constraint('Vertical', 1))
sk.addConstraint(Sketcher.Constraint('Vertical', 3))
sk.addConstraint(Sketcher.Constraint('DistanceX', -1, 1, 0, 1, x0))   # position: origin -> pt1
sk.addConstraint(Sketcher.Constraint('DistanceY', -1, 1, 0, 1, y0))
i = sk.addConstraint(Sketcher.Constraint('DistanceX', 0, 1, 0, 2, width))
sk.renameConstraint(i, 'H_Width')     # NAMED constraints, so Erik can find/edit them
i = sk.addConstraint(Sketcher.Constraint('DistanceY', 1, 1, 1, 2, depth))
sk.renameConstraint(i, 'H_Depth')
sk.ViewObject.ShapeColor = (0.66, 0.22, 0.13)   # classification colour, per-sketch
```

**Gotcha — `Sketcher.Constraint('DistanceX', 'Name', 0, 1, 0, 2, value)` does NOT
work.** The named-constraint constructor form only accepts a bare type + index (or
nothing) — passing a name string inline raises `TypeError: Invalid parameters`. Add
the constraint unnamed, then `sk.renameConstraint(index, 'Name')` as a separate call
(exactly as shown above). This is a real API gotcha, not a style choice.

**Wall frame as two nested closed rectangles in ONE sketch** (outer footprint + inner
face inset by the wall thickness, both closed 4-line loops, geometry indices 0-3 outer
/ 4-7 inner):

```python
# ... outer rect geo 0-3 built the same way as above ...
# ... inner rect geo 4-7 built the same way ...
i = wf.addConstraint(Sketcher.Constraint('DistanceX', 0, 1, 0, 2, 7000))
wf.renameConstraint(i, 'Overall_Width')
i = wf.addConstraint(Sketcher.Constraint('DistanceY', 1, 1, 1, 2, 12000))
wf.renameConstraint(i, 'Overall_Depth')
i = wf.addConstraint(Sketcher.Constraint('DistanceX', 0, 1, 4, 1, 150))   # outer pt1 -> inner pt1
wf.renameConstraint(i, 'WallThickness_X')
i = wf.addConstraint(Sketcher.Constraint('DistanceY', 0, 1, 4, 1, 150))
wf.renameConstraint(i, 'WallThickness_Y')
i = wf.addConstraint(Sketcher.Constraint('DistanceX', 4, 1, 4, 2, 6700))  # inner rect's own size
wf.renameConstraint(i, 'Inner_Width')
i = wf.addConstraint(Sketcher.Constraint('DistanceY', 5, 1, 5, 2, 11700))
wf.renameConstraint(i, 'Inner_Depth')
```

Then project all six sketches together, unmodified, no extrusion:

```python
fp = doc.addObject("TechDraw::DrawViewPart", "FloorPlan")
fp.Source = [wall_sketch, sketch_H, sketch_F, sketch_S, sketch_W, sketch_C]
fp.ScaleType = "Custom"; fp.Scale = 0.01
fp.Direction = App.Vector(0,0,1); fp.XDirection = App.Vector(1,0,0)
page.addView(fp)
fp.X = 90.0; fp.Y = 201.0   # set AFTER addView() — same gotcha as everywhere else on this page
doc.recompute()
page.ViewObject.doubleClicked()
doc.recompute()
```

**The critical geometric rule this migration exists to enforce: zone rectangles must
sit inside the wall's INNER face, never touch the outer wall coordinates.** The bug
being fixed was zones dimensioned against the building's *outer* footprint (e.g. a
zone corner at `x=7000` when the outer wall is also at `x=7000`) while the wall itself
consumed 150mm of that same space inward — an invisible-until-you-check overlap. Fix:
every zone boundary that faces an outer wall must be positioned at the **inner face**
coordinate (`wall_thickness` in from the outer edge), not the outer envelope
coordinate. Verify this arithmetically before trusting the render — sum every zone's
extent along an axis and confirm it does not exceed `(outer_envelope - 2*thickness)`
for that axis, **for zones that are meant to be contiguous along that axis** (see the
depth-budget gotcha immediately below — this sum frequently does not have enough
slack once a real wall thickness is subtracted, even when the original solids-based
design summed exactly to the outer envelope with zero allowance for wall thickness).

**Gotcha found doing this migration — a zone schedule that sums exactly to the outer
envelope depth has ZERO slack for any wall thickness, and this is a hard mathematical
conflict, not a rounding issue.** `Unit 01 Zone Layout.FCStd`'s original (buggy) zone
depths for the H/F/S column summed to exactly 12000mm — the full outer building depth
— which only worked because the walls weren't actually being subtracted from that
column's usable space (the bug). Once wall thickness (150mm front + 150mm back =
300mm) is correctly subtracted, interior depth is 11700mm, and 12000mm of zone depth
physically cannot be stacked in series inside 11700mm no matter how gaps are
distributed (gaps can only ever add length, never remove it — proved by writing the
depth-budget as `g1+d1+g2+d2+g3+d3+g4 = interior_depth` with fixed `d1+d2+d3 >
interior_depth` and every `g` constrained `>= 0`: no non-negative solution exists).
**The only real fix, once zone sizes are fixed and non-negotiable, is to take the
conflicting zone OUT of the same shared-width column** (give it an X-range that does
not overlap the other zones' X-range) so it no longer needs to share linear depth
budget with them at all — not to hunt for a clever gap placement, which cannot work.
On `Unit 01 Zone Layout.FCStd` this meant relocating the bulk-storage zone (S) from
directly under the F/H column to the opposite (left) side of the building, alongside
W/C, which required real repositioning of documented "front-right corner" zone intent
— flagged explicitly to Erik as a phase-1 side effect of fixing the overlap bug
correctly, not something to fix quietly. **If a zone schedule's depths sum exactly to
an outer envelope dimension, treat that as a red flag before building anything** — it
means the original numbers were never checked against a real wall thickness.

## Phase 1 corrected, 2026-09-05 second pass — two-tier wall thickness, S restored to front-right

**The phase-1 writeup immediately above (single 150mm wall thickness, zero interior
partition budget, S relocated from front-right to the left side to make the depth
sum work) was Erik's call to override, and it was wrong.** It fit the exact
spreadsheet zone sizes inside the OLD nominal 7000x12000mm envelope by treating wall
thickness as effectively zero-budgeted against that fixed envelope — that's what
forced S off its documented front-right-near-the-door position. **The zone sizes are
interior/clear dimensions, not shrunk-to-fit numbers; the building's outer footprint
is supposed to grow to accommodate real wall thickness, not the other way round.**

**Corrected model, researched against SANS 10400-K:**
- **Exterior perimeter wall: 230mm** (cavity brick/blockwork, standard SA
  commercial-industrial practice) — around the whole building outline.
- **Interior partition: 76mm** (standard SA drywall partition) — applied **only
  around the bathroom (C)**, not between every zone pair. Reasoning: rev J's own
  zone descriptions treat H/F/S/W as open-plan zoned floor areas with deliberate
  zero-gap adjacency ("F...pulled back to sit directly in front of Zone H so the
  U-bench feeds straight into it", S "loading off the door") — only the bathroom is
  a real enclosed room needing privacy/plumbing walls. C sits in the back-left
  corner already flush against 2 exterior walls, so it only needs its own partition
  built on its 2 interior-facing sides (right + front) — 2 separate rectangular
  wall-strip loops in a new `Sketch_C_Partition` sketch, same rectangle-building
  pattern as the zone sketches, fully constrained with named `DistanceX`/`DistanceY`
  constraints, `solve()==0`.
- **Layout (rev J relative positions restored):** right column, flush to the right
  wall, stacked front-to-back with zero gap: S (front, near door) - F - H (back).
  Left column, flush to the left wall: W (front/door), open floor, C (back, enclosed
  by its own 76mm partition). **S is back in its documented front-right corner near
  the door** — with wall thickness properly accounted, this created no conflict: S's
  right-column stack (S+F+H depths 3500+4500+4000=12000mm, zero-gap) fits exactly
  within the interior clear depth because S no longer shares depth budget with the
  left column at all (different X range) — the original "S had to move" bug is fully
  resolved without a second workaround.
- **Computed outer building footprint: 6536mm (W) x 12460mm (D).** Interior clear
  width 6076mm (left column 2000mm + C's 76mm partition + right column 4000mm) +
  2x230mm exterior = 6536mm. Interior clear depth 12000mm (right column S+F+H
  stacked, zero-gap) + 2x230mm exterior = 12460mm — matches the simple
  12000+2x230=12460 back-of-envelope check exactly, confirming no hidden internal
  partition was needed on the S-F-H stack.
- Exact zone coordinates used (global sketch frame, origin at the building's outer
  corner): W (230,230)-(2230,2230); C (230,10230)-(2230,12230), partition strips at
  X2230-2306,Y10230-12230 and X230-2306,Y10154-10230; S (3806,230)-(6306,3730); F
  (2306,3730)-(6306,8230); H (2306,8230)-(6306,12230). Wall frame outer 0,0 to
  6536,12460, inner (interior face) 230,230 to 6306,12230.
- Verified: `solve()==0` and `FullyConstrained` on all 7 sketches
  (`Sketch_WallFrame`, `Sketch_C_Partition`, `Sketch_H`, `Sketch_F`, `Sketch_S`,
  `Sketch_W`, `Sketch_C`); every zone bbox checked programmatically against the
  computed plan (`Shape.BoundBox`, exact match, zero overlaps, correct flush
  zero-gap boundaries at both the 230mm exterior and 76mm C-partition faces);
  confirmed visually too — cropped/zoomed rasterized-PDF regions read directly
  showed C top-left with its visible partition corner, H top-right, F middle-right,
  W bottom-left, S bottom-right, matching rev J exactly. GUI round-trip: changed
  `H_Width` 4000->4200mm via `setDatum`, recomputed, confirmed the shape's bbox
  moved correctly (XMax 6306->6506), reverted to 4000, confirmed it returned to the
  original bbox exactly (XMax back to 6306). Full document recompute clean (45
  objects, `mustExecute()` false). Legend, notes and title block untouched (they
  don't depend on floor-plan geometry). **Sheet state unchanged from the first
  phase-1 pass: bare black-outline geometry only** — hatching/labels/dimensions/door
  symbol still need reattaching to this corrected geometry (phases 2-3), a fresh
  "final" PDF export is still deliberately deferred until then.

**Lesson for next time a zone-schedule sums exactly to an old nominal envelope
number**: don't assume the envelope is fixed and the zones must compress to fit —
check whether the zone sizes are meant to be clear/interior dimensions first. If
they are, the correct fix is growing the outer envelope by the real wall thickness,
not silently reflowing zone positions to make a fixed envelope work. Silently moving
a zone to fix a dimensional conflict is exactly the kind of thing to flag explicitly
rather than solve quietly — this is the second time in this same file that lesson
has been reinforced (see the very next section below for the flat-wire/solid
lesson learned the same way).

## Floor plan geometry: solids only, not flat faces/wires — SUPERSEDED, kept for record

**This was the original working method (2026-09-05, first build of this drawing) and
is preserved below for the history of why it was tried and what its real limits were —
see the superseding section immediately above for the current standing method.** A
flat Draft rectangle or Draft wire (a bare face or edge loop at Z=0) does not reliably
project through TechDraw's HLR (hidden-line-removal) pipeline when viewed top-down —
it can render correctly the first time and then silently vanish after an unrelated
change (Source list edit, Direction change, recompute), with no error. The fix used at
the time was to always give the floor plan real 3D solids to project, even for a "2D"
drawing:

```python
face = Part.Face(outline.Shape.Wires[0])   # outline = a Draft rectangle/wire, closed
solid = face.extrude(App.Vector(0, 0, 10))  # any nonzero thickness, e.g. 10mm
slab = doc.addObject("Part::Feature", "Zone_H_slab")
slab.Placement = App.Placement()            # IDENTITY - see next gotcha
slab.Shape = solid
slab.ViewObject.ShapeColor = (0.66, 0.22, 0.13)   # classification colour, not brand colour
```

**Gotcha — double-applied placement.** If `outline` already has a non-identity
`Placement` (e.g. it was moved with `.Placement.Base = ...`), and you extrude its
`.Shape` (which is already in **global** coordinates), then also leave the new
`Part::Feature`'s own `Placement` unset/copied from the outline, the offset gets
applied twice — the solid renders at roughly double its intended offset from origin.
Symptom: the wall/zone-block projected positions don't match each other at all even
though each one's *shape* looks individually fine. Fix: always set the new solid
object's `Placement` to `App.Placement()` (identity) since the shape is already
correctly positioned in global coordinates by the extrude.

**Outer walls / partitions as hollow-frame solids** (this is what finally worked,
after `Arch::Wall` produced walls that intermittently vanished the same way flat
wires did — `Arch::Wall` on a `Draft.make_wire` base is not obviously more reliable
than a plain solid here):

```python
outer = Part.makePlane(7000, 12000, App.Vector(0, 0, 0))
inner = Part.makePlane(6700, 11700, App.Vector(150, 150, 0))  # 150mm wall thickness
frame_solid = outer.cut(inner).extrude(App.Vector(0, 0, 10))
wall = doc.addObject("Part::Feature", "WallFrame")
wall.Shape = frame_solid
wall.ViewObject.ShapeColor = (0.0, 0.0, 0.0)
```

Sanity-check every boolean result before trusting it in the view — this is not
optional, a bad boolean is the most common silent-failure cause:

```python
assert wall.Shape.isValid()
assert len(wall.Shape.Solids) == 1
```

**`Arch::Space` was tried and abandoned.** `Arch.makeSpace(baseobj=rect)` created an
object whose `.Shape` raised `RuntimeError: shape is invalid` when accessed, with no
further diagnosis obviously available. Zone colouring and area labelling is handled
directly on the Draft-rectangle-turned-solid instead (see above) — simpler and it
actually works. If a future job needs FreeCAD's automatic Area computation specifically,
that's the open problem to solve, not colour-only zoning.

## Dimension-cluster overlap - fix by respacing X/Y nudges, not by re-deriving values

Hit 2026-09-05: the six zone-width dimensions (`Dim_Overall_Width`, `Dim_H_Width`,
`Dim_F_Width`, `Dim_S_Width`, `Dim_W_Width`, `Dim_C_Width`) all reference collinear/
nearby edges and had all been given `X=0`, with `Y` stepped only 10mm apart
(-15, -25, -35, -45, -55, -65) — far too tight for ~3.5mm text plus arrowheads, causing
exactly the reported collisions ("4000"/"4000" reading as "4000+4500", "2000"/"2000"
stacked unreadably). Separately, `Dim_H_Depth` (X=-15,Y=20) and `Dim_F_Depth` (X=8,Y=-30)
sat close enough that their text visually crossed ("4000"/"4500" overlapping digits).

**Fix confirmed working**: widen the `Y` step between stacked dimensions of the same
type to ~20mm minimum (used -12, -28, -48, -68, -88, -108 for the six width dims) and
give crowded pairs distinctly different `X` as well as `Y` so they separate in two
directions, not one (`Dim_H_Depth` moved to X=-40,Y=45; `Dim_F_Depth` to X=32,Y=-45).
**Every nudge stayed in the same "modest, not wild" range** this doc already documents
below (single digits to ~40mm) — no dimension needed re-referencing or its
`References2D` touched, `getRawValue()` on all twelve dimensions was re-checked after
the moves and every value was unchanged (values are driven purely by `References2D`,
positions don't affect them). Verified by rasterizing after each change, not just
reading the numbers — this is a purely visual/layout fix, and the only way to confirm
"no more overlap" is to look at the rendered PNG.

**Second round, same day, later pass**: after the hatching pass, `Dim_F_Depth`
(X=32,Y=-45) and `Dim_S_Depth` (X=22,Y=-60) had drifted close enough again for their
rotated "4500"/"3500" text to visually cross. **A big single jump in one axis
overcorrects** — moving `Dim_S_Depth` to X=5,Y=-95 in one step cleared the F/S
collision but walked the text down into a *different* cluster (the far-left
`Dim_C_Depth`/`Dim_W_Depth` "2000"/"2000" stack), because `X`/`Y` offsets are relative
to each dimension's own auto-derived anchor, not a shared page grid — nudges on
different dimensions aren't comparable in raw magnitude. **Fix that stuck**:
`Dim_F_Depth` to X=65,Y=-30 and `Dim_S_Depth` to X=50,Y=-22 — both pushed the same
direction (further right, less far down) by a similarly modest amount, keeping each
clear of both its neighbor and the unrelated cluster further down the page. Lesson:
after any nudge, rasterize and check the *whole* dimension-cluster region, not just
the two objects you touched — a fix can solve the reported collision while creating a
new one just outside the crop you were looking at.

## Zone letter labels: geometry, not page-coordinate annotations

Do **not** try to hand-compute the page-mm position of a `TechDraw::DrawViewAnnotation`
from the 3D real-world coordinates and the view's `X`/`Y`/`Scale`/`Direction` — the
transform (in particular whether real-world Y maps to +page-Y or −page-Y, and where
the view's `X`/`Y` anchor point actually sits relative to the projected bounding box)
was not reliably derived by hand in this session across several attempts, and the
result was labels sitting nowhere near their intended zones.

Instead, model the label as real 3D text geometry and let it project through the same
pipeline as everything else — it then can't be wrong relative to the zones, because
it's positioned in the same coordinate space as them:

```python
ss = Draft.make_shapestring(String="H", FontFile="/usr/share/fonts/TTF/DejaVuSansMono-Bold.ttf",
                             Size=600, Tracking=0)
ss.Placement = App.Placement(App.Vector(zone_center_x - 300, zone_center_y - 300, 15), App.Rotation())
ss.ViewObject.ShapeColor = (0.0, 0.0, 0.0)
# add ss to view.Source alongside the zone slabs
```

`Size` is in real-world mm (before the view's Scale is applied), so 600mm tall text at
1:100 scale prints at 6mm — legible on an A3 sheet.

## Dimensions: SOLVED, 2026-09-05 — use `getEdgeBySelection`/`getVertexBySelection`, never `getEdgeByIndex`

**The earlier "fragile, not yet solved" writeup below (kept struck through in spirit,
not deleted, because the wrong-tool mistake is worth recording) was diagnosing the
wrong API.** `TechDraw::DrawViewPart` exposes TWO unrelated numbering schemes and the
first session used the wrong one:

- `getEdgeByIndex(i)` / `getVisibleEdges()` — a 0-based, **HLR-cache-order** numbering,
  apparently meant for cosmetic-geometry helpers (`makeCosmeticLine` etc). It does
  **not** match what `References2D`'s `"EdgeN"` strings resolve to, and worse, it is
  unstable: querying it, then feeding `f"Edge{i+1}"` into a freshly-created
  `DrawViewDimension` gave wrong values (0, or another edge's length) even when done
  immediately with no intervening recompute.
- `getEdgeBySelection("EdgeN")` / `getVertexBySelection("VertexN")` — **this is the
  real one.** It resolves the exact same "EdgeN"/"VertexN" SubElement string that
  `References2D` consumes and that GUI click-selection would produce, returning the
  actual `Part.Edge`/`Part.Vertex` in the view's local (unscaled, center-of-bbox-origin)
  coordinate frame. Confirmed by round-trip: for every edge/vertex needed on this
  drawing, using `getEdgeBySelection`/`getVertexBySelection` to pick the index, then
  building a `TechDraw::DrawViewDimension` with that exact reference and checking
  `dim.getRawValue()`, gave the **exact expected length every time** (12 for 12,
  verified against all 5 zone Size(m) rows and the overall 7000x12000 footprint on
  `Unit 01 Zone Layout.FCStd`). `fp.getSubObject("EdgeN")` (the generic FreeCAD
  subelement resolver) returns `None` for TechDraw views — don't reach for it either.

**Worked method:**

```python
cx, cy = 3500.0, 6000.0   # half of the Source geometry's real-world bounding box (X,Y)
                          # — TechDraw centers the view's local origin on that bbox center
edge_map = {}
for n in range(1, 250):                       # scan generously past the real count
    try:
        e = fp.getEdgeBySelection(f"Edge{n}")
    except Exception:
        continue
    if e is None or len(e.Vertexes) != 2:
        continue
    p0 = (round(e.Vertexes[0].X + cx, 1), round(e.Vertexes[0].Y + cy, 1))  # back to real-world mm
    p1 = (round(e.Vertexes[1].X + cx, 1), round(e.Vertexes[1].Y + cy, 1))
    edge_map[n] = (p0, p1)

# now find the EdgeN whose real-world endpoints match the boundary you want to dimension
target = [(p0, p1) for n, (p0, p1) in edge_map.items()
          if {p0, p1} == {(3000.0, 8000.0), (7000.0, 8000.0)}]   # e.g. zone H's width edge
```

Same pattern for vertices with `getVertexBySelection`. Once you have the confirmed
`n`, build the dimension and **verify numerically before trusting it visually**:

```python
d = doc.addObject("TechDraw::DrawViewDimension", "Dim_H_Width")
d.Type = "DistanceX"                                   # or DistanceY for vertical
d.References2D = [(fp, f"Edge{n}")]                     # or two Vertex refs for a span
                                                         # that isn't a single clean edge
page.addView(d)
doc.recompute()
assert abs(float(d.getRawValue()) - 4000) < 0.5         # H is 4.0 x 4.0 m per the schedule
```

**Not every boundary is a single selectable edge.** Where a zone's boundary lies
exactly along a line another object also occupies (e.g. a zone's edge coincident with
the outer `WallFrame` boundary, or a short zone edge that's a sub-span of a longer
neighbouring zone's edge on the same line), TechDraw's HLR merges/drops the
would-be-shorter edge and it simply isn't independently selectable — searching for it
by coordinate correctly returns no match, this isn't a search-tolerance bug. Don't
force it; use a **two-vertex `DistanceX`/`DistanceY` dimension** instead (the corner
vertices of that boundary are still independently selectable via
`getVertexBySelection` even when the connecting edge between them isn't). This is
exactly as valid a `TechDraw::DrawViewDimension` reference as a single edge.

**`X`/`Y` on `DrawViewDimension` are absolute page-mm coordinates, not a
relative/perpendicular offset, and moving them does not affect the measured value**
(`getRawValue()` is driven purely by `References2D`, independent of placement).
Confirmed: `X=0,Y=0` triggers an internal "auto" default that visually clusters
multiple dimensions referencing nearby/collinear geometry on top of each other with
overlapping text (this is the real cause of what looked like "dimension chaos" before
placement was touched — it is a layout collision, not a wrong-value bug). Moving
`X`/`Y` by a **modest amount (single-digit to ~20mm)** cleanly separates the line and
its text and is safe. Setting `X`/`Y` to something wildly different from the auto
value (tested with `999,999`) does **not** cleanly translate the dimension — it draws
a long stray line toward that point, visibly wrong. Nudge, don't teleport, and
re-render after every change; there's no other way to know where a given `X`/`Y` will
land.

**One further real (not cosmetic-only) surprise: large values silently switch display
unit.** `Dim_Overall_Depth` (12000mm, the only dimension on this drawing ≥10 m)
rendered as `12` with `ShowUnits=False`, not `12000` — every other dimension (all
<10 m) rendered in full mm. This is not a bug or truncation: `ShowUnits=True` on the
same object rendered `12 m`, confirming TechDraw auto-converts to metres above some
internal length threshold regardless of `FormatSpec`/`ShowUnits` (`ShowUnits` only
toggles the suffix, not the underlying unit choice). Changing `FormatSpec`
(`"%.2w"` → `"%.0f"`) had **no effect** on this, including after fully deleting and
recreating the object — the conversion is happening upstream of `FormatSpec`. If a
drawing has any dimension ≥10 m, check its rendered text specifically (don't assume
mm-in, mm-out) and either accept the metre display (set `ShowUnits=True` so it reads
"12 m" rather than a bare, confusing "12") or keep every such value under 10 m by
construction.

**Superseded findings from the first attempt at this, kept for the record of what
NOT to retry:**
- Do not use `getEdgeByIndex`/`getVisibleEdges` output as `References2D` indices —
  wrong numbering scheme entirely (see above), not merely off-by-one.
- Do not assume `fp.getSubObject("EdgeN")` will resolve a TechDraw view's edges —
  returns `None`; that's a generic Part-shape subelement API and TechDraw views
  don't expose their geometry through it.
- The original "guess indices by trial-and-export" method genuinely never landed on
  a reliably-correct index across several iterations, which is what prompted this
  proper investigation — that failure mode was real, the fix was using the right
  accessor, not more trial and error.

## Zone fills: hatching - SOLVED, 2026-09-05 - `DrawHatch` does NOT work reliably; use 3D line geometry instead

**`TechDraw::DrawHatch` was tested directly against plain (non-section)
`TechDraw::DrawViewPart` zone faces and rejected as a mechanism — not because it errors,
but because its `Face1`/`Face2`/... addressing is not deterministic or controllable
enough to guarantee a specific hatch lands on a specific zone.** Full findings, so nobody
re-spends this time:

- `DrawHatch.Source` is a `PropertyLinkSub` — set it as a **tuple**, `h.Source = (view,
  ["FaceN"])`, not a list of tuples (`[(view, "FaceN")]` raises `ValueError: Expect input
  sequence of size 2`). `TechDraw::DrawViewPart` has no `getFaceBySelection` helper (only
  `getEdgeBySelection`/`getVertexBySelection` exist) — face indices have to be found by
  trial and error, checking the rendered result, exactly the "guess indices" failure mode
  the edge/vertex work above already solved and moved away from for dimensions.
- Bound to the real `FloorPlan` view (Source = many touching, coplanar zone solids +
  wall frame), `Face1` **did** produce a visible hatch fill in the exported PDF — but it
  filled the ENTIRE merged interior silhouette (every zone: C, WAX, H, F, S, W all at
  once) with one pattern, not a single zone. Confirmed by cropping a clean interior
  pixel region and comparing against a hatch-hidden baseline (0 → 540 non-white pixels).
  Touching, coplanar top faces get merged into one addressable "face" for hatch
  purposes — there is no per-zone index available this way.
- Bound to an **isolated single-object view** (Source = one zone slab alone, otherwise
  built identically to the working zone-label pattern below, same X/Y/Scale/Direction as
  FloorPlan so it overlays exactly) — `Face1` through `Face6` (a box has 6 faces) all
  produced **zero** hatch, every time, confirmed by exporting all 6 and diffing every
  render against a hatch-hidden baseline pixel-for-pixel. A lone, non-merged solid
  appears not to get an addressable "SectionFace" at all in a plain top view.
- Bound to a **two-object view** (one real zone box + a small non-touching "junk" box
  added purely to test whether separation unlocks addressing) — `Face1` hatched the
  JUNK box, not the zone box that was first in the `Source` list, and none of `Face2`
  through `Face12` hatched anything (confirmed the same diff-against-baseline way).
  Indexing is neither list-order nor a clean per-object face count; it is whatever
  internal artifact the HLR/section-face code happens to produce, which for an ordinary
  (non-section) `DrawViewPart` is not something this API is meant to expose.

**Conclusion: `DrawHatch` on `TechDraw::DrawViewPart` is built for actual
`DrawViewSection` cut faces and is not a controllable mechanism for hatching individual
zones of a plain top-view floor plan. Do not reach for it here again — go straight to
the alternative below.**

**What actually worked: hatch lines built as real 3D solid geometry, projected through
the same `FloorPlan.Source` pipeline as everything else** — exactly the same idiom this
doc already uses for zone-letter labels ("geometry, not page-coordinate annotations"),
extended to hatch lines. For each zone, generate a set of parallel line segments at the
desired angle/spacing, clipped to the zone's real-world rectangle (Liang-Barsky
clipping against the axis-aligned bbox), and build each as a thin extruded rod solid
(`Part.makeBox` rotated to the line's angle, ~25mm wide, ~2mm tall, sitting at
Z=10 to Z=12, directly on top of the zone slab) — solids, per the doc's own strongest
rule, not flat wires, for guaranteed HLR reliability:

```python
def clip_segment(angle_deg, offset, bbox, big=20000.0):
    # Liang-Barsky: infinite line through (rect-center + offset*normal), clipped to bbox
    ...  # see Unit 01 Zone Layout.FCStd build history for the full ~15-line function

def rod_solid(p1, p2, width=25.0, zbase=10.0, height=2.0):
    length = math.hypot(p2[0]-p1[0], p2[1]-p1[1])
    ang = math.degrees(math.atan2(p2[1]-p1[1], p2[0]-p1[0]))
    box = Part.makeBox(length, width, height, App.Vector(0, -width/2.0, 0))
    box.Placement = App.Placement(App.Vector(0,0,0), App.Rotation(App.Vector(0,0,1), ang))
    box.translate(App.Vector(p1[0], p1[1], zbase))
    return box
```

Generate a family of `(angle, spacing)` lines per zone bbox, one `Part::Feature` per
segment, `ShapeColor = (0,0,0)`, add them all to `FloorPlan.Source` alongside the zone
slabs, recompute, `page.ViewObject.doubleClicked()`, re-export. This is fully
deterministic — each hatch line is built directly from the same real-world zone bbox
the zone slab itself uses, so there is no face-index guessing and no risk of upstream
renumbering. Verified on `Unit 01 Zone Layout.FCStd`: 5 visually distinct patterns per
zone type — H = 45° diagonal (600mm spacing), F = 135° diagonal/opposite (600mm), S
(bulk storage + wax store, same zone type, same pattern) = horizontal (500mm), W =
vertical (500mm), C = cross-hatch grid (900mm, both directions) — confirmed by
rasterizing and reading the PNG, each pattern confined correctly to its own zone with no
bleed into neighbours.

**Legend built the same way, at legend scale.** The existing `LegendView` (a second,
independent `TechDraw::DrawViewPart`, `Scale=1.0`) already has one small colour-swatch
box per zone (`LegendSwatch2_<zone>`, 6x6mm boxes at local coords). Add matching
hatch-line rods on top of each swatch using the identical `clip_segment`/`rod_solid`
functions, just with legend-scale numbers (2-3mm spacing, ~0.35mm line width, sitting at
Z 2.0-2.5mm above the swatch's Z 0-2mm), one small rect bbox per swatch row, add them all
to `LegendView.Source` (not `FloorPlan.Source`), remember to re-add via `page.addView()`
gotcha does NOT re-trigger here since the view is already in `page.Views` — just append
to `.Source` and recompute. Result: the legend now shows colour swatch AND the exact
matching hatch pattern per zone, single source of truth for both, verified by
rasterizing and reading the PNG at full resolution.

**Cost note for next time**: finding this out took ~15 rendered PDF exports (each
3-9 seconds) plus pixel-diffing against baselines to get a trustworthy negative result
on `DrawHatch` — budget for that if re-verifying, but don't re-run the `DrawHatch`
experiments themselves; the conclusion above is final for a plain `DrawViewPart`.

## Hatch density increase, 2026-09-05 — spacing halved, plus a real fill-merge bug in cross-hatch

Erik's review: hatch reads as "basically still black background" at normal viewing
distance because TechDraw doesn't fill zone faces on a plain top view (see the
`DrawHatch` writeup above) — the only colour on the page is the hatch-line rods
themselves, and the original spacing (H/F 600mm, S/W 500mm, C 900mm-with-only-2-lines)
left too much white between them. **Decision: increase density, not add a translucent
fill layer** — denser rods cover more of the zone's printed area, which reads as
stronger colour without changing the "geometry, not a fill" approach documented above.

**New values (real-world mm, at 1:100)**, same `Liang-Barsky clip + rotated rod solid`
method as before, spacing roughly halved, rod width bumped modestly:
- H: 600→300mm spacing, 25→32mm rod width, 45°
- F: 600→300mm spacing, 25→32mm rod width, 135°
- S (bulk store + wax store): 500→250mm spacing, 25→30mm width, horizontal
- W: 500→250mm spacing, 25→30mm width, vertical
- C: 900→400mm spacing (both directions), 25→30mm width, orthogonal cross-hatch —
  this zone also needed a real multi-line grid built (the old version had only 2 lines
  total, nowhere near a "grid"), see the bug below for why a naive full grid doesn't
  render safely.
Every clip bbox also got an **inward inset** (80–120mm) before generating lines, so rod
endpoints stop short of the zone's true boundary edge rather than touching it exactly.

**New bug found and fixed — a genuine "closed grid" fill-merge, not the already-known
`DrawHatch` merge.** Building C's cross-hatch as full-length horizontal + full-length
vertical rods (even on separate Z-slabs so they don't volumetrically overlap in 3D)
made FreeCAD's HLR/SVG export intermittently render the **entire zone as one solid
filled block** instead of thin grid lines — confirmed by rasterizing: sometimes correct
thin lines, sometimes 60%+ pixel fill, same geometry, same export call, no property
changed in between. Root cause: when enough parallel members in both directions span
edge-to-edge, the outermost rods together trace a **closed rectangular loop** around
the zone (this is inherent to a real N×M grid, not proximity to the true wall — a
denser `inset` alone does not fix it), and TechDraw's SVG generator fills the enclosed
region as a single face rather than treating each rod as its own face. Z-separating the
two directions did **not** fix it (their 2D projected silhouettes still touch/overlap in
XY, which is what the fill code keys off, not 3D collision).

**Real fix: break every horizontal rod into short segments with a small XY gap
(rod-width/2 + ~4mm) wherever it would cross a vertical rod**, so no two rods from
different directions ever touch or overlap in projection — no possible closed loop, no
merge, regardless of density. Vertical rods stay full-length (only one direction needs
splitting to break every crossing). This is a real, deterministic fix — verified
correct across a repeated export.

**Separate, real gotcha hit on the same object: intermittent stale TechDraw export,
independent of the fill-merge bug above.** After rewiring `HatchColor_C.Source` to the
new (already-correct, gap-fixed) rod objects, repeated `doc.recompute(None, True,
True)` + `page.ViewObject.doubleClicked()` + `FreeCADGui.updateGui()` +
`TechDrawGui.exportPageAsPdf()` cycles produced **wildly inconsistent output for that
one view across otherwise-identical export calls** — sometimes the correct thin grid,
sometimes fully blank (0% coverage, not even the black outline), with no property
touched in between and every object reporting `Up-to-date`. Other views on the same
page (`HatchColor_F`, `LegendHatchColor_*`) intermittently blanked out on other export
passes the same way, then reappeared correctly on the next export with zero changes —
this reads as a real caching/redraw race in the QGraphicsScene behind the exported SVG,
not a geometry or property bug (every underlying value was confirmed correct via
`execute_python` reads throughout). **Fix that reliably cleared it for the object that
kept sticking**: don't just edit `.Source` and re-recompute — **delete and recreate the
`TechDraw::DrawViewPart` object itself** (read off its old `Source`/`X`/`Y`/`Scale`/
`FaceColor`/`FaceTransparency`/`LineWidth` first, `page.removeView()` +
`doc.removeObject()`, then `doc.addObject()` fresh, reapply every property, `X`/`Y`
**after** `page.addView()` per the existing gotcha above). A fresh object forces a new
graphics item rather than reusing a possibly-stale cached one. **Always rasterize and
re-check after every export in this kind of multi-view session — a clean recompute and
an `Up-to-date` state are not proof the exported file is current**, this cost several
throwaway export/rasterize cycles here before landing on the fix.

**Verification method for "is the hatch actually denser" — pixel coverage, not just
looking at it.** Rasterize the PDF at 300dpi (`pdftoppm -png -r 300`), then for each
zone: compute its real-world bbox → page-pixel bbox using the view's `X`/`Y` (page-mm,
anchored at the view's own bbox centre) and `Scale`, at `300/25.4 * Scale` px per
real-world-mm; crop that pixel box out of the full-resolution PNG; count pixels within a
small RGB tolerance (±40) of that zone's exact `ShapeColor`/`FaceColor` (converted
0–1→0–255) against the crop's total pixel count. Because a plain top-view
`DrawViewPart` renders zone slabs with no fill (`FaceTransparency` defaults to 100 on
the base `FloorPlan` view), the only pixels that can match a zone's exact colour are its
own hatch rods — no need to mask out the outline/dimension/label geometry separately.
Measured on `Unit 01 Zone Layout.FCStd`, before → after this pass: H 2.45%→7.2%, F
2.48%→7.24%, S (bulk+wax combined) 3.01%→7.36%, W 2.24%→6.99%, C 1.47%→9.03% (C's
increase is largest because it went from a near-empty 2-line "grid" to a real 5×5-line
one). All five patterns re-confirmed visually distinct at this density: H/F opposite
diagonals, S horizontal, W vertical, C orthogonal cross — no pattern reads as a
converged grey mush, and the zone letters (H/F/S/W/C) remain legible through the
hatching in every zone. Verified one page, `pdftotext`/`pdfinfo` clean, re-exported PDF,
re-saved FCStd.

## Worked order of operations

1. Create `Parameters` spreadsheet, fill identity + schedule range.
2. Build floor-plan solids: outer `WallFrame`, `PartitionFrame`, one solid per zone,
   text-geometry labels — all as `Part::Feature` objects with identity `Placement`.
3. Create `Template` (SVG template) → `Page` (uses Template).
4. Create `FloorPlan` = `TechDraw::DrawViewPart`, `.Source` = the list of all the
   solids above, `.ScaleType = 'Custom'`, `.Scale` = drawing scale (e.g. `0.01` for
   1:100), `.Direction = App.Vector(0,0,1)`, `.XDirection = App.Vector(1,0,0)`, `.X`/
   `.Y` = page position (mm). `page.addView(FloorPlan)`.
5. Create `ZoneSchedule` = `TechDraw::DrawViewSpreadsheet` bound to the schedule
   range, `page.addView(ZoneSchedule)`.
6. Push spreadsheet identity values into `Template.EditableTexts` (see gotcha above —
   this is a one-time push, not a live binding; re-run after editing the sheet).
7. `doc.recompute()`.
8. **First-render gotcha**: call `page.ViewObject.doubleClicked()` once before the
   first export of a session — this forces FreeCAD to actually initialize the page's
   QGraphicsScene. Skipping it produces an export with the floor-plan geometry and
   title block silently missing, even though every object's `PageResult`/`Shape`
   looks correct. Re-call it (and recompute again) after *any* structural change to
   `view.Source` or a newly-added view like the schedule table — it is not a one-time
   setup step, it is "make sure the page has actually redrawn" and cheap to call
   defensively before every export.
9. `TechDrawGui.exportPageAsPdf(page, out_path)`.
10. Verify: `pdftoppm -png -r 150 out.pdf page` then look at the PNG with Read — do
    not trust "the script ran clean." `mcp__freecad__get_screenshot` is broken in
    this environment (`AttributeError: 'dict' object has no attribute '__name__'`) —
    export-to-PDF-then-rasterize is the standing substitute.
11. `pdftotext out.pdf -` — confirm title-block and schedule values appear as real
    text (they will, since this whole pipeline is SVG/vector-based, not a raster
    print) and `pdfinfo | grep Pages` confirms one page.

## Fixing a mirrored/flipped floor plan — verify against the reference, don't guess

Hit 2026-09-05: Erik reported the exported plan was "mirrored left-right" vs the approved
`Unit 01 Zone Layout Drawing.html` (rev J) reference. **The reported "left-right" framing
was wrong once actually measured — it was a front-back (Y-axis) flip, not a left-right
(X-axis) one.** Don't trust a verbal description of which axis is wrong; derive it from
coordinates on both sides.

**Method that actually settles it** — compare real numbers, not impressions:
1. Read each zone's real-world bounding box off the FreeCAD objects
   (`obj.Shape.BoundBox`) contributing to the `TechDraw::DrawViewPart.Source`.
2. Get the reference's real numbers too. If the reference is HTML/SVG (as here), extract
   the inner `<svg class="plan" ...>...</svg>` block with `awk` and rasterize it directly
   with `rsvg-convert -w 800 plan.svg -o ref.png` (works even with unresolved CSS custom
   properties — fills go black but stroke/layout geometry, which is what you need, still
   renders correctly). Read the PNG and note which zones sit top/bottom/left/right.
3. Compare zone-by-zone which real-world axis (X or Y) needs which sign flip to match.
   Here: X-grouping (which zones share the small-X vs large-X side) already matched the
   reference; only the Y assignment (which zones sat at high vs low Y) was backwards.

**Tried and abandoned — do not retry these:**
- `TechDraw::DrawViewPart.XDirection` flipped to `(-1,0,0)`: **had zero effect** — the
  exported PDF was byte-for-byte pixel-identical (`ImageChops.difference(...).getbbox()`
  returned `None`). FreeCAD 1.1.3 appears to silently normalize/ignore this when it's
  collinear with the existing X axis in this configuration. Don't spend more time on
  `XDirection` as a fix without first proving it changes the render at all.
- `TechDraw::DrawViewPart.Direction` flipped to `(0,0,-1)`: **did** change the render, but
  it performs a true reflection of the whole projected 3D scene, which mirrors the
  ShapeString text geometry too — letters came out backwards/unreadable ("Ɔ" for "C",
  etc). Any fix that changes the *view's* projection/orientation properties mirrors
  everything it projects, text included. Never use it to fix an orientation problem where
  text labels are built as real 3D geometry (per this doc's own labeling convention).

**What actually worked — translate the geometry, not the view.** Since the floor-plan
solids all carry identity `Placement` (per this doc's own convention) with their real
position baked into the `Shape`, fix a front/back (or left/right) mix-up by translating
each contributing object along the wrong axis to its correct slot — not by mirroring/
reflecting anything:

```python
for name in floor_plan_source_object_names:
    o = doc.getObject(name)
    bb = o.Shape.BoundBox
    delta = 12000 - bb.YMin - bb.YMax   # 12000 = full building depth on the flipped axis
    base = o.Placement.Base
    o.Placement = App.Placement(App.Vector(base.x, base.y + delta, base.z), o.Placement.Rotation)
```

This is a **translation** (rigid-body move to the mirrored slot), not a reflection — it
swaps which zones sit at which end of the axis while leaving every object's internal
orientation untouched, so text labels stay upright and readable. A full-height object
(e.g. the outer `WallFrame` spanning the entire 0-12000 range) computes `delta = 0` and
correctly doesn't move. Verify with `page.ViewObject.doubleClicked()` + recompute +
export + rasterize + compare pixel bboxes against the reference render before trusting it.

## Notes / legend block — a permanent template feature, not a one-off

Added 2026-09-05, validated on `09 - Okracandle/Property/Unit 01 Zone Layout.FCStd`.
Erik's framing: this is a **standing feature of the generic template**
(`11 - Arkoatelier/FreeCAD Templates/`), reusable for any future client's drawing —
build it once, reuse it, don't rebuild bespoke per job. Two objects, both placed
in the sheet's open area (not the title block, not overlapping the floor plan or
the zone schedule):

**Gotcha discovered here — `DrawViewPart` AND `DrawViewAnnotation` both anchor
`.X`/`.Y` at the CENTER of their rendered bounding box, not a corner.** Confirmed
by placing a single calibration line of text at `X=50, Y=50` and measuring where
it actually rendered: the text's bounding-box center landed at (50, 50), not its
top-left corner. This matters because a long single line (or an oversized font)
makes the block balloon outward from that center point in both directions —
explains an early failed attempt where notes text overlapped the floor plan on
the *opposite* side of the page from where it was "placed": the true center was
on-target, but the block was 2-3x too wide/tall for the font size used, so its
edges reached across half the sheet. Fix any positioning surprise by shrinking
the content (font size / line length) before assuming the coordinate is wrong.

**Legend** (`LegendView`): a *second, independent* `TechDraw::DrawViewPart`,
separate from the floor-plan `FloorPlan` view — not added to its `Source` list.
`Scale = 1.0` (1 model-mm = 1 page-mm, unlike the floor plan's 0.01), so swatch
and font sizes can be specified directly in the page mm you want them to print
at. Built the same way as the floor-plan zone labels (geometry, not annotation),
for the same reason — guaranteed alignment between swatch and label, since both
are positioned in the same local coordinate frame before the view ever projects
them:

```python
box = Part.makeBox(6.0, 6.0, 2.0, FreeCAD.Vector(0, -i*9.0, 0))  # one row per zone, 9mm pitch
sw = doc.addObject("Part::Feature", f"LegendSwatch_{zone}")
sw.Placement = FreeCAD.Placement()          # identity — box coords are already final
sw.Shape = box
sw.ViewObject.ShapeColor = zone_color        # match the actual zone slab's ShapeColor exactly

ss = Draft.make_shapestring(String=f"{zone} -- {use}",
                             FontFile="/usr/share/fonts/TTF/DejaVuSansMono-Bold.ttf",
                             Size=3.5, Tracking=0)
ss.Placement = FreeCAD.Placement(FreeCAD.Vector(6.0+3.0, -i*9.0+1.0, 1.0), FreeCAD.Rotation())

lv = doc.addObject("TechDraw::DrawViewPart", "LegendView")
lv.Source = [sw, ss, ...]   # all swatches + labels
lv.ScaleType = "Custom"; lv.Scale = 1.0
lv.Direction = FreeCAD.Vector(0,0,1); lv.XDirection = FreeCAD.Vector(1,0,0)
page.addView(lv)
lv.X = 320; lv.Y = 255      # MUST be set again after addView() — see gotcha below
```

**Known limitation, accepted deliberately**: the legend's zone-name text is
geometry (outline curves), so it does **not** appear in `pdftotext` output —
same trade-off already accepted for the floor-plan's H/F/S/W/C zone letters, for
the same reason (position reliability beats text-searchability here). If a future
job needs a text-searchable legend, the alternative is a *separate*
`TechDraw::DrawViewAnnotation` next to (not combined with) the swatch view — not
attempted here because matching its automatic line-spacing to the swatch view's
manually-set row pitch, precisely enough to keep rows aligned, looked more
fragile than the geometry approach for the time available. Flag this if it
becomes a real requirement rather than solving it speculatively.

**Notes** (`NotesView`): a `TechDraw::DrawViewAnnotation` added straight to the
page (not through any 3D view/projection) — this is real, extractable PDF text,
confirmed with `pdftotext`. Content pulled from a `Notes` range in the
`Parameters` spreadsheet at build time (same "push, not bind" pattern as the
title block — editing the spreadsheet after the fact requires re-running the
push before re-export). **No auto-wrap** — each entry in `.Text` (a Python list)
renders as exactly one page line, so wrap long sentences yourself before
assigning:

```python
import textwrap
notes_lines = [str(sheet.get(f"A{26+i}")) for i in range(5)]   # one sentence per spreadsheet row
wrapped = ["NOTES", ""]
for line in notes_lines:
    wrapped.extend(textwrap.wrap(line, width=78, subsequent_indent="   "))
na = doc.addObject("TechDraw::DrawViewAnnotation", "NotesView")
na.Text = wrapped
na.TextSize = 3.2
page.addView(na)
na.X = 300; na.Y = 172      # again: set AFTER addView()
```

**Gotcha — `addView()` resets `.X`/`.Y` to an auto-placed default.** Both
`LegendView` and `NotesView` had their explicitly-set `.X`/`.Y` silently
overwritten the moment `page.addView(...)` ran. Always set position *after*
`addView()`, not before, and re-verify with `float(view.X)` — this cost a full
extra render/rasterize cycle here since the first attempt looked plausible in
code but rendered in the wrong place.

**Sizing that worked at A3 1:100** (5-row legend, ~5-sentence notes block):
swatch 6mm, legend label `Size=3.5`, `NotesView.TextSize=3.2`, wrap width 78
chars. Both blocks fit comfortably in the sheet's open area (right of the floor
plan, above the title block/schedule) without touching either. Scale these
together if a future drawing has more zones or longer notes.

## Layout rebalance after adding legend/notes — table shrank, notes ballooned

Hit 2026-09-05, same session as the legend/notes build above. After adding
`LegendView` and `NotesView` to the open area, the pre-existing `ZoneSchedule`
(`TechDraw::DrawViewSpreadsheet`) was left at its old `Scale=0.6`/position,
which put it in a cramped ~61x27mm box squeezed between the notes text and
the title block — not because anything actively resized it, but because the
notes block above it had grown far larger than planned (see below) and ate
the space the table used to have room to breathe in.

**`DrawViewSpreadsheet.Scale` scales column widths, row heights and text
together, uniformly** — increasing `Scale` alone (no column-width edits
needed) is enough to go from illegible to comfortable. Empirically on this
table (3 cols, 100-unit default width, 30-unit row height): `Scale=0.6` gave
~61mm wide x 27mm tall; `Scale=1.3` gave ~132mm wide x ~59mm tall. Kept the
table rather than dropping it — its `Size (m)` column has data the legend
doesn't carry — just enlarged it (`Scale` 0.6 -> 1.3) and repositioned into
its own clear slot below the notes and above the title block.

**`DrawViewAnnotation` with `ScaleType="Automatic"` produced badly inflated
line spacing** — 17 wrapped lines rendered at ~7mm/line pitch (a ~120mm-tall
block) even at `TextSize=3.2mm`, `LineSpace=100`, on a `Page.Scale=1.0`
document — pitch/TextSize ratio of ~2.2, way more than normal single
spacing. The `Scale` property read back `0.5` while `ScaleType` was
`Automatic`, i.e. a stale/inconsistent value was in play. Fix: set
`ScaleType = "Custom"` and `Scale = 1.0` explicitly, drop `LineSpace` from
its default to something tighter (e.g. `70`, an **int**, not a string —
`LineSpace = "70"` raises `TypeError: type must be int, not str`), and widen
the wrap width so fewer lines are needed (78 -> 98 chars cut 17 lines to
13). Together this took the notes block from ~120mm tall to ~60mm tall,
freeing the room the schedule table needed.

**Lesson for next layout pass**: when adding a new block to an already-tight
open area, re-measure and if necessary resize *every* neighbor, not just
place the new one — a block that used to have plenty of room can end up
crushed by a neighbor that grew, with no error or warning from FreeCAD.

## Colored TechDraw hatch — SOLVED, 2026-09-05 — `FaceColor` + `FaceTransparency=0`, per-object views need their own bbox anchor

Fourth review pass. Erik: too many dimensions/overlaps, drop the `ZoneSchedule` table
now that dimensions carry real sizes, colour the hatch to match each zone, and the
legend's descriptive text is too thick/heavy.

**Colored hatch — three findings, in the order that actually gets you there:**

1. `TechDraw::DrawViewPart.ViewObject.ShapeColor` doesn't exist; the relevant property is
   `ViewObject.FaceColor` (a single uniform colour for **the whole view**, not per source
   object). Grepping the exported SVG for `fill:` (CSS syntax) found nothing and looked
   like TechDraw was hard-coded monochrome — **wrong conclusion**, caused by grepping the
   wrong syntax. TechDraw emits **SVG attribute** syntax, `fill="#rrggbb"`, not a CSS
   `style="fill:..."` string. Grep `fill="#` and the colours are there.
2. Even with the right colour present, it didn't render: `ViewObject.FaceTransparency`
   defaults to **100** (fully transparent) on every `DrawViewPart`. Confirmed by finding
   `fill="#a83821" fill-opacity="0"` in the exported SVG for a correctly-coloured hatch
   rod's face — the colour was already correct, the face was just invisible. Set
   `FaceTransparency = 0` and the fill appears. This is the actual fix, not `FaceColor`
   alone.
3. Also set `ViewObject.LineWidth` low (e.g. `0.05` mm) on the colour-carrying view. The
   default `0.7` mm line width draws a black outline around each thin hatch rod that is
   *wider than the rod itself at 1:100* (25 mm rod = 0.25 mm on paper), so the coloured
   face fill is completely hidden behind its own outline stroke even after fixing (1) and
   (2). This is also why the legend's descriptive text looked bold/heavy — see below,
   same root cause, different symptom.

**Because `FaceColor` is per-view, colouring N zones means N separate
`TechDraw::DrawViewPart` objects**, each `Source = [that zone's hatch-rod objects only]`,
each with its own `FaceColor`/`FaceTransparency=0`/low `LineWidth`, all stacked at the
same `X`/`Y`/`Scale`/`Direction` as the base view so they overlay exactly. Move (not
copy) the hatch rods out of `FloorPlan.Source` into these new views so they don't render
twice (once black via the base view, once coloured via the new one).

**The bbox-anchor trap this creates, and the fix**: `DrawViewPart.X`/`Y` anchor the
**centre of that view's own projected bounding box** (established doc convention,
repeated here because it bites differently with per-zone Source lists). Copying
`FloorPlan.X`/`Y` onto a new view whose `Source` is only one zone's hatch rods does
**not** put it in the right place — that view's own bbox is much smaller than
`FloorPlan`'s full-building bbox, so the same anchor coordinate now centres a different,
smaller box, and the hatch renders shifted into the middle of the drawing (all zones'
hatch bunched together) instead of over its own zone. Symptom looked exactly like a
coordinate-transform bug but was pure bbox-anchor mismatch.

**Fix: give every per-zone colour view an "anchor" pair of tiny (0.02–0.5 mm) cube
solids at the exact same two extreme corners used for the base view's own full bbox** —
add them to each colour view's `Source` alongside the real hatch rods. This forces every
colour view's bbox to match the base view's bbox exactly, so the same `X`/`Y` anchor
lands them all in the correct, identical position. Keep the cubes' `ViewObject.Visibility
= False` (harmless either way since TechDraw projects everything in `Source` regardless
of that flag) and size them small enough (≤0.02 mm at 1:1 legend scale, ≤0.5 mm at 1:100
floor-plan scale) that they don't leave a visible dot in the export.

**Second trap: this anchor breaks again if you change unrelated geometry that shares the
view.** Swapping the legend's descriptive-text font (see below) shrank
`LegendView`'s own local bbox (proportional font is narrower than the old monospace),
which shifted where `LegendView`'s *own* content rendered (`X`/`Y` anchor = bbox centre,
and the bbox moved) — but the separately-anchored `LegendHatchColor_*` views, anchored to
the **old**, now-stale corner coordinates, didn't move with it. Result: swatch and hatch
visibly detached from each other on the page, even though every underlying model
coordinate was still numerically correct (verified directly — this was a pure rendering/
anchor-position issue, not a geometry bug). **Fix: any time a view's Source content
changes shape/size (font swap, added/removed objects, resized geometry), re-derive and
re-set every dependent view's anchor cubes from the CURRENT bbox, not a value computed
once at the start of the session.**

**Legend text "too thick/heavy" — same LineWidth-vs-glyph-size issue, not font weight.**
Bold monospace font at small size read as heavy; switching `FontFile` to a regular-weight
font (tried `DejaVuSansMono.ttf`, then `LiberationSans-Regular.ttf`) changed the glyph
*shape* (confirmed via bbox width change, e.g. `ShapeString`'s width dropped from ~109 mm
to ~84 mm on the font swap) but the rendered PDF looked **identical** — because
`LegendView`'s `ViewObject.LineWidth` (default `0.7` mm) was drawing a heavy black
outline around every glyph regardless of the underlying font. The actual fix was
`LegendView.ViewObject.LineWidth = 0.1` (down from 0.7) — combined with the regular-
weight font, this produces genuinely thin, readable outline text. **If a ShapeString-
based label looks bold no matter what font you assign, check the view's `LineWidth`
before assuming the font didn't take.**

**Full recompute discipline for this kind of multi-view edit**: after touching several
interdependent views/objects, `doc.recompute()` alone was not always enough to get a
correct re-export — use `doc.recompute(None, True, True)` (force flag) before calling
`page.ViewObject.doubleClicked()`, and consider `FreeCADGui.updateGui()` between steps to
let Qt process the redraw before `TechDrawGui.exportPageAsPdf()` runs. A stale export
(old geometry, unchanged pixel-for-pixel across a real property edit) was the first
symptom that surfaced this — always diff the actual PDF/PNG after a change, don't trust
that a successful `recompute()` call means the exported file reflects it.

**Dimension chains replacing per-zone floating pairs**: rebuilt from 12 dimensions (5
zones × width+depth pair + 2 overall) to 11, organised as touching chains rather than
independent floating pairs — removed genuine duplicates (H and F share the same X-span
3000–7000 so both `_Width` dims measured the identical distance; W and C share the same
0–2000 X-span likewise) and replaced them with a single 4-segment `DistanceX` chain
(0→2000→3000→4500→7000, one dimension per segment, all referencing real
`getVertexBySelection` vertices at those X coordinates regardless of their Y — a
dimension's witness/extension line is free to run any length to reach the datum line, so
picking a vertex anywhere along a needed X or Y is fine) plus a 3-segment `DistanceY`
chain along the naturally-contiguous right column (bulk-store/F/H, 0→3500→8000→12000).
Two genuinely isolated dims (`W_Depth`, `C_Depth`) don't share a boundary with anything
else and were kept as-is. Every value re-verified with `getRawValue()` after the
rebuild — chain segments sum exactly to the values they replaced (2000+1000+1500+2500 =
7000; 3500+4500+4000 = 12000).

**Positioning the "total" dimension relative to a chain is genuinely unpredictable —
budget several iterations.** Placing `Dim_Overall_Width` below its own 4-segment chain
(itself at `Y=-20`) hit three different overlaps across four attempted `Y` values before
landing clear: `Y=-12` (original) crossed the template's own grid-reference tick mark;
`Y=-18/-20/-25` (modest nudges toward the chain) landed exactly on top of the chain's own
text; `Y=-40` — a **single larger jump** — didn't move it further along the same line as
expected, it jumped to overlapping unrelated geometry deep inside the W zone, matching
this doc's existing "don't teleport" warning but demonstrating it applies to a total-vs-
chain relationship too, not just dimension-vs-dimension. What worked: keep jumping in the
same direction in large steps until clearly past everything (`Y=-150` landed it well past
the bottom of the whole drawing), then walk it back up in modest steps (`Y=-128`) until it
sits just clear of the plan with its own witness lines — i.e. bracket first, then narrow,
rather than incrementally creeping from the original position.

**`ZoneSchedule` removal**: `page.removeView(sched)` then `doc.removeObject("ZoneSchedule")`
— removing from the page's `Views` list first is required, `removeObject` alone leaves a
dangling reference. The underlying `Parameters` spreadsheet range is untouched; only the
on-page table view is gone. Freed vertical space was left as clean margin rather than
backfilled — legend and notes already had comfortable room, and Erik's brief explicitly
allowed "just leave clean margin" as a valid outcome.

## Finding the title-block ceiling and fitting notes into a fixed-height band, 2026-09-05

Erik: notes must move to bottom-left, top edge no higher than the title block's top
edge; legend stays top-right unchanged. **Method to get the real ceiling coordinate**:
grep the template SVG for `title_block_frame` (`x/y/width/height`, SVG top-down coords)
and `drawing_space_frame` (the overall usable-area border) - do not guess from the
rendered image. On this template: `title_block_frame` y=239,height=48 (spans SVG y
239->287); `drawing_space_frame` x=20,y=10,width=390,height=277 (spans SVG y 10->287,
x 20->410). TechDraw's own `X`/`Y` page coordinates are **Y-up from the bottom** (the
opposite of the SVG's Y-down), confirmed by cross-checking known object positions
(`LegendView.Y=255` sits near the top, `FloorPlan.Y~155-200` sits mid-page) against
where they render. Convert: `TechDraw_Y = page_height - svg_y`. So the title block's
top edge, in TechDraw Y, is `297 - 239 = 58` - that is the hard ceiling for a
bottom-left block's top edge. The overall drawing-space frame gives the floor: its
bottom edge is `297 - 287 = 10`. **Available band for a bottom-left block: Y 10 to 58
(48mm tall), X 20 to 230 (210mm wide, i.e. left of where the title block starts)** -
generous horizontally, tight vertically.

**Line pitch on `DrawViewAnnotation` has a floor that `TextSize` alone barely moves -
use `LineSpace` to actually compress a block.** Cutting `TextSize` from 3.0 to 2.3 (a
23% reduction) shrank a 13-line notes block's rendered height by only ~5%, confirmed by
pixel-measuring line-to-line pitch in the rasterized PDF before/after (23.1px -> 21.9px
at 150dpi) - nowhere near proportional. `LineSpace` (default ~100, a percent-of-em
value) has a much bigger, close-to-linear effect: dropping it from 70 to 50 alongside
`TextSize=2.3` took the same 13-line block from ~46-47mm tall (touching the page's
bottom border with ~0px clearance) to ~32.5mm tall (clean ~8.5mm gap at the ceiling,
~6.4mm gap at the floor) - now comfortably inside the 48mm band. **If a
`DrawViewAnnotation` block doesn't fit its allotted vertical band, reach for
`LineSpace` before shrinking `TextSize` further** - shrinking `TextSize` alone will run
out of usable font size long before it frees enough room.

**Verification method that actually catches a too-tight fit**: don't just look at the
render, measure it. Rasterize at 150dpi, find the page's bottom border row
programmatically (`np.where((row_slice<128).mean()>0.8)` scanning full page width -
the frame border is one of the only rows that's >80% dark edge-to-edge), then find the
topmost/bottommost ink row of the text block in its known X range and diff against
that border row and against the computed ceiling-in-pixels. A block that "looks fine"
zoomed out can still have its last line's descenders touching the border by 0-2px,
which is exactly what happened here on the first two attempts (`TextSize` reduction
alone) before `LineSpace` fixed it.

**Reflow note**: moving `NotesView` into the vacated bottom-left band required also
shifting `FloorPlan.Y` up (155 -> 201, i.e. +46mm) so its own bottom-most dimension
line (the overall-width `7000` dimension, which extends well below the zone geometry
itself) cleared the new notes band with margin. `FloorPlan`'s rendered footprint
including radiating dimension lines is much bigger than just the scaled building
outline - measure the *dimensioned* cluster's extent, not just the raw zone bbox, when
deciding how much clearance a neighbor needs.

## Editing this drawing interactively — what Erik can actually do at the GUI, 2026-09-05

Erik's complaint: everything so far has been one-shot `execute_python` script pushes,
and he wants the normal FreeCAD experience — a freely-navigable 3D model with a
TechDraw page that just reflects it, rotate/zoom/edit by hand, watch it update.
Investigated directly on `09 - Okracandle/Property/Unit 01 Zone Layout.FCStd`
(FreeCAD 1.1.3, xmlrpc bridge). Findings below are tested, not assumed.

**1. The 3D view is real and navigable, but you have to open one — and there's a
landmine in it.** No 3D view was open when this document loads by default; the only
MDI tab was the TechDraw Page. Opening one is standard FreeCAD (`Std_ViewCreate`, or
in the GUI: View menu → nothing needs scripting, just click the document and a fresh
3D view opens/any existing one activates) and rotate/zoom/pan then work exactly like
any other FreeCAD model — this part is not clunky, it's just that nobody had opened
the tab yet. **But two leftover objects, `Wall` and `Wall001`, are visible by
default and are opaque 3m-tall solids spanning the entire 7000x12000mm footprint** —
an abandoned early attempt at the walls (superseded by `WallFrame`/`PartitionFrame`,
per this doc's own build history) that was never deleted or hidden. They contribute
nothing to `FloorPlan.Source` (confirmed: not in the 14-object source list) and don't
affect the exported PDF at all, but in the 3D view they sit as one giant gray block
directly over the real colour-coded, hatched zone layout underneath, hiding it
completely from any angle. **This is the main reason opening the 3D view "feels
wrong" — it isn't a FreeCAD limitation, it's clutter left over from the build.**
Hiding `Wall`/`Wall001` (right-click → Toggle visibility, or Spacebar in the tree)
immediately reveals the correct, fully-colored, fully-hatched floor plan, confirmed
by screenshot. **Recommended fix, not yet applied** (left as found per this
investigation's scope): delete or permanently hide `Wall`/`Wall001` in a future pass.

**2. Recompute propagation genuinely works, standard FreeCAD dependency-graph
behavior, no special-casing.** Tested directly: moved `Rectangle002_slab` (the C
zone/bathroom solid) by -50mm in Y via `Placement`, ran a plain `doc.recompute()`,
then queried `FloorPlan.getVertexBySelection()` (the same API `References2D` uses) —
the zone's corner vertex in the TechDraw view moved by exactly 50mm, matching the
edit. Reverted the placement to its exact original value, recomputed again, and
confirmed the vertex returned to its original position, document back to
"Up-to-date," zero errors, same 175-object count as before. **So: if you select a
zone slab in the 3D view and drag/nudge its Placement (position), the TechDraw page
updates automatically on next recompute — no script involved.** This is the one part
of the workflow that already matches the normal FreeCAD mental model.

**3. What that Placement-drag can't do — the real limits, checked, not assumed:**

- **Position, yes; size, no.** Each zone slab (`Rectangle002_slab` etc.) is a plain
  `Part::Feature` with a Shape baked once by the build script — it has a `Placement`
  (movable) but no dimensional parameters (no Length/Width property, no Sketcher
  constraints, `ExpressionEngine` empty). You can drag it to a new position and it
  will recompute into the page correctly. You cannot resize it by dragging a handle —
  there is no handle, because the shape itself is a static blob of geometry, not a
  parametric primitive. Resizing a zone means editing the `Parameters` spreadsheet
  and re-running the build script to regenerate that slab's `Shape` from scratch.
- **The Draft rectangles in the tree are a red herring.** Each zone slab has a
  same-numbered Draft rectangle sibling (`Rectangle002` next to `Rectangle002_slab`)
  that looks like the normal editable Draft primitive you'd expect to drag/resize in
  the GUI. **Checked directly: it is completely disconnected** — `Rectangle002.OutList`
  and `.InList` are both empty, meaning nothing reads its geometry. Editing it does
  literally nothing to the visible drawing. It's a leftover intermediate from the
  build process (the rectangle used once to generate the slab's extruded Shape),
  not a live driver. Don't edit these expecting an effect.
- **Hatch lines and zone labels don't follow the slab.** Confirmed by the same
  Placement-move test above: after moving the C zone slab, a vertex that should have
  moved with it (the hatch rod geometry) was still sitting at the old, un-moved
  coordinate. The ~185 `Hatch_*`/`LegendHatch_*` rod objects and the `ShapeString`
  letter labels are independent scripted geometry generated once from the zone's
  real-world bounding box at build time — they have no dependency link to the zone
  slab (or to each other). Moving or resizing a zone by hand desyncs its hatch
  pattern and label from its new outline; only a script re-run regenerates them
  correctly in the new position.
- **The title block does not live-sync from the spreadsheet — confirmed, and it's
  actually a stronger gap than previously documented.** `Template.ExpressionEngine`
  is empty and the `Parameters` spreadsheet's `OutList` is empty — nothing in the
  current document reads from the spreadsheet live at all (the one thing that used
  to, `ZoneSchedule`, was deleted in this drawing's sixth revision pass). Editing a
  spreadsheet cell interactively changes nothing else in the document until a script
  re-runs the push-to-`EditableTexts` step. Spreadsheet edits alone are not currently
  useful without a script/agent follow-up.

**Bottom line for Erik, honestly:** rotating, zooming, panning the 3D model and
toggling what's visible is fully normal FreeCAD right now (once `Wall`/`Wall001` are
hidden) — no agent needed for that. Nudging a zone's *position* by hand and watching
the TechDraw page follow also genuinely works today, no script needed, verified live.
Everything else that reads as "the layout" — a zone's *size*, its hatch pattern, its
label position, and anything driven by the `Parameters` spreadsheet (title block,
schedule data) — still requires a script/agent pass to regenerate correctly, because
those pieces were built as one-shot generated geometry rather than as parametric
Sketch-driven features. That's a real gap against "true PartDesign-style parametric,"
not a GUI limitation — the fix (if wanted later) is rebuilding the zone slabs as
Sketch+Pad features with named, spreadsheet-bound dimensional constraints instead of
Python-extruded static shapes, which is a real rebuild, not a quick patch.

## Zone + hatch + label as ONE coupled object — SOLVED, 2026-09-05, phase 2a of the Sketcher migration

**Standing pattern now, superseding the separate "sketch + baked hatch-rod objects +
baked label" approach used everywhere above.** Erik's requirement: "hatch and zones and
all need to be together... hatching should be a property of the zone" — resizing a
zone's sketch and recomputing must move/resize its hatch and letter automatically, with
zero manual re-baking or re-positioning. Real precedent checked:
[chopinregis/FreeCAD-Parametric-Hatch-Generator](https://github.com/chopinregis/FreeCAD-Parametric-Hatch-Generator).

**Mechanism: `Part::FeaturePython`, one object per zone (`Zone_H`, `Zone_F`, `Zone_S`,
`Zone_W`, `Zone_C`), Proxy class in a module file on `sys.path`, not defined inline in
the console.** `App.getUserMacroDir(True)` (`~/.local/share/FreeCAD/v1-1/Macro`) is
already on `sys.path` at FreeCAD startup — put the class there
(`zone_hatch_feature.py`) so the document can `import` it and unpickle the Proxy on
reopen. A class defined ad-hoc in an `execute_python` call is NOT importable later and
the document will fail to restore the Proxy on next load — this is not optional
housekeeping, it's the difference between a feature that survives a restart and one that
doesn't.

**Portability gap, real, not hypothetical — fixed 2026-09-05.** The Macro directory is
per-machine, not synced or version-controlled, so `zone_hatch_feature.py` living only
at `~/.local/share/FreeCAD/v1-1/Macro/zone_hatch_feature.py` means a fresh machine (or
anyone else opening this .FCStd) fails to recompute every `Zone_*` object with an
import/unpickle error. **The source of truth is now
`10 - Side Hustle/Drafting Service/zone_hatch_feature.py`** in the vault (tracked,
synced). A future FreeCAD session opening `Unit 01 Zone Layout.FCStd` (or any document
using this pattern) on a machine where the Macro-dir copy is missing must copy (or
symlink) that vault file into the local FreeCAD Macro directory
(`App.getUserMacroDir(True)`) *before* the first recompute — otherwise every zone
object's Proxy fails to restore. The original Macro-dir copy is left in place on this
machine so the currently-working session isn't broken; treat the vault copy as
authoritative for edits and `diff` the two if in doubt which is current.

```python
obj = doc.addObject("Part::FeaturePython", "Zone_H")
ZoneHatchFeature(obj, sketch=doc.Sketch_H, letter="H", color=(0.66,0.22,0.13,1.0),
                 hatch_angles=[45.0], spacing=300.0, rod_width=32.0, cross=False)
ZoneHatchViewProvider(obj.ViewObject)
```

`execute(obj)` each recompute: reads `obj.BoundarySketch.Shape.Wires[0]` fresh (not
cached), builds `Part.Face(wire).extrude(...)` for the zone slab, generates hatch rods
(same Liang-Barsky-clipped rotated-rod technique as the old baked approach, computed
fresh from the live `bbox` every time — for a cross-hatch zone (C), gap the horizontal
rods wherever they cross a vertical one, same closed-loop-fill-merge bug as before),
builds the letter label via `Part.makeWireString(letter, fontfile, size, 0)` (the
non-GUI primitive behind `Draft.make_shapestring` — use this, not
`Draft.make_shapestring` itself, inside `execute()`: the Draft call creates a persistent
document object each time it's called, so calling it every recompute would leave a
growing trail of orphaned ShapeString objects), centres it on the boundary's own bbox,
and sets `obj.Shape = Part.makeCompound([zone_solid, *rods, label_solid])`. One object,
one `Shape`, one `BoundarySketch` link — editing the sketch and recomputing regenerates
everything. Verified: `Zone_H.Shape.isValid()`, `len(Solids)` matches rod+label+slab
count, all 5 zones valid.

**Real, hard limitation found doing this: `TechDraw::DrawViewPart` only fills faces
with colour (`ViewObject.FaceColor` + `FaceTransparency=0`, per the existing "Colored
TechDraw hatch" section above) when its `Source` is a SINGLE-SOLID object.** Confirmed
empirically, not assumed: a lone `Part.makeBox` in `Source` filled correctly (colour
pixel count jumped 5→31 in a controlled A/B export); the *exact same box* put in a
`Part.makeCompound([box])` of just one member still worked, but a compound of **two**
touching boxes dropped straight back to baseline (~5, i.e. no fill at all) — reproduced
three times, not a fluke. This means the zone's own `Shape` (compound of slab + hatch
rods + label, needed for one-object coupling) can NEVER be colour-filled directly by a
`DrawViewPart`, no matter what `FaceColor`/`FaceTransparency` are set to.

**Resolution: the FeaturePython's `execute()` also maintains a small companion
`Part::Feature` (`Zone_H_ColorFill` etc.) holding ONLY the plain boundary solid** (no
rods, no label — a single solid, satisfying the fill requirement above), auto-created on
first execute and its `.Shape`/`.ViewObject.ShapeColor` refreshed every execute after.
This is not a second thing for Erik to maintain — it's fully derived, named to read as
generated (`Label = "... (colour fill, auto-generated - do not edit)"`), and can never
drift out of sync with the sketch because the same `execute()` call that updates the
zone's own outline/hatch/label also updates it. A separate `TechDraw::DrawViewPart`
(`ColorFill_H` etc.) sources ONLY this companion, with `FaceColor`/`FaceTransparency=0`/
low `LineWidth` per the existing pattern — this is what actually shows the colour; the
zone's own compound stays in `FloorPlan.Source` uncoloured (default black outline/hatch/
label, exactly as before), and the coloured view sits **underneath** it as a background
tint (`FaceTransparency ≈ 55` reads as a wash, not a block, and lets the black hatch/
outline/letter show clearly on top — this is a better look than the old
hatch-rods-only colouring, which Erik had already flagged as reading "basically still
black background" at normal viewing distance).

**Z-order for the tint is NOT controlled by `page.Views` list order or object creation
order — tested both, neither had any effect.** Reordering `page.Views` so the colour
views appeared first in the list made no visible difference; deleting and recreating
`FloorPlan` *after* the colour views (so it would have a later internal ID) also made no
difference. What actually determines whether the tint renders behind or in front of the
black line art was never conclusively isolated — but empirically, on this build, colour
views added to the page consistently rendered **underneath** views like `FloorPlan` that
already existed on the page before them, once each was correctly (re)computed and
`page.ViewObject.doubleClicked()` had run. Don't assume this is universal; verify by
rasterizing every time, same discipline as everywhere else in this document.

**The single-zone colour view needs its own `X`/`Y` recomputed on every resize, and this
is now automated inside the same `execute()`, not a manual step.** `DrawViewPart.X`/`Y`
anchor the *centre* of that view's own current bbox (established convention, see
"Colored TechDraw hatch" above) — a zone that grows asymmetrically (one edge pinned to a
wall, the opposite edge moves, the normal case) shifts its own bbox centre in real-world
space, and a fixed `X`/`Y` would silently re-centre the OLD anchor point on the NEW
shape, cancelling the shift instead of showing it (confirmed: after a resize with a
stale anchor, the tint rendered narrower than the now-wider black outline/hatch — a
visible, wrong result). Fix, done inside `ZoneHatchFeature.execute()` itself every time:
recompute the tint view's `X`/`Y` from `(zone_bbox_center - wall_bbox_center) * scale +
FloorPlan.X/Y`, using the static `Sketch_WallFrame` bbox as the fixed reference point
and `FloorPlan.X/Y/Scale` as the base anchor:

```python
wbb = doc.Sketch_WallFrame.Shape.BoundBox
building_cx, building_cy = (wbb.XMin+wbb.XMax)/2.0, (wbb.YMin+wbb.YMax)/2.0
zbb = zone_solid.BoundBox
zone_cx, zone_cy = (zbb.XMin+zbb.XMax)/2.0, (zbb.YMin+zbb.YMax)/2.0
scale = float(fp.Scale)
view.X = float(fp.X) + (zone_cx - building_cx) * scale
view.Y = float(fp.Y) + (zone_cy - building_cy) * scale
```

This is an alternative to the "anchor cube" trick in the "Colored TechDraw hatch"
section above (matching the base view's bbox exactly by adding invisible calibration
geometry) — either works; this one avoids adding extra geometry per view, at the cost of
needing a known static reference object (`Sketch_WallFrame`) to compute against. Verified
by direct round-trip: widened `Sketch_H`'s `H_Width` constraint 4000→4200mm by hand
(`sk.setDatum('H_Width', 4200.0)`), recomputed — `Zone_H.Shape.BoundBox` and
`Zone_H_ColorFill.Shape.BoundBox` both updated automatically (`XMax` 6306→6506) with
**zero code touching either object directly**, and `ColorFill_H.X` moved automatically
100.38→101.38 (half the 200mm growth at 1:100 scale, exactly as expected from the
formula) — then reverted `H_Width` back to 4000 and confirmed both bboxes and the anchor
returned to their exact original values. Rasterized and read at each step: the black
hatch/outline/letter and the colour tint both visibly grew together on the resize and
both visibly reverted together — this is the actual proof the coupling works, not just
the numbers.

**The intermittent stale/blank TechDraw export bug (documented in the "Hatch density
increase" section above) recurred here and is a real cost, not a one-off**: after a
`setDatum` change + `recompute()`, the FIRST re-export was pixel-identical to the
pre-change render (confirmed via `ImageChops.difference(...).getbbox() is None`) despite
every object correctly reporting the new geometry programmatically — only after
deleting and recreating `FloorPlan` and the affected `ColorFill_H` view (not just
`recompute()`/`doubleClicked()` again) did a SECOND export actually show the change, and
even then a subsequent one-shot export of `ColorFill_H` alone came back blank once more
before a third attempt rendered correctly. **Budget 2-3 export/rasterize cycles per
verification pass on this file, every time** — a single clean recompute + a single PDF
export is not sufficient evidence that the render reflects the current model; diff
against the previous export or eyeball it before trusting any claim about what changed.

## Other gotchas hit building Unit 01

- `ShapeColor` lives on `.ViewObject`, not the document object directly —
  `obj.ViewObject.ShapeColor = (...)`, not `obj.ShapeColor = (...)`.
- A `Quantity` (e.g. `view.X`) can't be mixed with a plain float in arithmetic —
  `ArithmeticError: Unit mismatch`. Convert with `float(view.X)` first.
- Don't call `doc.save()` before the document has ever been saved —
  `ValueError: Object attribute 'FileName' is not set`. Use `doc.saveAs(path)` once,
  `doc.save()` after.
- A dict value that includes a bound method (e.g. trying to return
  `qty.getValueAs('m^2')` from `execute_python`) fails XML-RPC marshalling with
  `TypeError: cannot marshal <class 'builtin_function_or_method'> objects` — return
  `float(qty.Value)` instead.

## Phase 2b, 2026-09-05: dimensions, door symbol, final export — migration complete

Three things done against the phase-2a coupled zone geometry: (1) copied
`zone_hatch_feature.py` into the vault (now `10 - Side Hustle/Drafting Service/`) as the
portability fix — see the README there and the "portability gap" note added to the
"Zone + hatch + label" section above; (2) 10 dimensions (a 3-segment interior width
chain + 1 isolated zone-width dim + 1 envelope-width dim, and a 3-segment interior
depth chain + 2 isolated zone-depth dims); (3) a door symbol plus, discovered as
in-scope work, an actual door opening cut into `Sketch_WallFrame` (it had none —
the wall was a solid, continuous frame with no gap for the roller door).

**`TechDraw::DrawViewDimension.X`/`.Y` are NOT the same coordinate frame as
`DrawViewPart`/`DrawViewAnnotation`.** This doc's own earlier dimension section
("nudge, don't teleport") was written against dimensions on a *different*,
simpler view and undersold how confusing this gets on a second attempt. Confirmed
by direct calibration (set a known input, export SVG, read the absolute rendered
`<text>` transform stack, solve for the mapping): for a **`DistanceY`** dimension's
text, rendered SVG-y-down mm = `127.79 - Y_input` (slope **-1**, not the `+1` you'd
expect from a direct page-mm coordinate); for a **`DistanceX`** dimension's text,
rendered SVG-y-down mm = `100.53 - Y_input` (same -1 slope, different constant —
the constant is NOT shared between the two dimension types, don't assume one
calibration transfers to the other). The **X** property does not follow a clean
linear map either — empirically it behaves like a small multiplier/offset near the
dimension's own auto-computed anchor rather than an absolute page coordinate, and
attempts to compute it analytically (matching the `DrawViewPart` X/Y formula) put
dimension lines and witness lines in wildly wrong places (off-page entirely, or
crossing through unrelated zone geometry). **What actually worked**: calibrate Y
using the formula above (two known data points of the same dimension TYPE are
enough — solve the two-point line), leave X at a reasonable guess, render, and fix
any remaining overlap by eye with modest empirical nudges — this is still a
render/inspect/adjust loop, not a closed-form placement, budget several PDF
export/rasterize cycles. **Also check `ViewObject.Fontsize` before troubleshooting
position** — its default is `10.0 mm`, enormous at A3 1:100 (compare
`NotesView.TextSize=2.3mm`); oversized text overflowing far past its anchor point
was mistaken for a positioning bug on the first pass here before the real fix
(`Fontsize = "3.5 mm"`, `Arrowsize = "2.0 mm"`) made everything legible and
revealed the actual (correct) anchor positions.

**Real, confirmed fragility: `TechDraw::DrawViewDimension.References2D` pointing at
a `Vertex`/`EdgeN` on a `Part::FeaturePython` zone object (the coupled
zone+hatch+label compound from phase 2a) is NOT stable across a resize, even
though the underlying geometry is correct.** Round-trip tested directly: widened
`Sketch_H.H_Width` 4000→4200mm by hand, recomputed — `Zone_H.Shape.BoundBox`
updated correctly (`XMax` 6306→6506, confirmed programmatically and matching the
phase-2a claim), but the dimension referencing that corner (`Dim_Width_FH`) still
reported the *old* value (4000), then after reverting `H_Width` back to 4000 the
dimension read a *stale* 4200 — i.e. it lagged one step behind, tracking neither
the live nor the reverted state reliably. Root cause: TechDraw assigns
`VertexN`/`EdgeN` numbers by scanning the WHOLE view's merged HLR result across
every object in `Source` together, and the zone's own hatch-rod count (part of
`Zone_H.Shape`, a compound) changes with the zone's size — that reshuffles the
global vertex numbering for the *entire view*, not just the zone that resized.
**Tried and did NOT fix it**: adding the zone's plain boundary `Sketcher::SketchObject`
(e.g. `Sketch_H`) into `FloorPlan.Source` *alongside* the already-present `Zone_H`
compound, on the theory that a small, stable 4-vertex rectangle would give an
independently-numbered, resize-proof reference. It didn't — TechDraw deduplicates
coincident vertices across different `Source` objects into a single shared
`VertexN` identity (confirmed: scanning found exactly one vertex number per
projected coordinate, not two, even with both the sketch and the zone compound
both contributing a vertex at that exact point), so the merged ID is still
downstream of the *whole view's* topology and still shifts when the zone's hatch
count changes elsewhere in the same `Source` list. **Net conclusion: there is no
cheap fix for this within the current "hatch rods as part of the zone's own Shape"
architecture** — any dimension on this drawing that references a zone-boundary
vertex/edge must be treated as build-time-correct-but-not-resize-safe, and
**re-verified (and likely re-built from a fresh `getVertexBySelection` scan) after
any zone resize**, exactly like re-running the old push-to-`EditableTexts` step
after a spreadsheet edit — not a one-time build-and-forget. Dimensions referencing
`Sketch_WallFrame`/`Sketch_C_Partition` directly (not through a zone compound) do
not have this problem — their vertex count never changes. On this drawing, all 10
dimensions were rebuilt with a fresh vertex scan against the FINAL `FloorPlan.Source`
order and every value re-verified with `getRawValue()` immediately before the last
save — correct **as delivered**, with this fragility flagged for whoever edits a
zone's size next. A real fix, if ever needed, would separate the hatch/label
geometry from the zone's dimensionable boundary edges (e.g. keep a stable,
un-hatched proxy solid purely for `References2D` to lock onto) — not attempted
here, out of scope for this pass.

**Cutting a door opening into an existing "two nested rectangle" wall sketch.**
`Sketch_WallFrame` (phase 1) was a continuous frame with no gap — there is no
shortcut for adding an opening to an existing simple rectangle pair; it required a
full geometry rebuild: delete all geometry, rebuild as **12 line segments** (the 4
untouched sides + the front split into left/right stubs on both the outer and
inner edges + 2 vertical "jamb" segments closing the wall thickness at each side of
the opening), re-add 12 `Coincident` constraints to close the loop, 12 `Horizontal`/
`Vertical` constraints, and the original named dimensions reapplied to the *correct
new geometry indices* (most stayed numerically identical since the untouched sides
kept their meaning — `Overall_Depth`, `WallThickness_X/Y` — but `Overall_Width` and
`Inner_Width`, which used to span the geometry no longer contiguous through the
door gap, had to be re-anchored to different corner vertices that are still
connected through the loop). Added `Door_Width` (3000mm) and `Door_Offset_Left`
(1768mm, i.e. centred on the 6536mm outer width) as new named constraints.
**Getting to `FullyConstrained` took one extra iteration**: after all the coincidences
+ H/V + the 8 named dimensions, `solve()==0` (consistent) but `FullyConstrained`
was `False` — 1 residual DOF. Diagnosed by testing candidate extra constraints one
at a time (each either came back `-2` redundant/conflicting, telling you that
direction was already pinned, or actually removed the last DOF). The real gap
here: dimensioning the **right** side's wall thickness only pinned that side's
front-to-back relationship via the front-left corner anchor + `Inner_Width`/
`Inner_Depth` on the *lengths* of the right/inner-right edges — nothing tied the
inner rectangle's **back** edge position independently, so the inner rectangle
could theoretically shift depth-wise while every named dimension still checked out.
Fixed with one more `DistanceY` from the sketch origin to the inner-back-left
corner (named `WallThickness_Back_Y`, value 12230 = the inner corner's absolute Y).
**Lesson: a "two rectangles, four thickness dimensions" wall is fully constrained
by symmetry/convention, but the moment you cut a doorway and the front stops being
one contiguous edge, re-derive DOF count from scratch — don't assume the same
constraint count that worked for the simple rectangle still closes the loop.**

**Door symbol convention used**: not a swing arc — a roller/sectional door in plan
is drawn as a series of parallel lines spanning the opening (representing the
slatted curtain, viewed edge-on) plus a small boxed rectangle at the head
representing the housing unit. Built as a hand-authored inline SVG string (5
horizontal lines + 1 rectangle, ~14 lines of SVG) assigned directly to a
`TechDraw::DrawViewSymbol.Symbol` property (a plain string of SVG markup, not a
file reference), added via `page.addView()`, positioned with the same "set X/Y
after addView()" gotcha as every other view type on this page. Real-world/page-mm
correspondence for `DrawViewSymbol` matched the plain `DrawViewPart`/
`DrawViewAnnotation` X/Y convention (center-of-bbox anchor, direct scale — no
special calibration needed, unlike `DrawViewDimension` above), confirmed by placing
it exactly over the computed door-opening coordinate and rasterizing to check.

**Final state confirmed 2026-09-05**: full document recompute clean (70 objects, 0
errors), one page (`pdfinfo`), `pdftotext` shows every dimension value, every note,
and the title block as real embedded text, and the rasterized PDF was read
directly to confirm the floor plan (geometry/colour/hatch/labels), all 10
dimensions (chains + isolated + envelope, no overlapping text anywhere), the door
symbol (correctly centred in the actual wall opening, not floating over solid
wall), the legend, the notes, and the title block all sit clear of each other on
one page. This is the final state of the Sketcher migration — the remaining known
gap is the dimension-fragility-on-zone-resize finding above, which is a
documented, flagged limitation rather than a silent one.

## Phase 3, 2026-09-06 — confirmed vertex-index fragility extends to ANY `FloorPlan.Source` edit, not just resize

The phase 2b finding above ("dimensions referencing a vertex on a `Zone_*` coupled
object are not resize-safe") was already known to be triggered by resizing a zone.
**This session found the same failure triggered by something much smaller: moving
one unrelated object's `Placement` (a symbol, not a zone) while it was still a
member of `FloorPlan.Source`, with zero change to its shape or size.** Moving
`DBSubBoard` sideways by 200mm to fix a real clipping-against-the-wall defect
shifted the HLR `VertexN` numbering for the *entire* `FloorPlan` view, and every
one of the 10 `TechDraw::DrawViewDimension` objects (which reference geometry by
that numbering via `References2D`) immediately reported wrong values — confirmed
numerically with `getRawValue()`, not just visually; e.g. `Dim_Depth_C` read `2000`
before the edit and `563.76` after, with no change to C's actual geometry.

**Worse: reverting the edit did not immediately fix it.** Setting `DBSubBoard.Placement`
straight back to identity and recomputing still showed the corrupted dimension
values. It took **two consecutive full forced recomputes**
(`doc.recompute(None, True, True)` called twice in a row, no object changes
between them) before the values settled back to the correct, spreadsheet-matching
set. A single recompute was not enough and there was no error or warning at any
point — every object reported `Up-to-date` throughout, including while the
dimensions were reading wrong values. This reads as the same underlying
caching/redraw race documented elsewhere in this file (the "intermittent stale
TechDraw export" section), extended to affect computed values, not just visual
staleness.

**Practical rule going forward**: treat `getRawValue()` on every dimension as part
of the verification step for *any* change that touches `FloorPlan.Source`
membership or any contained object's `Placement`/`Shape`, even a change that looks
completely unrelated to the dimensioned geometry (a symbol nowhere near the
dimensioned edges, in this case). Compare against the spreadsheet/known-good
values before trusting a re-export. If a fix to one object would require touching
`Source`-list geometry and the drawing has load-bearing dimensions, weigh the
fix's value against this risk — on `Unit 01 Zone Layout.FCStd` 2026-09-06 the
correct call was to leave a minor cosmetic clipping defect (`DBSubBoard` text
sitting ~8mm real-world, effectively 0mm at 1:100, from the wall inner face) unfixed
rather than re-risk the dimensions, since the dimensions are the harder
requirement. If recompute-settling is needed, budget for **at least two** forced
recomputes before reading values, not one.

**Correction, same day, caught by Erik reviewing the final export directly (not by
this section's own verification):** the "settled back to correct" claim above is
WRONG. The final exported PDF shows dimension values of 233.3mm, 2732.45mm, and
47.15mm — none of which are real geometry anywhere in this drawing (the actual
values are the clean chain 2000/76/4000 width, 3500/4500/4000 depth, 2500, 2000,
2000, 6536mm). The two-forced-recompute fix described above did NOT actually work,
or worked only transiently before a later export re-corrupted the values — this
was never re-verified against ground truth after the "settled" check, only against
`getRawValue()`'s own internally-consistent-but-wrong output. **Do not trust this
section's "settled back to correct" claim.** The real lesson stands (verify against
known-good spreadsheet/constraint values, not just internal consistency, treat two
recomputes as a minimum not a guarantee) but the specific outcome claimed for this
document is false. As of 2026-09-06 session close, `Unit 01 Zone Layout.FCStd`'s 10
dimensions are confirmed WRONG and logged as an open defect — next session must
rebuild them from scratch using `getEdgeBySelection`/`getVertexBySelection` against
current geometry and verify every value against the Parameters spreadsheet/sketch
constraints directly before trusting any export again.
