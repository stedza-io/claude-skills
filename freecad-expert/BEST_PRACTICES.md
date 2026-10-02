# FreeCAD Expert: Evolving Best Practices & Gotchas
*This file is automatically updated by Claude as it learns from your specific workflow.*

## THE HARD RULE — all geometry derives from sketches (Erik, 2026-09-08, non-negotiable)

**Every feature is driven by a fully-constrained sketch. The constraints live in the sketch;
that is where editability comes from — later you change a sketch dimension (or the VarSet it's
bound to) and everything downstream updates.**

- A cut is ONE sketch on a real datum plane, pocketed/padded. An **obround/slot is 2 arcs + 2
  tangent lines** (or the Sketcher slot tool) constrained to its pattern — **NOT three stacked
  circles, NOT a boolean of primitive solids, NOT `Part.makeCylinder` unions.**
- **Project/reference the mating part's geometry into the sketch plane** (external geometry,
  or dimensions bound to the driving VarSet) rather than eyeballing numeric offsets.
- No `execute_python` Part-module primitive-stacking for shapes that belong in a sketch.
  `Part::Feature` boolean hacks are not parametric and Erik will reject them on sight.
- If a shape genuinely cannot be sketched (helical sweep, lofted transition), say so
  explicitly and get agreement before scripting it.

This has regressed across multiple agent sessions (caught again 2026-09-08 on a mount-plate
obround built as 3 circles). Treat it as a gate, not a preference: check `sketch.FullyConstrained`
on every sketch before padding/pocketing, and if you catch yourself stacking primitives, stop
and rebuild it as a sketch.

**For a sheet-metal part with corner/direction-change bends (a ring, a mitered frame, anything that turns a corner, not just a straight flange fold) — build the final 3D SOLID first, then Unfold. Do NOT build the flat pattern and try to Fold it up (Erik, 2026-09-12, standing rule).**

An entire session (2026-09-12) was lost trying to make `SMFoldWall` bend a flat hexagon-ring
strip at its 5 internal corners — NULL shapes, garbage hundred-mm out-of-plane sweeps,
self-intersecting slivers, no matter the bend-line geometry tried. Root cause: `Fold`
fundamentally rotates material out of the plane of whatever reference face it's given: it
can produce a flange bend (out-of-plane, one direction change) but cannot produce an
in-plane direction change (a true corner turn), because that needs a rotation about an axis
perpendicular to the sheet, which `Fold`'s "bend line lies in the reference face" model
can't express.
**The correct, SheetMetal-documented workflow for this shape family is the reverse
direction:**
1. Sketch the part's own cross-section (or, for a simple case, a solid envelope shape) and
   Pad it to a real solid — build the FINAL 3D geometry directly, corners and all, not a
   flat strip meant to be bent up later.
2. `PartDesign::Thickness` (Shell) to hollow it to material thickness, selecting the faces
   that should be OPEN (removed), leaving the rest as the shell.
3. (Superseded for KinSculpter 2026-10-01: split corner-turning parts into straight single-fold pieces and build each as a flat plate plus Add Wall - see the 2026-10-01 entry. Not `Make Bend`.)
4. `Make Relief` / `Make Junction` at corners where 3+ faces meet, so the geometry isn't
   "closed" at those points (unfold will fail otherwise).
5. `Unfold`, selecting a reference face and K-factor - this computes the correct flat
   pattern and notch geometry, rather than needing it hand-derived (which is what burned
   the whole prior session: hand-deriving miter-notch trig that kept needing correction
   against folded evidence, when the tool can just compute it correctly from a known-good
   solid).
Proven on a disposable test part first (`HexTube_Study`, 2026-09-12) before touching the
real ring - build the solid shape (annular hexagon, Pad + `PartDesign::Thickness`) and
confirm the cross-section is right before committing to bends/`Unfold` on production
geometry. Full session narrative: `01 - Daily Notes/2026-09-12.md` Session 4.

**Use the right workbench for the job — each serves its proper purpose (Erik, 2026-09-08).**

- **Sheet-metal parts → the Sheet Metal Workbench**, always. Base flange from a sketched
  profile, real Bend / Wall features with bend radius + k-factor + reliefs, and Unfold for
  the flat pattern. Do NOT fake a bent part with a pad + chamfers, mitred boxes, or rotated
  solids — it won't unfold, the developed length is wrong, and it isn't editable. If SMWB
  isn't installed, stop and get it installed (see the live-addon-install note below), don't
  work around it.
- **Solid parts → PartDesign + Sketcher** (per the hard rule above). Not loose `Part`
  primitives.
- **Assemblies → real joints** (Assembly WB / `assembly` joints), not `App::Link` placement
  numbers typed in by hand. A locked `App::Link` is acceptable only as an explicit provisional
  placeholder that is called out as such in the report.
- **Drawings → TechDraw.** Patterns → the pattern features (`linear_pattern`, `polar_pattern`,
  `mirrored_feature`), not copy-paste geometry.

Picking the wrong workbench to save a step produces geometry that looks right in a screenshot
and is unusable for manufacture or downstream edits.

## Current Known Gotchas
- **Bridge API is not neka-nat's full set**: the XML-RPC server on :9875 only exposes
  `execute`, `get_view`, `ping`, `get_instance_id`. There is NO `get_report` /
  `get_console_output` RPC — read the Report View by running Python via `execute`.
- **`execute` return shape**: `{'success', 'result', 'stdout', 'stderr', 'execution_time_ms'}`.
  `print(...)` output lands in `stdout`; the code's `result` is usually None.
- **`get_view` view_type is case-sensitive** and must be one of exactly:
  `Back, Bottom, FitAll, Front, Isometric, Left, Right, Top`. Lowercase `iso`/`isometric` errors.
- **`get_view` returns base64 under key `data`** (not `image`), alongside `format/width/height`.
- **`get_view` takes positional args `(width, height, view_type)`** — `get_view("Isometric")`
  fails trying to `int()` the string as a width. Always `get_view(800, 600, "Isometric")`.
- **Report View dock** is a `QDockWidget` named `"Report view"`; its text is in a child
  `QTextEdit` (`.toPlainText()`). It may be hidden — `setVisible(True)` to show it.
- **`hideGrid()` is not a real API**: the Draft grid is off by default on a fresh doc. To
  suppress it set Draft params `grid=False` / `alwaysShowGrid=False`.

- **Helical groove/thread sweep balloons on the analytic helix**: sweeping a small
  profile along `Part.makeHelix(...)` with `makePipeShell(..., frenet=True/False)` or
  `makePipe` produces a tube that spirals OUTWARD (bbox grows with turn count — a Ø48
  drum groove ballooned to Ø77), then `cut()` yields an invalid, 0-volume, multi-solid
  result. **Fix that works on 1.1.1**: discretize the helix into points, `interpolate`
  a `Part.BSplineCurve`, and `makePipeShell([circle_profile], True, True)` along that
  B-spline wire. Gives a valid single-solid groove that cuts cleanly. (~48 samples/turn.)
- **Always sanity-check booleans**: after fuse/cut, assert `shape.isValid()` and
  `len(shape.Solids)==1`; a negative `Volume` or >1 solid = fragmented/non-manifold.
- **`get_view` renders the CURRENT camera, not the `view_type` arg** — to get a real
  front/top/side you must set it first via `execute` (`v.viewFront()/viewTop()/viewRight()`
  + `Gui.SendMsgToActiveView('ViewFit')` + `Gui.updateGui()`), THEN `get_view`.
- **Blank / odd-angle screenshots**: `ViewFit` + `get_view` occasionally returns a black
  or bottom-up frame (timing, or parts spread far apart). Add `Gui.updateGui()` and
  re-shoot; hide far-flung dummies so `ViewFit` frames the target.
- **Establish front/back definitively** before spatial edits: drop small colored marker
  cubes at +X/+Y/−Y and read `v.getViewDirection()` — don't argue orientation from memory.
- **Sketch won't extrude / "not a closed wire"**: check `sk.solve()` (−3 = conflicting
  constraints) and count `Coincident` constraints vs corners. Corners can sit at matching
  coords yet be sub-tolerance-open when positioned by *dimension* constraints instead of
  endpoint coincidences. Robust fix: capture the corner points, `delGeometries` the profile
  lines, re-add them as a clean closed loop with a `Coincident` at every corner, recompute,
  verify `Part.Face(sh.Wires[0])` builds.
- **Bridge/FreeCAD can crash mid-edit** (seen during `delGeometries`/`addGeometry` on a
  sketch, and one XML-RPC drop). `execute` may report success but the crash loses it —
  `doc.save()` often and **verify with a fresh read-back** after important edits. If both
  :9875 and :9876 refuse, the whole MCP Bridge died — the user must restart it from the
  Robust MCP Bridge workbench (GUI stays up).
- **Can't put a `Part::Feature` into a PartDesign `Body` programmatically**: `body.addObject`
  → "object is not allowed"; setting `body.BaseFeature` alone → null shape. The GUI
  drag-into-body (which reparents/scopes it) is the only reliable path — instruct the user.
- **`Shape.makeChamfer(size, edges)` is symmetric only** (single distance). A chamfer on a
  *vertical* edge stays vertical — it can't tilt in 3D to trace a sloped flank. For an
  asymmetric bevel, cut a triangular prism (two distances) yourself; for a truly tilted
  face, cut with a rotated box, not a chamfer.
- **Parametric rebuild pattern that preserves user edits**: keep the housing SHELL and the
  user-edited feature (e.g. a buttress `Cut`) as separate objects; to change a global param
  (wall thickness), rebuild only the shell from params and `fuse` the user's `Cut.Shape`
  back in, then assign to the existing object's `.Shape` (don't remove/re-add — keeps
  downstream links intact).
- **Never call `Gui.showPreferences(...)` (or open any other modal dialog) via `execute`**:
  it blocks the FreeCAD GUI thread until a human closes it. `ping()` still answers (runs off
  the GUI thread) but every subsequent `execute()` call hangs for the full 30s timeout until
  the dialog is closed — the user has to close it manually. If you need a preference, read/
  write it directly via `App.ParamGet(...)`, don't open the dialog to inspect it.
- **`PartDesign::FeatureBase.BaseFeature` (e.g. `BaseFeature` importing an external
  `Part::Feature` like `Housing` into a `Body`) is a ONE-TIME bake, not a live link.**
  Editing the source object's `.Shape` afterward does NOT propagate — `doc.recompute()`
  silently no-ops on it. To reposition/transform the imported base, set
  `BaseFeature.Placement` directly (a real tracked property); downstream face-referenced
  features (`Pocket.Profile = (obj, ['FaceNN'])`) recompute correctly off of that.
- **To shift a Body's local origin without moving it in world space** (e.g. so
  `Body.Placement`'s pivot lands on a real datum like a mounting face, for correct GUI
  rotation/assembly behavior): set `BaseFeature.Placement.Position = -P`, then
  `Body.Placement.Position = +P`. Verify by diffing `Body.Shape.BoundBox`/`.Volume` before
  and after — should be identical. Works cleanly only when downstream features are
  face-referenced (topological), not fixed-coordinate sketches.
- **FreeCAD 1.1.1's 3D-view orbit pivot is `View/RotationMode`** (`User parameter:BaseApp/
  Preferences/View`, int, global not per-doc): `0` = Window center (default — orbits around
  the 3D view's screen-space center, which is visually off-center whenever side panels are
  open), `1` = Drag at cursor (orbits around whatever's under the mouse when the drag
  starts), `2` = Object center. This is completely separate from any object's `Placement` —
  changing a Body's Placement origin has zero effect on mouse-orbit behavior in the viewport.
  Confirmed via `strings` on `libFreeCADGui.so` (labels: "Window center", "Drag at cursor",
  "Object center", combobox index == stored int), not from memory.

- **MCP server registration must be USER scope, not project/local scope.** `claude mcp add`
  defaults to `local` scope, keyed to the Claude Code session's project root (NOT the shell's
  `pwd` — these can diverge, e.g. after `/cd`, and `mcp add` follows the session root). A
  local-scope entry only appears in that one exact project path's tool list. Register with
  `claude mcp add freecad -s user -e FREECAD_MODE=xmlrpc -- freecad-mcp` so it's always
  available everywhere. Verify via `python3 -c "import json; print(json.load(open('/home/erik/.claude.json'))['mcpServers'])"`
  — should show top-level `mcpServers`, not nested under `projects.<path>`.
- **New/changed MCP server registrations require a full session restart** to take effect —
  `/mcp` reconnect alone does not pick up config file changes; the running process loaded
  its server list at startup. The FreeCAD XML-RPC bridge being "live" (port 9875 answering)
  is a separate fact from this Claude Code session having the `freecad` MCP tools connected.

- **When the agent has no `mcp__freecad__*` tool bindings**, drive the live GUI instance
  directly over the raw XML-RPC bridge on :9875 from a shell (`xmlrpc.client.ServerProxy`,
  call `.execute(code)`). This is the same transport the MCP tool layer sits on top of —
  it is still the live GUI Erik is watching, not headless `freecadcmd`. Confirmed working
  end to end (PartDesign body build, expression binding, screenshots via `get_view`, STL
  export) on a session missing the tool bindings; flag the missing bindings for whoever
  owns agent tooling rather than falling back to headless scripting.
- **`Gui.showMainWindow()` does not reliably set up `Gui.ActiveDocument`/`ActiveView`** for
  a document created via `App.newDocument()` through `execute()`. Use
  `Gui.getDocument(doc.Name)` explicitly, assign it to `Gui.ActiveDocument`, then pull
  `.ActiveView` off that — `Gui.ActiveDocument` alone after `showMainWindow()` silently
  stays wrong and the next `execute()` call reports `success=False` with an empty stderr
  (the real exception is swallowed; check the Report View text via
  `mw.findChild(QtWidgets.QDockWidget, "Report view").findChild(QtWidgets.QTextEdit).toPlainText()`
  — import `QtWidgets` from `PySide6` first, `PySide2` is not installed on this FreeCAD).
- **`execute()` reporting `success=True` with only partial stdout, and a downstream
  property silently not updated, means a mid-script exception was swallowed** — this
  bridge does not surface exceptions raised after the first few print statements the same
  way as a clean top-level failure. Wrap anything doing `setExpression` on a property path
  you're not 100% sure of in `try/except` with `traceback.print_exc()` and re-check
  `stderr`, or verify the result with a fresh read-back (`obj.ExpressionEngine`) rather
  than trusting the truncated stdout.
- **`Placement`'s position sub-property for `setExpression` is named `Base`, not
  `Position`.** `obj.setExpression("AttachmentOffset.Position.z", ...)` raises
  `AttributeError: No attribute named 'Position'` and, if the exception is swallowed (see
  above), leaves the property with whatever numeric value was last assigned directly and
  an empty `ExpressionEngine` — looks bound, isn't. Use
  `obj.setExpression("AttachmentOffset.Base.z", "<<VarSet>>.Param")`.
- **A revolved tube with a hole (bore) doesn't need a separate Pocket**: draw the
  PartDesign::Revolution profile sketch so its axis-side boundary sits at the bore radius
  instead of at r=0 (i.e. a plain closed polygon offset from the axis, not touching it).
  Revolving that single wire about a `ReferenceAxis=(sketch, ["V_Axis"])` axis produces a
  hollow tube directly — simpler and more robust than revolving a solid disc and cutting
  the bore afterward.
- **A non-axisymmetric feature on an otherwise-round bore (e.g. a D-shaft flat) is an
  ADDITIVE Pad, not a subtractive Pocket.** The flat narrows the hole (adds material back
  in), so pocketing a chord into a full-radius bore is backwards — instead revolve the
  bore at its full round radius, then Pad a circular-segment "cap" shape (short arc +
  chord, both dimensioned from the sketch origin/named and expression-bound) into the void
  for the sketch's full length. Build the arc+line via `addGeometry` with computed
  numeric coordinates first, solve with plain numeric constraint values, confirm
  `solve()==0`, THEN convert each numeric constraint to an expression one at a time and
  re-solve — binding radius/position expressions on a still-being-built arc+line pair
  straight away, before the geometry is confirmed closed and consistent, produced
  `solve()==-3` (`ConflictingConstraints` listing every constraint) even though the same
  constraint set with plain numbers solved instantly and the expressions evaluated to the
  identical numbers. Build-then-bind avoided it.
- **A grub-screw / set-screw pilot hole needs a genuine local BOSS, not just "somewhere in
  the solid".** A large-diameter part that's solid all the way from its bore to its OD
  (e.g. a wide flanged drum) will happily let a radial Pocket tunnel the entire radius —
  it recomputes clean and passes `isValid()`/`Solids==1`, but the resulting screw hole is
  as deep as the part is wide (tens of mm), which no real grub screw can reach. Model an
  actual raised boss (small-diameter, few-mm-tall Pad) around the bore first, and put the
  grub screw's datum plane / pocket depth on the BOSS radius, not the part's outer radius.
  This is a design-intent bug, not a FreeCAD error, so nothing in the recompute or the
  validity checks catches it — only reasoning about the real screw length (or a probe with
  `shape.isInside()` along the hole's own axis at a few stations) does.
- **`shape.isInside(point, tol, checkFace)` sampled exactly on a feature's own axis of
  symmetry tells you almost nothing** — every point along a hole's centerline reads
  `False` (empty) by construction, including at radii where the surrounding material is
  perfectly fine. Offset the probe point off-axis (e.g. by more than the hole's own
  radius, in a direction with no other cut) to actually test bulk material presence.

## Verified Workflows
- **Connection**: XML-RPC on localhost:9875. Confirm liveness with `ping()` (returns
  `pong`, `timestamp`, `instance_id`).
- **Clean view sequence** (via `execute`): `Gui.activeDocument().activeView().viewIsometric()`
  then `Gui.SendMsgToActiveView("ViewFit")`; camera reports `Orthographic`.
- **Version**: Detected FreeCAD 1.1.1 on CachyOS. Also confirmed on 1.1.3
  (`44987 (Git)`, 2026-08-31 session) — same XML-RPC surface and gotchas apply.
- **Active project**: `kinsculpter` document (dir: stedza/cad-projects/kinsculpter).
- **XZ_Plane / YZ_Plane origin-plane local-axis mapping** (verified by placing a sketch
  and reading `Placement.multVec` on unit vectors, not assumed): sketch on `XY_Plane` ->
  local X=global X, local Y=global Y (identity). Sketch on `XZ_Plane` -> local X=global X,
  local Y=global Z. Sketch on a datum plane built off `YZ_Plane` -> local X(U)=global Y,
  local Y(V)=global Z, and the datum's own normal is global X (so an `AttachmentOffset`
  along that datum's local Z moves it along global X). `body.Origin.OriginFeatures` list
  order is `[X_Axis, Y_Axis, Z_Axis, XY_Plane, XZ_Plane, YZ_Plane, Origin]` — index by
  this order, don't guess.

## PartDesign/Sketcher fundamentals (distilled 2026-08-31 from the bridge's own docs)
*Prompted by a session that fought raw `execute_python` Part/Sketcher scripting for hours on
a spoke-wheel cut-and-pattern feature instead of using the bridge's purpose-built tools —
read this section before hand-rolling `Sketcher.Constraint`/`Part.ArcOfCircle` calls again.*

- **Prefer the structured `mcp__freecad__*` tools over raw `execute_python` scripting for
  standard operations.** `add_sketch_line`, `add_sketch_arc`, `add_sketch_circle`,
  `add_sketch_rectangle`, `pad_sketch`, `pocket_sketch`, `polar_pattern`, `linear_pattern`,
  `mirrored_feature`, `fillet_edges`, `chamfer_edges` are real, tested wrappers — not a
  reduced subset of what raw scripting can do. `pocket_sketch(sketch_name, length, type)`
  notably has **no `Direction`/`reversed` fiddling exposed at all** (unlike `pad_sketch`,
  which has a plain `reversed: bool`) — it's designed to just cut into the existing body
  correctly, sidestepping the exact Direction/Reversed guessing game that ate significant
  time in the 2026-08-31 session (a raw-scripted `PartDesign::Pocket` defaults to `Type=
  ThroughAll`, `Direction=(0,0,-1)` regardless of the sketch's actual orientation, and
  silently cuts nothing if that direction misses the material — no error, just a no-op that
  looks like success until you check the volume).
- **A sketch should be FULLY CONSTRAINED before padding/pocketing it.** The bridge's own
  `USER_GUIDE.md` troubleshooting section says so explicitly: "Sketches need to be fully
  constrained for PartDesign operations." A raw-scripted sketch built by placing geometry at
  literal computed coordinates (e.g. via `Part.ArcOfCircle`/`Part.LineSegment` with numeric
  `App.Vector`s, then only adding `Coincident`+`Radius` constraints) is geometrically correct
  but NOT necessarily fully constrained in Sketcher's sense — check `sketch.FullyConstrained`
  and treat `False` as a real warning sign before feeding it to Pad/Pocket, not a cosmetic
  nag. (Cross-reference the existing "Sketch won't extrude" gotcha above, which independently
  found the same class of problem from a different angle — sub-tolerance-open wires from
  dimension-constrained corners instead of coincidence.)
- **The bridge's own docs describe a considerably larger tool surface (~150 tools per
  `guide/tools.md`, including a whole Validation category: `validate_object`,
  `validate_document`, `undo_if_invalid`, `safe_execute`, plus `sketcher_array_polar`,
  `sketcher_add_constraint_*`, Spreadsheet tools) than what the "freecad" Claude Code agent
  actually has bound (~65 tools, confirmed 2026-08-31 by diffing the agent's tool list
  against `MCP_TOOLS_REFERENCE.md`'s actual table of contents — Validation and the granular
  Sketcher constraint/array tools are absent from both the agent's binding AND
  `MCP_TOOLS_REFERENCE.md` itself, despite being advertised in `guide/tools.md`).** If a tool
  named in the docs isn't available, that's a real capability gap worth flagging to whoever
  owns agent tooling, not a sign it doesn't exist in the bridge — `undo_if_invalid` in
  particular is exactly the "validate and auto-recover" pattern that would have caught the
  null-shape Pocket failures immediately instead of requiring manual `.isValid()` checks
  after every single `execute_python` call.
- **Idiomatic order for a repeated-cut feature (e.g. a spoke wheel):** `create_partdesign_body`
  → `create_sketch` on a plane → draw ONE cut profile with `add_sketch_line`/`add_sketch_arc`
  → fully constrain it → `pocket_sketch` → `polar_pattern(feature_name=<the pocket>,
  occurrences=N)`. This patterns the FEATURE (the dedicated, documented, most-idiomatic path)
  rather than pre-drawing N copies of the geometry inside one sketch by hand — the latter
  (computing each copy's rotated coordinates in Python and adding them all to one sketch)
  works and was used successfully in the 2026-08-31 session once `PartDesign::Pocket` itself
  was cooperating, but it's the harder-to-edit, non-standard path: `polar_pattern` on a
  single-gap feature keeps the source geometry as ONE small sketch, GUI-editable and
  visually simple, with the repetition handled by a dedicated feature instead of N
  interleaved wires in one sketch object.

## From "FreeCAD 1.0 Ultimate Beginners Crash Course" (Mango Jelly Solutions, watched/transcribed 2026-08-31)
*Mostly reinforces what's above (fully-constrained/green/0-DOF sketches, Body/Sketch/Pad/Pocket
order, attachment=FlatFace) — three independent sources agreeing is worth noting but not
worth re-writing. Only the genuinely new, high-value points below.*

- **A single open/stray edge inside an otherwise-fine multi-profile sketch breaks the WHOLE
  Pad/Pocket, not just that one profile.** Demonstrated directly: a sketch with a valid
  closed rectangle plus one extra unconnected line fails Pad with "wire not closed" for the
  entire operation — Part Design (unlike some other workbenches) refuses anything but a
  fully closed wire per profile, and a sketch can hold multiple independent closed profiles
  (each pads/pockets as its own volume) as long as *every* wire in it is genuinely closed.
  **Directly relevant to the still-unresolved 2026-08-31 Pocket null-shape entry below**: the
  6-gap `Sketch_SpokeGap` built by computing rotated coordinates in Python had `Coincident`
  constraints added per-wire, but was never checked wire-by-wire for closure after all six
  were in — if even one of the six had a sub-tolerance gap, this predicts exactly the
  document-wide (well, sketch-wide) Pocket failure that was seen, and would show up as
  `len(sk.Shape.Wires) == 6` but `not all(w.isClosed() for w in sk.Shape.Wires)` — check this
  specifically, it's cheap and wasn't tried.
- **A chamfer/fillet that exactly consumes the material at a nearby edge/vertex can fail or
  visibly "snap back" to an untouched state right at the boundary size, not just error out.**
  Shown live: chamfering an edge at a size that reaches down to a perpendicular face works
  fine up to some threshold, then at the exact boundary (their example: a 20mm chamfer where
  20mm was exactly the remaining edge length) it silently reverts/fails, while backing off a
  hair (19.99mm) succeeds — "this is where you have to set this to something like 19.99, it's
  something to watch out for." **Same shape of failure as the 2026-08-31 session's pocket-edge
  chamfer wall at 0.35mm→0.4mm** (worked at 0.35, failed at every size from 0.4 up) — that was
  treated as unexplained OCC instability, but this is a named, expected FreeCAD behavior:
  approaching a geometric limit (here, likely the point where the chamfer would have to
  consume/eliminate an adjacent tiny face) rather than random solver flakiness. Worth
  systematically bisecting *down* from a known-good size in small steps next time instead of
  concluding "OCC is unstable here" from a handful of round-number attempts.
- **A Pad/Pocket that visibly "does nothing" (recomputes clean, no error, shape unchanged) is
  the single most common beginner tell for wrong `Reversed`/direction, called out explicitly
  in the video** ("it looks like nothing's happened — that's because the pad has travelled
  this way, it needs to go this way... click reverse direction"). Matches the 2026-08-31
  session's own first `Pocket_SpokeAnnulus` attempt exactly (`Type=ThroughAll` default
  direction cut nothing, silently, until `Reversed` was set) — worth having this as a named
  first-check rather than rediscovering it via trial and error each time.

## 2026-08-31 — Unresolved: PartDesign::Pocket fails document-wide with a null-shape error, cause still unconfirmed
Session spent several hours on a spoke-wheel redesign (`08 - Jan Ernst/KinSculpter/cad/
bench_capstan_drum.FCStd`) hitting `Part.OCCError: 19Standard_NullObject BRepCheck_Analyzer::
Init() - NULL shape` on **every** `PartDesign::Pocket` attempt against that file's Body —
regardless of profile shape (rectangle, circle, arc-capped gap wire), size, direction, or
`Reversed` setting — while the identical code succeeded immediately in a brand-new blank
document (bare Body, and again wrapped in an `App::Part`). Isolated by elimination, not
solved: ruled out geometry (raw `Part` module booleans on the identical shapes always
worked), ruled out session/state drift (persisted after a full `closeDocument`/`openDocument`
reload from disk), ruled out `App::Part` container wrapping. **Never tested: whether the
failing sketches were actually `FullyConstrained` before the Pocket call** (see the
fundamentals note above, added later while writing this entry) — every failing sketch in
that session was built via raw `execute_python` geometry placement with only partial
constraints (Coincident + Radius, not full dimensional closure), which is now the leading
suspect and should be the FIRST thing checked next time this recurs, before re-running the
whole elimination process. If `FullyConstrained` turns out true and it still fails, the
document-corruption theory stands and is worth reporting upstream with a minimal repro file.

**Update, same session, later: `FullyConstrained` checked and ruled out.** Rebuilt the
single-gap sketch with genuine full constraint (`Coincident` on both arc endpoints AND arc
centers pinned to origin, `Radius` on both arcs, `DistanceX`/`DistanceY` pinning two
non-adjacent corner points absolute — confirmed via `sk.solve()==0` AND
`sk.FullyConstrained==True`, not assumed) and reran `PartDesign::Pocket`: **identical
null-shape failure, both `Reversed` states.** Raw `Part.cut()` on that exact same
fully-constrained sketch still worked perfectly. This closes out the constraint hypothesis
cleanly — worth having checked, but not the cause. Root cause remains genuinely unknown;
still-untried diagnostics from the video-transcript section above (wire-by-wire closure
check, systematic chamfer-size bisection) remain the next things to try, not more variations
on constraint completeness. **Practical resolution used instead:** the reliable raw-`Part`
route (arc-capped profile geometry, computed in Python, `Part.Wire`→`Part.Face`→`.extrude()`→
`.cut()`) reproduces the correct geometry every time and was used to actually ship the
corrected part. Treat this as the working fallback whenever `PartDesign::Pocket` hits this
wall, not a failure to fall back on script — but remember to actually write the result back
to the object that gets exported (see below).

**A second, non-FreeCAD lesson from the same session, worth stating plainly: fixing the
geometry in a temporary/test object is not the same as fixing the part.** After diagnosing
and mathematically validating the corrected (arc-capped) spoke profile, every follow-up
attempt went into throwaway `PartDesign::Pocket`/`Sketch_*` test objects that failed and got
deleted — the actual exported `Part::Feature` (`SpokedDrum`) was never updated with the fix,
so the file on disk still held the ORIGINAL defective geometry for the rest of the session
until Erik looked at the real model and pointed out it hadn't changed. When a fix is
validated via a side experiment, the very next step must be applying it to the object that
actually gets saved/exported — not moving on to the next refinement request first.

## 2026-09-07 - Gearmotor envelope build (10_lib_gearmotor.FCStd), lessons
- **Stale scene-graph after a scripted PartDesign build renders as a garbage "spike"/pyramid.**
  After building a body via `execute_python`, `saveImage` produced a wildly wrong shape
  (box padded from an XZ sketch rendered as a tapered pyramid) while `Shape.BoundBox`,
  `.Volume`, face areas and `isValid()` were all perfectly correct. It is a Coin redraw
  bug, not geometry. **Fix that works:** toggle `body.ViewObject.Visibility` False/True,
  `d.recompute()`, then a loop of `Gui.updateGui()` + `QtWidgets.QApplication.processEvents()`
  (from PySide6) ~5x, then `view.fitAll()` + more processEvents, THEN `saveImage`. A blank
  white PNG from `saveImage` is the same class of problem - same fix.
- **Fully-constrained obround slot in a script: do not fight the arc+line+tangent solver.**
  Every combination of coincident/tangent/equal/vertical on 2 arcs + 2 lines either left
  1 continuous DoF or flagged redundant tangents (which also blocks `FullyConstrained`).
  What worked first try, `solve()==0` + `FullyConstrained==True`: build the slot as THREE
  trivially-constrained primitives - a circle at each end (Radius + DistanceX/Y on centre)
  plus a centred rectangle - and let two SEPARATE sequential `PartDesign::Pocket` features
  (one for the end circles, one for the rectangle) union into the obround. Overlapping
  profiles in ONE sketch/pocket fragmented the body into 3 invalid solids; the same
  profiles as separate pocket features cut cleanly.
- **Fully-constrained obround in a SINGLE 2D sketch (no pockets available): `Block` the
  cap arcs.** (2026-09-09, `30_lib_bracket` flat master, 5 slots - 4 keyholes + detent.)
  When the slot must live as real closed wires in one sketch (laser DXF, no PartDesign
  pocket to split it), the 3-primitive trick above does not apply. What worked first try,
  `solve()==0` + `FullyConstrained`, no redundancy: 2 arcs + 2 lines, coincident chain,
  `Equal` the two arcs, `Vertical`/`Coincident` to place one cap centre, `DistanceX/Y`
  (or `Symmetric` about a construction axis) for the other centre, `Radius` on one arc -
  then `Constraint('Block', arcGeoId)` on BOTH cap arcs. Block removes only the arcs'
  leftover angular DoF; it does NOT freeze world position, so the slot still tracks its
  centre/travel/width expressions on a param round-trip (verified: change `kh_travel`,
  arc centres and lines follow, stays `FullyConstrained`). Do NOT add `Tangent`
  line<->arc - that is the constraint that makes the solver report redundants and refuse
  `FullyConstrained`.
- **`Sketcher.fillet` overload dispatch is order-dependent and bites.** `sk.fillet(g1, g2,
  r, True, True)` (the `int,int,float` form) does NOT reliably fillet the corner shared by
  g1 and g2 - on a closed polygon it filleted the *wrong* vertex for some index pairs and
  raised "Not able to fillet point" for others. Use the point-reference overload:
  `sk.fillet(g1, g2, Vector_on_g1_near_corner, Vector_on_g2_near_corner, r, True, True)`
  (`int,int,Vector,Vector,float` - the only other accepted signature). Deterministic, and
  `createCorner=True` keeps the corner as a construction point with `PointOnObject`
  constraints so the tangents it adds don't strand the sketch. Note it also *deletes* some
  pre-existing constraints that touched the two edges (a `Symmetric` and a `DistanceY`
  each time in this session) - re-add them after all fillets and re-check `FullyConstrained`.
- **`App::PropertyLength` on a sketch constraint that a param drives with a two-operand
  expression needs BOTH operands qualified.** `sk.setExpression("Constraints.x",
  "<<doc>>#<<VS>>.a - b")` where `b` is meant to be `<<doc>>#<<VS>>.b` silently stores a
  broken expression (`b` resolves to nothing) - `setExpression` does not raise. Write
  `"<<doc>>#<<VS>>.a - <<doc>>#<<VS>>.b"`. `a / 2` (param over literal) is fine.
- **Cross-doc constraint expressions don't auto-fire on a param edit in the OTHER doc,
  same session.** After `vs.someParam = X; paramDoc.recompute()`, the consuming sketch's
  `getDatum` still returns the old value and `doc.recompute()` no-ops on it. Fix:
  `sketch.enforceRecompute()` then `doc.recompute()`. (On file reopen the documented
  `for o in doc.Objects: o.touch()` + recompute works instead.) The binding itself is
  correct - verify with the forced recompute, don't assume it's broken.
- **App::PropertyLength silently clamps negatives to 0.** A param that can go negative
  (an offset like `pilot_hole_offset_z`, `motor_center_x`) must be `App::PropertyDistance`.
- **`_result_` marshalling**: returning FreeCAD `Quantity` objects over the bridge throws
  `cannot marshal builtin_function_or_method`; the code's side effects still ran. Return
  `str(q)` or floats.
- Cross-doc expression binding by Label (`<<00_params>>#<<VS_Gearmotor>>.name` on
  `Constraints.<namedConstraint>` and on `Pad.Length`) worked cleanly; 57 expressions,
  survived close-all + reopen-from-disk with `for o in doc.Objects: o.touch()` then
  `recompute()`. Round-trip test (change `bolt_dx` 28->34) moved the hole pattern and kept
  every sketch fully constrained.
- **Features on the far side of a body (a rear stub, an opposite mounting face) go on an
  expression-driven `PartDesign::Plane` datum, not a face.** Recipe that works: datum
  `AttachmentSupport=[(YZ_Plane,'')]`, `MapMode='FlatFace'`,
  `AttachmentOffset=App.Placement(Vector(0,0,-depth),Rotation())`, then
  `dp.setExpression("AttachmentOffset.Base.z", "-(...params...)")`. Sketch on the datum
  (local u=global Y, v=global Z, normal +X same as YZ), pad with `Reversed=True` to go
  outward (-X). Verified in a round-trip: changing `reducer_box_depth` moved the datum and
  the entire rear shaft/spigot/flat stack with it, staying valid and fully constrained.
- **`App::PropertyLength` param that must accept a signed offset in an expression**
  (`bolt_pattern_offset_z`, `motor_center_x`, `pilot_hole_offset_z`) must be
  `App::PropertyDistance` - Length silently clamps to >= 0 and the expression then
  evaluates wrong with no error.

## 2026-09-08 - "blank grey face" / ghost-wireframe render after PartDesign tree surgery
Symptom: after inserting a feature mid-tree by hand (`body.insertObject`, or reassigning
`body.Group` and rewiring `BaseFeature` links), one or more views render as a flat grey
rectangle with no edges, or the tip solid shows as ghost wireframe while an earlier chunk
renders solid. `body.Shape` is a valid single solid the whole time; screenshots lie.
Cause: the hand-inserted feature was left with `Visibility = True` /
`ViewObject.Visibility = True`, so the Body VP was drawing *that intermediate feature's*
shape (an incomplete mid-tree body) instead of the tip. `body.insertObject` in particular
leaves the moved feature visible and can also leave a **duplicate entry in `body.Group`**
(same object listed twice) - check `len(grp) != len(set(names))`.
Fix: `for o in d.Objects: if o.TypeId.startswith(("PartDesign::","Sketcher::")) and o.Name!="Body": o.Visibility=False; o.ViewObject.Visibility=False`, then `body.Tip = <real tip>`,
`body.ViewObject.Visibility=True`, recompute. De-dup `body.Group` by reassigning the list.
Prefer wiring `BaseFeature` links directly + one clean `body.Group = [ordered unique list]`
over `insertObject`; `body.Group` *is* writable on a PartDesign Body in 1.1.3.
Also: `saveImage` dead-on axis views (viewLeft/viewRight along the part's long features)
are flaky - a three-quarter iso is more reliable, and close+reopen the doc from disk if
`viewIsometric` itself starts returning a non-standard `getViewDirection()`.

## 2026-09-08 - blank white render after hiding all PartDesign features
After `for o in doc.Objects: o.Visibility=False` to clean a view for rendering, setting
only `Body.ViewObject.Visibility=True` is NOT enough in 1.1.3 - `saveImage` comes back
pure white. The Body VP draws nothing unless the **tip feature itself** also has
`tip.Visibility=True` AND `tip.ViewObject.Visibility=True`. Fix: hide every
PartDesign/Sketcher object, then re-show just the tip (both flags) and the Body VP,
recompute, `updateGui()`+`processEvents()` x6, then saveImage. Also: FreeCAD's
`viewFront()` on a part whose long axis is X shows the housing side, not the mount face -
for an X-axis part, mounting-face-on is `viewRight()` (+X face) / `viewLeft()` (-X face).

## 2026-09-08 - expression on Placement.Base.x silently fails on a bare-number/Quantity subtraction
Setting `box.setExpression("Placement.Base.x", "-320 - <<VS>>.rib_gauge/2")` (a
dimensionless literal minus an mm `Quantity`) does NOT raise at `setExpression` time. It
fails at recompute: the object is left stuck `State = ['Touched','Invalid']` with the
sub-property value at 0, its shape stays whatever it was, and every downstream boolean
(`Part::Cut`, `Part::MultiFuse`, `Part::Common`) that consumes it comes back `isNull()` -
which then throws `Standard_NullObject BRepCheck_Analyzer::Init() - NULL shape` on any
`.isValid()` call. `doc.recompute()` reports a normal object count and never surfaces it.
Reproduced from a blank doc. `Length`/`Width`/`Height` expressions with the same
arithmetic work fine - it is specific to the `Placement.Base.*` path.
Fix: put a unit on every literal in a Placement expression - `-320 mm - <<VS>>.rib_gauge/2`.
A leading unary minus on a bare reference (`-<<VS>>.grid_half`) is fine; `0 - <<VS>>.grid_half`
is not. Once poisoned, the stuck box does not recover on `touch()`+`recompute()` or
`abortTransaction()`/`clearUndos()` - delete and rebuild it with corrected expressions.

## Sketcher constraint names collide with unit symbols
`sk.setExpression('Constraints.h', ...)` raises `Base.ParserError: Failed to parse
expression 'Constraints.h'` - `h` parses as *hours*, `rad` as radians, `s`/`m`/`A`/`N`
etc. likewise. Name driving constraints `obr_height`, `end_radius`, `pos_z` - never a bare
unit letter/word.

## PartDesign::Hole - threaded tapped hole with a param-driven bore (1.1.3, verified)
`h.Threaded = True` makes `h.Diameter` ReadOnly (thread-driven) *in the editor*, but
`h.setExpression('Diameter', '<<params>>#<<VS>>.tap_drill_dia')` still binds and wins - so
you get the "reads as tapped M4x0.7" callout AND a parametric Ø3.3 modelled bore.
`h.ThreadSize` is an enumeration: use the full string e.g. `'M4x0.7'`, `'M3x0.5'` (plain
`'M3'` errors). `HoleCutType='Countersink'` + `HoleCutCountersinkAngle=90` +
`setExpression('HoleCutDiameter', ...)` for a CSK screw seat. `DepthType='Dimension'`,
`DrillPoint='Flat'`. Sketch must contain **points**, not circles.

## 2026-09-08 - Installing an addon workbench live (no FreeCAD restart)
SheetMetal WB 0.8.22 installed on FreeCAD 1.1.3 by `git clone --depth 1
https://github.com/shaise/FreeCAD_SheetMetal.git SheetMetal` into
`~/.local/share/FreeCAD/v1-1/Mod/`. FreeCAD only scans `Mod/` at startup, so to register
it in the *running* GUI without a restart: exec the addon's `InitGui.py` with the right
names injected into the exec globals -
```python
import FreeCAD, FreeCADGui as Gui
from FreeCADGui import Workbench
g = dict(globals()); g.update(Workbench=Workbench, FreeCADGui=Gui, Gui=Gui, FreeCAD=FreeCAD)
exec(compile(open(p+"/InitGui.py").read(), p+"/InitGui.py","exec"), g)
```
`InitGui.py` bodies reference a bare `Workbench` and `Gui` that FreeCAD's normal startup
provides as builtins; without injecting them you get `NameError: name 'Workbench' is not
defined`. After exec, `Gui.listWorkbenches()` shows it (class name key, e.g. `SMWorkbench`)
and `Gui.activateWorkbench("SMWorkbench")` runs `Initialize()` and loads all commands.
It also loads normally on the next real restart since it is in `Mod/`.
- **SheetMetal scripting**: base+bend feature =
  `SheetMetalBaseCmd.SMBaseBend(obj, sketch)` on a `Part::FeaturePython`, then
  `SMBaseViewProvider(obj.ViewObject)`. Sketch = an **open** 2-segment polyline (foot +
  web); the corner becomes the bend (radius = `obj.Radius`). Props: `Thickness`, `Radius`,
  `Length` (extrude dist along sketch normal), `MidPlane`, `Reverse`, `BendSide`.
- **Unfold**: the new unfolder (`SheetMetalNewUnfolder`) imports `networkx` at module load
  - not present in FreeCAD 1.1.3's bundled Python 3.14, `NameError: name 'nx' is not
  defined`. Use the legacy one:
  `SheetMetalUnfolder.getUnfold({1: kfactor}, featurepy_obj, "FaceN", "din")` - returns
  `(flatSolid, foldLinesCompound, ...)`. Pick `FaceN` = the largest planar face whose
  normal is along the base plane normal.

## 2026-09-10 - SheetMetal U channel from a rectangle (KinSculpter 50_lib_frame_member)
- Recipe that worked first try, both flanges valid single solid: `Part::Extrusion` of a
  fully-constrained rectangle sketch on XZ_Plane (web), `Dir=(0,1,0)`, `LengthFwd` bound to
  `thickness` -> then `SMBendWall` on one long base edge, recompute + check, then a second
  `SMBendWall` on the opposite long edge taking Flange1 as its base (re-find the 150-long
  edge on `Flange1.Shape` by CenterOfMass - numbering shifts).
- **`BendType = "Material Outside"` was used for both walls without even trying "Material
  Inside"** (the 30_lib_bracket entry below documents Inside crashing OCC in this build).
  Outside is reliable. Consequence: each flange sits `(thickness+bend_radius)` outboard of
  its web edge, and the web keeps its full width in the flat pattern.
- **Unfold developed width does NOT match a hand-derived `strip_width` that assumed
  Material-Inside legs.** Here `SMUnfold` (legacy `SheetMetalUnfolder.getUnfold({1:k},
  flangeObj, "FaceN", "din")`) gave 106.2 mm vs a spreadsheet `strip_width` expr of
  84.0 mm - a 22.2 mm delta = ~2*(thickness+bend_radius) on the web + the flange offset,
  NOT a k-factor error (bend allowance 13.006 for k=0.38 was consistent). If the flat
  blank must be 84 mm, the strip_width formula has to be rebuilt for Material Outside, or
  the geometry needs true Material Inside (which crashes here).
- Expression-bindable SMBendWall props: `radius`, `length` took `setExpression` fine.
  `angle`, `kfactor` set as literals (0.38 / 90).
- Param round-trip (web_depth 45->60, flange_width 25->30, reset) tracked cleanly, stayed
  one valid solid, sketch stayed FullyConstrained. Cross-doc needs `for o in doc.Objects:
  o.touch()` then `doc.recompute()` after the param doc recompute.

## 2026-09-10 - SheetMetal WB 0.8.x fold tree on a closed sketch (KinSculpter 30_lib_bracket)
- **There is NO "base plate from a closed sketch" command in SheetMetal 0.8.x.**
  `SMBaseBend` (SheetMetalBaseCmd) needs an OPEN foot+web polyline to form a bend;
  `SMBaseShape` (SheetMetalBaseShapeCmd) is primitive L/U/Tub/Hat/Box only. For a flat
  sheet from an arbitrary closed profile, a plain `Part::Extrusion` (or PartDesign Pad) of
  the sketch IS the correct base - `SMBendWall` and `SheetMetalUnfolder.getUnfold` both
  work on it fine. A claim that "an extrusion won't unfold" is wrong; unfold failures are
  almost always bad flange geometry.
- **`SMBendWall` scripted:** `obj = doc.addObject("Part::FeaturePython","Bend")`;
  `SheetMetalCmd.SMBendWall(obj, baseObj, ["Edge9"])`;
  `SheetMetalCmd.SMViewProviderTree(obj.ViewObject)`. Then set `.length`, `.radius`,
  `.kfactor`, `.BendType`, `.reliefType`, `.reliefw`, `.reliefd`. Multiple edges in one
  feature (`["Edge9","Edge23"]`) was flaky (null shape) - chain one `SMBendWall` per edge,
  each taking the previous flange as its base, and re-find the edge index on the previous
  flange's `.Shape` (numbering changes).
- **`BendType = "Material Inside"` crashes OCC** here (`BRepOffsetAPI_MakeOffsetShape not
  done` -> null shape), intermittently - sometimes recomputes once then fails on the next
  edit. `"Material Outside"` is reliable; cost is the flange sits ~(radius+thk) outboard
  of the selected edge.
- **Round bend relief = semicircle:** `reliefType="Round"`, `reliefw = 2*r`, `reliefd = r`
  -> `(reliefD - reliefW/2) < eps` branch in `smMakeReliefFace` makes a pure arc notch of
  radius r at each bend end.
- **Scripted unfold works** despite the `networkx`/`nx` failure in `SheetMetalNewUnfolder`:
  use legacy `SheetMetalUnfolder.getUnfold({1: kfactor}, flangeFeaturePyObj, "FaceN",
  "din")` -> `(flatSolid, foldLinesCompound, ...)`. Pick `FaceN` = largest planar face
  whose normal is along the base-plane normal (here -X). Result is a STATIC shape, not a
  live feature - bake into `Part::Feature` and regenerate on param change.
- Expression-bind flange props to a VarSet: `f.setExpression("length",
  "<<00_params>>#<<VS_Bracket>>.flange_len")` etc. `reliefw` took `"<<...>>.flange_root_r * 2"`.
  Param round-trip (flange_len 18->24) propagated and stayed valid.

## 2026-09-11 - Trimming a SMBendWall fold to a narrow off-axis tab (KinSculpter 60_lib_trimplate foot)
Task: fold a flat plate's edge 90° (Material Outside), then keep only a narrow diagonal
strip of the resulting horizontal foot (containing 2 rotated slots) instead of the full
fold width, to fit inside a much narrower real part (a 25 mm sheet-metal flange).
- **A `Part::MultiCommon` (intersect) between the fold and a small "keep" box destroys
  everything outside that box, INCLUDING the connection back to the hinge/base plate** -
  it does not "trim a wing", it silently amputates the whole feature down to just the
  box overlap (single small box-shaped solid, correct volume for the box, wrong for the
  part). Confirmed by volume matching the box exactly.
- **Fix: build a removal TOOL, not a keep filter.** `removal = (envelope_box_over_the_
  fold_region) - (keep_shape)`; `result = fold.cut(removal)`. This only removes material
  that's both inside the fold's own region AND outside the keep shape, leaving everything
  else (the base plate, the bend fillet, anything the envelope box doesn't reach)
  untouched.
- **The envelope box must not reach into the region occupied by the UNRELATED base
  feature**, or the removal tool will eat into it too. Scope the envelope tightly to the
  folded region only (here: `X > 0`, well clear of the base plate's `X ∈ [-3,0]`), and if
  the keep shape must be a rotated diagonal strip that does NOT reach back to the hinge on
  its own, `fuse()` in a small axis-aligned "root" block at the hinge (full original fold
  width, short depth) into the keep shape first - a keep zone that doesn't touch the hinge
  produces a disconnected island (2 solids) even though every individual boolean call
  reports success and `isValid()==True`. Always check `len(shape.Solids)==1` after this
  kind of trim, not just validity - a 2-solid non-error result is the actual failure mode.
- **Obround/slot as a cut TOOL outside a PartDesign body (Part::Extrusion + Part::Cut,
  not Pocket): the "2 arcs + 2 tangent lines, Block the cap arcs" recipe from the
  2026-09-09 bracket entry is not required here and fought the solver again (redundant-
  constraint flags on the spacing DistanceX with `DoF==0` but `FullyConstrained==False`,
  cause not resolved).** Falling back to the OTHER documented recipe - **2 end circles +
  1 connecting rectangle, each in its own trivially-constrained sketch (`Radius`+
  `DistanceX`/`DistanceY` from origin only), extruded separately, then explicitly
  `Part::MultiFuse`d together** - solved cleanly first try (`solve()==0`,
  `FullyConstrained==True` on both sketches) and produced a valid 2-solid cut tool (one
  per slot) that cut a clean single-solid result. This recipe generalizes past the
  PartDesign-Pocket case it was first documented for: it's the reliable one whenever the
  arc-tangent-line obround fights the solver, sketched-in-a-Part-not-a-Body or not.
- **To get an in-plane rotation onto sketched geometry that must land at a world-fixed
  angle after the whole part is later rotated (e.g. a slot that must end up parallel to a
  fixed external member regardless of the part's own clocking): rotate the SKETCH's
  `Placement.Rotation`, not the geometry inside it.** Draw the slot/rectangle as a plain
  axis-aligned shape in sketch-local coordinates, then set
  `sk.Placement = App.Placement(Vector(anchor), Rotation(Vector(0,0,1), offset_angle))`.
  Keeps the in-sketch geometry trivial to constrain and puts the one number that actually
  varies per-instance (the offset angle) in exactly one place, expression-bindable.
- **Multiple sketches that must share one non-standard `Placement`+rotation (here: the
  trim-strip sketch and the 2 slot-tool sketches all sit on the same folded, rotated
  plane) each need `MapMode = "Deactivated"` and their OWN explicit
  `Placement`/`setExpression("Placement.Base.x", ...)` /
  `setExpression("Placement.Rotation.Angle", ...)` - there is no "attach to another
  sketch's plane" shortcut; duplicate the same two expressions on each one.**
- **Round-trip verified**: changing the driving off-square angle (34° -> 20° -> back to
  34°) via the VarSet, with `for o in doc.Objects: o.touch()` + `doc.recompute()` on the
  consuming document, kept the assembly at 1 valid solid and every sketch
  `FullyConstrained` throughout - confirms the whole trim/slot chain (2 boolean cuts + 1
  fuse + 3 rotated sketches) is genuinely parametric, not a one-shot script result.

## Screenshot workaround (2026-08-31): `get_screenshot` MCP tool is broken
The `mcp__freecad__get_screenshot` tool errors on every call in this session/bridge version:
`AttributeError: 'dict' object has no attribute '__name__'` (a bug in the bridge's own
wrapper code, not the caller's fault — confirmed by trying it repeatedly across totally
different documents/states, always the same traceback). **Working fallback via
`execute_python`:**
```python
Gui.ActiveDocument.ActiveView.viewIsometric()
Gui.SendMsgToActiveView("ViewFit")
Gui.updateGui()
Gui.ActiveDocument.ActiveView.saveImage("/tmp/some_path.png", 900, 700, "White")
```
Then read the PNG off disk directly (e.g. via the calling agent's file-read tool). Same
`export_stl`/`export_3mf` MCP tools also errored (`IndexError: list index out of range`)
in the same session — fall back to `obj.Shape.exportStl(path)` directly, and for 3MF/mesh
export use `MeshPart.meshFromShape(Shape=obj.Shape, LinearDeflection=0.1,
AngularDeflection=0.3)` then `.write(path)` rather than the bare default `Shape.tessellate()`
(which silently ignores its own tolerance argument on a Compound shape and returns
FreeCAD's cached high-fidelity display mesh — produced a 51.8MB/204k-facet STL for a 100mm
part at default settings, vs 585KB/11.7k facets with explicit `MeshPart` deflection).

## 2026-09-08 - Part::FeaturePython proxy: reload/rebind does not reliably re-fire execute() on recompute
Building a parametric model with custom `Part::FeaturePython` proxies (`RibStrip`, `StrapBar`,
`BracketSaddle` in `25_mount_frame.FCStd`). After `importlib.reload(lib)` + rebinding
`obj.Proxy = lib.Cls.__new__(lib.Cls)` + `obj.touch()`, `d.recompute()` returned a nonzero count
and set State to `Up-to-date` **without actually calling the new `execute()`** - the shape stayed
stale (a debug `App.Console.PrintMessage` in execute never printed). Some proxies in the SAME
recompute did re-fire (`RibStrip`, `StrapBar`), others (`BracketSaddle`) never did - no pattern
found. Calling `lib.Cls.execute(obj.Proxy, obj)` directly always worked.
- **Fix used:** for the objects that would not re-fire, dropped the FeaturePython entirely and
  rebuilt them as plain `Part::Feature` with geometry baked once, then drove position with native
  `setExpression("Placement.Base.x", ...)` / `.y`. Native Placement expressions on Part::Feature
  recompute 100% reliably (same as the expression-driven `Part::Box` notch-cutter boxes did).
- **Takeaway:** for a param that only needs to *move/scale* a fixed shape, prefer a native
  `Part::Feature` + Placement/dimension expressions over a FeaturePython proxy. Reserve
  FeaturePython for genuinely generative geometry (e.g. offset-a-spine-and-extrude ribs), and
  after any proxy rebind, verify with a fresh `obj.Shape.BoundBox` read-back, never trust the
  recompute return count.
- **`shape.Placement = App.Placement(...)` on a shape returned by `Part.makeBox`/`.fuse()` does
  NOT survive assignment to `obj.Shape`** in a FeaturePython execute - the transform is silently
  dropped. Bake it: `shape.translate(vec)` / `shape.rotate(...)`, or pass the base-vector arg to
  `Part.makeBox(l,w,h, base)`, or build the geometry from already-placed points.
- **`makeOffset2D` on a single straight edge** raises `wires are nonplanar or noncoplanar` - a lone
  line segment has no offset plane. Special-case straight spines to a plain oriented box.

## Multi-part clearance / interference sweeps (KinSculpter 12-up, 2026-09)

- **`Shape.distToShape()` on real B-rep of a 90+-face part is ~0.5-2 s per call.** A loop of
  ~80 combos x several pairs will run past 30 s and the MCP bridge drops the connection
  mid-call (then `list_documents` returns `[]` until FreeCAD is restarted). Keep any single
  `execute_python` under ~20 s: ~6-10 real `distToShape` calls max per call.
- **Working method that scales:** do the full parametric sweep analytically OUTSIDE FreeCAD
  on convex primitive envelopes (axis-aligned boxes + cylinders, plain arithmetic distances -
  box-box, box-cyl, cyl-cyl). Thousands of combos in a second. Then confirm ONLY the handful
  of binding pairs on real B-rep inside FreeCAD, at nominal + the specific worst offset the
  analytic pass found. On the 12-up job the convex-envelope result matched real `distToShape`
  within ~0.5 mm everywhere - the envelope method is trustworthy and conservative (it can
  over-report a touch near corners, never under-report).
- **Scaling a spoked/detailed drum to a new OD:** `shape.transformGeometry(Matrix())` with
  `A11=A22=k, A33=1` scales the two radial axes and keeps the axial length - a faithful
  real-B-rep part at the new diameter (groove, flanges, spokes all scale). Uniform
  `.scaled()` would also shrink the length.
- **`gv.saveImage(path, w, h, "Current")`** works when `get_screenshot` MCP tool throws
  `'dict' object has no attribute '__name__'` (bridge bug with multiple docs open). Call
  `App.setActiveDocument(name)` and use `Gui.getDocument(name).ActiveView` first.
- `viewTop()` + `SendMsgToActiveView("ViewFit")` still leaves a slightly 3D-looking result
  because tall parts show their sides; set `gv.setCameraType("Orthographic")` if a true flat
  plan is needed.

## Packing/arrangement optimisation outside FreeCAD (KinSculpter theta_z, 2026-09)

- When the task is "find the arrangement" (per-part rotation/position to avoid collisions),
  do NOT try to do it with Part.distToShape in a loop - a single restart of a 12-part
  optimiser is millions of distance calls and the heavy B-rep (a 100-face spoked drum is
  ~1-2 s per distToShape) makes it hopeless.
- Do it analytically: each part -> a convex plan polygon (rotated rectangle) + a Z-interval.
  Clearance = plan polygon-polygon distance (SAT overlap test, else min segment-segment)
  combined with the Z-gap. Cylinders on a horizontal axis -> a rectangle in plan + [-r,r]
  in Z. Add a bounding-circle pre-filter. This runs the full 66-pair eval in ~10 ms;
  random-restart greedy coordinate descent over ~20 candidate angles converges in seconds.
- Then B-rep verify ONLY the winning arrangement, and only the few binding pairs, and use a
  plain cylinder for the drum envelope (the flanges ARE a cylinder of the OD; the groove is
  smaller so a cylinder is slightly conservative) - keeps distToShape fast enough.
- The analytic result ran ~1-2 mm conservative vs the cylinder-drum B-rep here (24.3 vs
  22.9 mm min clearance) - trustworthy, erring safe.

## 2026-09-11 - Chamfer replacing a concave fillet at a reflex vertex (KinSculpter 60_lib_trimplate)
Erik reviewed a render of the fillet-relief at the trim-plate foot's two "root" reflex
corners (~326°/~304° interior, where the trimmed tab bridges back to the fold hinge) and
asked for a straight chamfer instead of the concave fillet - and, as a new standing
project convention (FREECAD_CONVENTIONS.md #11), for every other sharp corner on the
part's outer silhouette to be broken too, fillet or chamfer per corner.
- **`Shape.makeChamfer(size, edges)` tolerated a much bigger size than `makeFillet` at the
  same reflex-vertex edges.** The prior fillet on these edges failed above ~4.6mm radius
  (rolling-ball blend across a near-360° reflex has to construct a much larger blend
  surface). The chamfer on the identical edges stayed valid up to at least 6mm (tested
  1-6mm, all `isValid()` + single-solid). A chamfer is the geometrically simpler operation
  at a reflex vertex and is worth trying first if a fillet is failing or barely fitting.
- **Added a second proxy class, `EdgeChamferFeature`, to the same
  `~/.local/share/FreeCAD/v1-1/Macro/edge_fillet_feature.py` module as the existing
  `EdgeFilletFeature`** (same pattern: `Part::FeaturePython`, `Base` link + `EdgeNames`
  StringList + a single `App::PropertyLength Size`, expression-bindable, unlike native
  `Part::Chamfer`'s per-edge tuple property which does not accept `setExpression` either).
  `make_edge_chamfer(doc, base_obj, edge_names, size, name=...)` factory mirrors
  `make_edge_fillet`.
- **Finding "the other sharp corners" on a folded/trimmed sheet part reliably: extract the
  flat face's `OuterWire.OrderedEdges`, walk the ordered vertex list, and compute the
  signed turning angle at each vertex** (`atan2(cross, dot)` of the two adjacent edge
  vectors) rather than eyeballing a screenshot or trusting edge-count guesses. This
  distinguishes real 90°/reflex corners (`GeomLine`-`GeomLine` junctions) from tangent
  points where a flat face meets a curved bend-radius surface (`GeomLine`-`GeomCircle`
  boundary, G1-continuous, NOT a corner - confirmed by checking the two faces sharing that
  edge: one `Part::GeomCylinder` + one `Part::GeomPlane` = a real sheet-metal bend tangent
  line, leave it alone). On this part 2 of the "corners" the outline turns at (x=3, the
  edge of a root reinforcement block that also borders the bend fillet) were tangent
  points, not sharp corners, and were correctly excluded from the break-every-corner pass.
- **A corner's matching pair of top/bottom face vertices is bridged by one short
  through-thickness edge** (length == plate thickness, e.g. 3.0mm here) - fillet/chamfer
  that ONE edge per corner, not the two long face-boundary edges on either side of it. Find
  it the same way the original root-corner fillet's edges were re-verified: scan
  `Base.Shape.Edges` for `length ≈ plate_thk` whose both vertices sit at the target (x,y).
- **Chaining two `EdgeChamferFeature`s (root corners, then the other 5 silhouette
  corners) requires re-deriving the second feature's `EdgeNames` on the FIRST feature's
  OUTPUT shape, not the original Base** - edge indices renumber after any boolean-style
  shape rebuild (`makeChamfer`/`makeFillet` included). Do the by-coordinate edge scan
  against `first_feature.Shape`, not `Foot_trimmed.Shape`, when building the second link.
- **Blank-white `saveImage` root cause this session was NOT the display-mode/tip-visibility
  issue from the 2026-09-08 entries - it was the object's containing `App::Part` group
  having `Visibility = False`.** An `App::Part`'s own Visibility gates every child inside
  it regardless of the child's own `ViewObject.Visibility = True`. Symptom: `saveImage`
  with `"White"` background returns a genuinely blank single-color PNG (verified via
  `PIL.Image.getcolors()` - 1 color, exactly `(255,255,255)`); switching background to
  `"Current"` still showed nothing (just the viewport gradient), which is what proves it's
  a true empty-scene render and not a white-shape-on-white-background coincidence. Fix:
  walk `obj.InListRecursive` and check every ancestor's `ViewObject.Visibility`, not just
  the target object's own. Also needed in the same session: the MCP bridge's active
  document can silently be a DIFFERENT open document (`FreeCAD.ActiveDocument.Name` /
  `Gui.ActiveDocument.Document.Name` both showed the params doc, not the part being
  screenshotted, after a cross-doc expression bind switched focus) - set both
  `FreeCAD.setActiveDocument(name)` and `Gui.ActiveDocument = Gui.getDocument(name)`, AND
  bring that document's actual MDI subwindow to front
  (`mdiArea.setActiveSubWindow(sub); sub.activateWindow()`) before calling `viewIsometric()`
  / `fitAll()` / `saveImage` - a `View3DInventor` that isn't the frontmost tab can still
  report a plausible-looking camera from `getCamera()` while `fitAll()` computes a
  near-degenerate near/far clip pair (span of ~0.04 units) that clips the whole model away.
  `Gui.runCommand('Std_ViewIsometric')` (a real command dispatch) reliably reset the camera
  orientation where calling `.viewIsometric()` directly on a stale `ActiveView` reference
  silently no-op'd.

## 2026-09-12 (continued) - SheetMetal SMFoldWall does NOT auto-compose multiple folds off a shared base - each Fold must chain off the PREVIOUS fold's actual shape

Follow-up to the `invert`/`invertbend` entry above, same U-channel (ring flat strip, two
flange folds). `SMBendWall` is documented elsewhere in this file as safely reusable off a
shared base (`SMBendWall(obj1, pad, [...])` and `SMBendWall(obj2, pad, [...])`, "obj2's own
resulting shape correctly includes both flanges regardless - SheetMetal composes them").
**That claim does NOT hold for `SMFoldWall`.** Building `Fold_flange1` off `pad` and then
`Fold_flange2` ALSO off `pad` (same base, different bend-line sketch) produced two shapes
that looked like a real U-channel at a glance - both valid, both nearly identical volume -
but each only had its OWN flange folded; the other flange was still flat in each one. The
matching volume was a false signal: bending either single flange changes the volume by the
same amount, so equal volumes prove nothing about composition.
- **The fix: base the second Fold on the FIRST Fold's own object**, not on the shared flat
  pad - `SMFoldWall(fold2_obj, fold1_obj, ["FaceN"], sk2)`, re-finding `FaceN` fresh on
  `fold1_obj.Shape` (face numbering shifts after every fold, same as the general Fold
  face-selection gotcha above). This is the opposite pattern from `SMBendWall` - don't
  assume the two tools behave the same way just because they're both SheetMetal wall/fold
  operations.
- **How to actually verify composition, not just "isValid() and plausible bbox"**: check for
  a real second vertical wall face at the SECOND flange's expected (Y, Z) position - group
  the solid's faces by area/normal and confirm distinct wall faces exist at BOTH flange
  locations, both reaching a similar Z height. A bounding box or volume match alone can look
  "roughly right" while one flange is still flat - this is the same failure mode the
  `invertbend` entry above already flagged for a different reason (never trust a box/volume
  proxy once folds are involved; re-derive real faces/vertices at the expected coordinates).

## 2026-09-12 (continued) - SheetMetal has a dedicated "Add Corner Relief" tool - never tried it before hand-deriving relief geometry

After a long, costly session hand-deriving flange-notch/relief geometry in the flat sketch to
get a corner miter to close cleanly under `SMFoldWall` (mouth width, 30° angle, setback gaps,
floating relief circles - none of it made the fold compute reliably), Erik surfaced that the
SheetMetal Workbench ships a **dedicated "Add Corner Relief" tool**, separate from
`Fold`/`SMFoldWall` entirely. Also relevant and unused so far: **"Sketch On Sheet Metal"**
(punch a feature across an already-bent face, sketching directly on the folded geometry
rather than pre-computing where it lands in the flat pattern) and **"Add Base Shape"** (an
alternative to a plain PartDesign pad for the base flange). **Try the purpose-built tool
before hand-deriving flat-pattern geometry for a problem SheetMetal WB already has a command
for** - check the workbench's own toolset first, this session burned a large amount of time
re-deriving corner-relief trig from scratch when a dedicated tool may have handled it in one
step. Not yet verified working - next session should try `Add Corner Relief` directly on the
folded U-channel before returning to the flat-sketch relief-geometry approach.

## `sk.setDatum()` silently no-ops on an expression-bound constraint (2026-09-11, trimplate)

Calling `sketch.setDatum("name", value)` on a constraint that has a live entry in
`sketch.ExpressionEngine` **returns no error and the datum simply does not change** - the
recompute re-evaluates the expression and snaps the value straight back. Read
`sk.ExpressionEngine` (a list of `[path, expression_string]` pairs) BEFORE trying to edit
any named constraint; if it's listed there, the real driver is upstream (usually a VarSet
in a params document reached via `<<00_params>>#<<VS_Foo>>.param`), and that's what needs
editing, via `vs.param_name = value` + `doc.recompute()` + the stale-on-reload touch loop
(`for o in doc.Objects: o.touch(); doc.recompute()`) on the consumer document.

**Also check which OTHER sketches share the same expression before changing it.** On
`60_lib_trimplate.FCStd`, `Sk_foot_trim` (the folded-foot trim strip) and both slot
sketches (`Sk_slot_circles`, `Sk_slot_rects`) all bind `.Placement.Base.x` and
`.Placement.Rotation.Angle` to the SAME VarSet params (`foot_anchor_x`, `foot_offsquare`)
because they share one physical reference frame (the frame-member flange). Editing that
shared param to fix one sketch's geometry would have silently dragged the mounting-slot
positions along with it. The safe edit was a *different* param that only one of the three
sketches consumes (`foot_trim_len`, which only drives `Sk_foot_trim`'s own rectangle size
via a `trim_posx = -foot_trim_len/2` expression) - always check `ExpressionEngine` on
sibling sketches at the same Placement before touching a shared anchor parameter.

## 2026-09-11 - Re-angling one edge of a rotated rectangle to be world-axis-parallel (60_lib_trimplate, foot-root wedge fix)
Growing a rotated rectangle (uniform scale) can't close a wedge against a straight
reference line that ISN'T rotated with it - the two corners of a rotated edge are always
at different distances from a world-axis-aligned line, by a fixed `width*sin(angle)`
spread, no matter how big the rectangle gets. The fix has to change the EDGE'S ORIENTATION,
not just its size.
- **To make a sketch edge come out parallel to a world axis after the whole sketch's own
  `Placement.Rotation` is applied, give the two endpoints DIFFERENT local-axis
  coordinates, computed by inverting the placement rotation** - don't try to add an
  in-sketch angle constraint referencing the sketch's own Placement (self-reference risk).
  For a near edge that must sit at constant world-X = M, with the sketch rotated `theta`
  about Z and anchored at `(anchor_x, anchor_y)`: for each point's local Y (`ly`), solve
  `local_x = (M - anchor_x - ly*sin(theta)) / cos(theta)` (sign of the `ly` term flips
  between the two endpoints per the rotation direction - verify numerically against a
  known-good corner before trusting the formula, don't just trust the algebra). Both
  endpoints get their own named `DistanceX(origin, point)` constraint bound to this
  expression (with the VarSet params directly, not the sketch's own constraints, to avoid
  self-reference) - this correctly turns one straight edge into a "trapezoid instead of
  rectangle" reshape while leaving the other three edges (and anything anchored to them,
  e.g. mounting slots) completely alone.
- **`Sketcher.Constraint("DistanceX", g1, p1, g2, p2, value)` is NOT commutative for sign -
  swapping which point is First vs Second negates the value.** Got this backwards on the
  new edge's second endpoint (geom-point first, origin second, instead of matching the
  existing convention of origin-first/geom-point-second) and it silently placed the point
  at `+expected_value` instead of `-expected_value`, producing an obviously-wrong shape
  (a near-zero-length edge, one line collapsed onto the wrong corner) that only showed up
  by inspecting the actual solved `Geometry` coordinates after recompute. **Always match
  the argument order of an existing working constraint on the same sketch when adding a
  new one of the same type** (read one via `sk.Constraints[i].First/FirstPos/Second/
  SecondPos` first), and verify the solved geometry numerically, not just `solve()==0`
  (a wrong-sign constraint still solves cleanly, it just solves to the wrong shape).
- **A relative-length constraint (`DistanceX` between a shape's own two points) silently
  drags the "far" point along whenever the "near" point it's measured from moves** - the
  original sketch had `trim_len` = length of the bottom edge (relative, pos1-to-pos2), so
  moving the near-side pos1 outward to fix the margin also dragged the far/slot-carrying
  corner inward by the same amount, wrongly shifting the mounting-slot-aligned edge that
  was supposed to stay fixed. **Fix: replace the relative length constraint with an
  ABSOLUTE position constraint on the point that must stay fixed** (`DistanceX(origin,
  far_point) = <the same numeric anchor the old relative constraint produced>`), and let
  length become a free/derived quantity instead of a named driver. Same failure shape as
  the general "which edges actually move" question below - it's not enough to check that
  the point you're constraining moved correctly, check every point *indirectly* tied to it
  through a relative dimension too.
- **Root-cause verification for "is the wedge actually closed" needs the REAL solved
  Placement transform, not hand arithmetic** - `sk.Placement.multVec(local_point)` on the
  actual sketch object after recompute, checked against both endpoints, is cheap and
  authoritative; a screenshot at a tight top-down ortho zoom on the fold/root region
  (`gv.setCamera(<Inventor ASCII camera string with a small `height`>)`, not just
  `viewTop()+ViewFit`) is the other half of the check - the coordinate check alone would
  have missed if an upstream root-reinforcement-block feature (a separate always-kept
  full-width slab near the hinge, added by an earlier pass specifically so the keep-zone
  boolean wouldn't create a disconnected 2-solid island) already masked or interacted with
  the region being fixed. Baked `EdgeNames` on downstream chamfer FeaturePythons
  (`Foot_root_chamfer`, `Foot_silhouette_chamfer`) survived this shape-topology change
  intact and were re-verified by coordinate (midpoint) lookup, not assumed safe just
  because recompute reported no error - a chamfer whose baked edge index now points at
  the wrong edge still recomputes clean and valid, it just chamfers the wrong corner
  silently.

## 2026-09-11 (later) - flat sketch then ONE Fold - RETIRED as the build method

**Do not build sheet parts this way.** The project standard is `FREECAD_CONVENTIONS.md`
rule 16 (KinSculpter): flat laser blank first from a sketch, then Add Wall for each fold, then Unfold
(2026-10-01 entry). Shape toward a bend with a taper or blend,
never a hard step that leaves the bend without material on one side (rule 13). Still valid
from this section: corner breaks belong in sketches, and the `SheetMetalUnfolder`
ground-truth technique below.

The `60_lib_trimplate` foot was rebuilt from scratch this session after the bend-then-trim
approach (pad the flange full width, bend it with `SMBendWall`/"Extend", then cut it down to
shape with a *second* sketch living in the already-folded plane) kept producing subtle,
hard-to-see defects - a sheared outline, then a mis-centered reshape, each looking fine in
`solve()==0`/`isValid()==True` checks and only caught by measuring real coordinates or by
Erik's eye on the actual render. **Root cause: reasoning about geometry across a bend's
coordinate transform is exactly the kind of thing humans (and this agent) get subtly wrong.**
The fix isn't a better formula, it's a different workflow, and it is now the standing rule for
every future laser+brake part in this project (all 12 module trim plates, and the ring/spine
triangle U-channel members once their holes/joints are modelled):

**The whole part is ONE sketch, drawn entirely flat/unfolded** - outline (including any
tapered/rotated tab region), every hole, every slot, all in the same 2D sketch on the same
plane, at the size and position they have *before* any bend. Pad/Extrude that one sketch to
thickness. Then apply **one bend per fold line** using SheetMetal's `Fold` tool
(`SheetMetalFoldCmd.SMFoldWall`, GUI: Sheet Metal → Bend/Fold on an existing face along a
sketched line) - **not** `SMBendWall`/"Extend" (which builds a new wall off an edge and can't
carry a pre-shaped taper through the bend correctly). This mirrors "wysiwyg" sheet metal CAD
practice generally, not just a FreeCAD quirk: reasoning about a flat pattern is reliable;
reasoning about a solid across an implicit fold is not.

- **`Fold` needs a separate small reference sketch holding just the bend LINE** (a single
  `Part.LineSegment` on the same plane as the part, positioned where the crease should be -
  it does not need to be an edge of the actual part, the tool projects/intersects it), plus
  the flat solid and the face name that should stay stationary:
  ```python
  fold_obj = doc.addObject("Part::FeaturePython", "SomeFold")
  proxy = SheetMetalFoldCmd.SMFoldWall(fold_obj, flat_solid_obj, ["FaceN"], bendline_sketch)
  fold_obj.radius = 3.0; fold_obj.angle = 90.0; fold_obj.kfactor = 0.38
  fold_obj.Position = "intersection of planes"  # treats the sketched line as the true crease
  ```
  `"intersection of planes"` is the mode that matches "the sketched line IS where the bend
  goes" rather than offsetting from it - use it unless there's a specific reason not to.
- **Getting ground-truth flat-pattern coordinates for something that already exists
  post-bend**: don't hand-derive the unfold transform (this is exactly where the earlier
  sheared-outline and mis-centered bugs came from). Call the workbench's own unfolder:
  ```python
  import SheetMetalUnfolder
  SheetMetalUnfolder.KFACTORSTANDARD = "din"   # module GLOBAL, must be set before calling
  k_lookup = {0.1: 0.76, 10: 0.76}             # {threshold: value} dict, get_val_from_range
                                                 # format - an EMPTY dict silently returns
                                                 # None and crashes k_Factor's arithmetic
  flat_solid, *_ = SheetMetalUnfolder.getUnfold(k_lookup, feature_obj, "FaceN", "din")
  ```
  Pass the actual **document object** (not `.Shape`) as the third positional arg - passing a
  bare shape fails with `'Part.Compound' object has no attribute 'Name'`. The `k_factor_lookup`
  passed to `getUnfold` and the module-global `KFACTORSTANDARD` are two separate settings that
  both have to be right; the DIN standard halves the looked-up value (`k_factor_lookup` value
  of `0.76` → actual k-factor `0.38`), so pick the lookup value accordingly if trying to match
  an existing part's known k-factor. The returned flat solid's circle/edge coordinates are
  real, trustworthy flat-pattern positions - use those directly rather than re-deriving them.
- **Corner treatments (fillet/chamfer) belong in the flat sketch, before the bend, not as a
  3D `Part::Chamfer`/`Part::Fillet` on the folded result.** `Part::Chamfer`/`Part::Fillet` on
  an edge immediately adjacent to a bend's curved (cylindrical) face reliably threw `OCC
  Standard_NullObject: BRepCheck_Analyzer::Init() - NULL shape` in this session, **at every
  radius tried down to 0.5mm** - not a size problem, a topology-adjacent-to-curve problem.
  The exact same corners filleted cleanly in the flat sketch with zero issues. If a 3D
  edge-fillet/chamfer feature fails near a sheet-metal bend, don't fight it - move the
  treatment upstream into the sketch instead of shrinking the radius further.
- **Use the sketch's own `fillet()` method, don't hand-build arc+trim geometry** -
  `sk.fillet(geoId1, geoId2, refPnt1, refPnt2, radius, trim, createCorner)` produces the
  correct `Tangent`+`Coincident` constraint set automatically (matching what the GUI tool
  produces). **`refPnt1` and `refPnt2` must be two DISTINCT points, each located near/on its
  respective edge** - passing the same exact shared-corner point for both is ambiguous when
  one of the two edges has its OTHER end also near other trimmed/filleted geometry, and it
  silently trimmed the wrong segment/wrong end in this session (broke wire closure - `Part
  exception: Wire is not closed`). Fix: nudge each reference point a fraction along its own
  edge toward the corner (`p_on_edge = edge.StartPoint + (edge.EndPoint - edge.StartPoint) *
  0.9`, or `* 0.1` from the other end) rather than using the literal shared vertex for both.
  `mcp__freecad__undo` cleanly reverts a botched `fillet()` call if this happens.
- **Numerically-matching endpoints are not the same as constrained-coincident endpoints, and
  a sketch built by dropping raw coordinates (no constraints at all) is NOT safely editable
  later, even though it looks fine and computes fine at first.** Erik caught this directly:
  a fillet tool (or anything else touching the sketch afterward) has nothing to grab onto
  without a real `Coincident` constraint, and worse, once you DO add one, if there's nothing
  else pinning the geometry's size/shape (no `Radius`, no `Tangent`, no dimensional
  constraint), the solver is free to satisfy the new constraint by quietly moving whichever
  points it likes - which is exactly what happened to two slots in this session: adding
  blanket nearest-endpoint `Coincident` constraints across the whole sketch silently
  distorted both slots' shape, even though the topology (which edges connect to which) was
  correctly identified. **A slot/stadium built by hand needs `Tangent` (not just
  `Coincident`) at all 4 line-to-arc junctions, a `Radius` constraint locking the arc size,
  an `Equal` constraint tying the two end-arcs together, and `DistanceX`/`DistanceY` locking
  each arc center's absolute position** - that combination is what makes it robust to any
  later edit of the sketch, not just correct at the moment it's drawn. Building it fully
  constrained the first time costs a handful of extra `addConstraint` calls; retrofitting
  constraints onto raw coordinates after the fact risks silently corrupting the shape, and
  the only way to know it happened is checking the actual solved geometry, not `solve()==0`.
- **`sk.RedundantConstraints` / `.ConflictingConstraints` / `.MalformedConstraints` /
  `.PartiallyRedundantConstraints` are real-time diagnostic properties** (not methods -
  Python attribute access, no parens) - check these after any bulk constraint-adding pass,
  even when `solve()` returns 0. A `solve()` return of `-2` with no visible exception
  generally means redundant constraints exist; `sk.autoRemoveRedundants()` clears genuinely
  harmless redundancy (verify `solve()==0` and shape validity afterward) - don't leave a
  sketch sitting in a `RedundantConstraints`-flagged state, it's a sign something upstream
  wasn't built as cleanly as it could be, even if the shape currently happens to be right.

## 2026-09-11 (same session, root cause of a Fold `NULL shape` failure) - the bend-line sketch must extend PAST the face edge, not just match it

`SheetMetalFoldCmd`'s `Fold` threw `Part.OCCError: Standard_NullObject:
BRepCheck_Analyzer::Init() - NULL shape` on every attempt, consistently, across all four
`Position` modes (`intersection of planes`/`forward`/`backward`/`middle`) - looked at first
like a fundamental incompatibility between the tool and two small (1.5mm) sketch fillets
sitting on the part's outer corners, since removing them was the only thing that had been
tried. **The real cause was simpler and unrelated to the fillets: the bend-line sketch's
line ran exactly `(-31,-56)` to `(31,-56)`, matching the face's own edge length precisely
instead of extending past it.** Erik supplied the documented requirement directly: *"The
line must extend beyond the face edges; otherwise, the operation may fail or produce
warnings."* Extending the same line to `(-45,-56)`/`(45,-56)` (well past the ±31/±35 face
extent) fixed it immediately - the fillets were never the problem, and are now confirmed
present and correct in the final folded solid. **Always build the bend-line sketch with
clear margin past the face's actual boundary in both directions**, not just long enough to
span it - a line whose ends land exactly on (or short of) the face edge is a real, silent
failure mode, not a style preference. Worth checking first, before chasing anything else,
the next time `Fold` throws a NULL-shape error.

## 2026-09-11 (same session) - App::Part group membership can silently re-enable a child's visibility

Calling `part_group.addObject(some_object)` on an object whose `ViewObject.Visibility` was
already set `False` was observed to flip it back to visible, more than once, across several
different objects, in this session - re-parenting into the group apparently re-triggers a
default-visible state rather than preserving whatever was set before the reparent. **Set
final visibility AFTER every group-membership change, never before** - if grouping and
visibility both need doing, do the grouping first, then set every child's visibility state
explicitly as the last step, and re-verify it (`obj.Visibility`, not just "I set it earlier
in the script") right before saving. This also explains a chunk of the "I don't see the
part" back-and-forth earlier in this session - the visibility state kept looking like it was
reverting "on its own," and group-membership changes made mid-session were the actual cause,
not a rendering bug.

## 2026-09-11 (same session, later) - the actual project file architecture, and a live reparenting corruption to avoid

**Erik's stated architecture, load-bearing for every future session:** one master file
(the production assembly, `70_asm_production.FCStd`) containing one top-level `Assembly`
(`App::Part` with `Type="Assembly"`), which holds sub-assemblies, which are built from
individual **parts**. A "part" means a real `PartDesign::Body` (one manufactured thing),
never a bare `Part::Extrusion`/`Part::FeaturePython` sitting loose in a generic `App::Part`
(that reads as "a mini-assembly", not "one part" - this was the mistake caught this session:
built the trim plate's base pad as a loose `Part::Extrusion`, not a `Body`). Existing
separate library files (`10_lib_gearmotor.FCStd`, `30_lib_bracket.FCStd`,
`50_lib_frame_member.FCStd`, `60_lib_trimplate.FCStd`) **stay as they are** for now, still
individually usable/editable - migration into the one master file happens by rebuilding each
part properly *inside* the assembly, not by deleting or retiring the source files.

**The proven container pattern, from `50_lib_frame_member.FCStd` (the trusted reference):**
`Body` (`PartDesign::Body`) contains ONLY the base sketch + pad
(`Body.Group == [sketch, pad]`, `Body.Tip == pad`). SheetMetal bend/wall features
(`Flange1`/`Flange2` there, `Foot_folded` for the trim plate) are **separate objects, not
inside the Body**, chained onto the Body's Tip output - and the whole cluster (Body +
those sheet-metal features + their bend-line sketches) sits together inside one `App::Part`
container, which is what represents "this one part" to an assembly above it. So the real
nesting is: `Assembly` → `App::Part` (one part, e.g. "TrimPlate_h12") → `[Body, bend-line
sketch(es), fold/flange feature(s)]`, with the Body itself narrowly scoped to just its own
base sketch+pad.

**Reparenting an already-built `Body` into a new `App::Part` after the fact is destructive -
build the container first, then the Body inside it, never the other way round.** Tried
`asm.removeObject(body)` → create `App::Part` → `part_container.addObject(body)` +
`.addObject(other siblings)` on a Body that already had a working `Sk_flat_full`/`Pad` inside
it (`Body.Group` populated, `Body.Tip` set) - the result was silent corruption: `Body.Group`
came back empty, `Body.Tip` came back `None`, and a stray duplicate object
(`Foot_folded001`) appeared from nowhere, with no exception raised anywhere in the process to
flag it. This looks like `App::Part.addObject()` on a `PartDesign::Body` doesn't preserve the
Body's own internal group/tip state through the reparent - it silently detaches the Body's
children. **The fix, next time:** create the `App::Part` container FIRST, `partContainer.
newObject("PartDesign::Body", ...)` (or `addObject` immediately after creating an empty Body,
before adding anything to the Body itself) so the Body is born already inside its intended
parent, then build the sketch/pad/fold chain into that already-correctly-nested Body from
the start. Never move a populated Body between containers via script - verify
`body.Group`/`body.Tip` immediately after any `addObject`/`removeObject` touching a Body's
parentage, every time, since this corruption raised no error and was only caught by
explicitly checking those two properties afterward.

**Also observed, same rebuild:** a `PartDesign::Pad` does not respect a manually-set
`Direction`/`Reversed` the way `Part::Extrusion` does - it always extrudes along the
attached sketch's own local normal, full stop, in this FreeCAD version's scripting API
(no `UseCustomDirection` property exists to override it). Setting `MapMode='FlatFace'` with
an `AttachmentSupport` on a sketch **also silently overwrites any `Placement` set on it
beforehand** - the attachment computation replaces whatever Placement was there. Neither is
a bug worth fighting: attach the sketch to the plane that gives the orientation actually
wanted, and let the Pad's direction follow from that, rather than trying to force a
Placement/Direction combination against the grain. When copying a companion sketch (e.g. a
Fold's bend-line reference) alongside a re-attached main sketch, **give it the exact same
`AttachmentSupport`/`MapMode`, not a copied `Placement`** - two sketches meant to share a
plane must both get there by attachment, or they silently end up in different planes even
though both "look" like they have sensible Placement values individually.

## 2026-09-11 (same session, continued next turn) - the Body-corruption bug, actually diagnosed, and its real fix

Follow-up to the reparenting corruption above. Building the container in the right order
(`App::Part` created first, `body = part.newObject("PartDesign::Body", ...)` so the Body is
born already inside its parent) avoided the corruption **while only the Body's own sketch and
pad existed**. But calling **`part.addObject(some_other_object)` later - even for an unrelated
object, not the Body itself** - reproduced the same symptom: `Body.Group` came back empty,
`Body.Tip` came back `None`. This time it was fully diagnosed rather than just worked around:

- **The Body's children were not lost, just silently reparented.** `part.Group` after the
  "corruption" contained the Body's own sketch and pad directly (`[Body, sketch, pad, ...]`),
  not nested inside the Body anymore. Confirmed the underlying Shape data was never damaged -
  `pad.Shape.isValid()` and every downstream feature's Shape stayed completely correct and
  computable throughout; only the Group/Tip *bookkeeping* properties were wrong. **Always
  check whether "corruption" is real data loss or just a stale/wrong tree-organization
  pointer before treating it as a rebuild-from-scratch problem** - re-verify the actual
  `.Shape.isValid()` of the objects in question first.
- **Root cause: `App::Part.addObject()` on ANY object, when that Part already contains a
  `PartDesign::Body`, silently pulls the Body's own children up into the Part's group too.**
  Not specific to reparenting the Body itself - adding an unrelated sibling object triggers it.
- **The fix is mechanical and reliable once you know the pattern:** an object can only be in
  one `GeoFeatureGroup` at a time (`RuntimeError: Object can only be in a single
  GeoFeatureGroup` if you try to force it into two) - so after any `part.addObject(...)` call,
  explicitly (a) `part.removeObject(the_sketch)` and `part.removeObject(the_pad)` (or whatever
  got pulled up) to release them from the Part's group, then (b)
  `body.Group = [the_sketch, the_pad]` and `body.Tip = the_pad` to put them back where they
  belong. **`part.removeObject()` on one item can also silently drop OTHER unrelated items
  from `part.Group`** (observed: removing the sketch+pad also dropped an already-correctly-
  placed sibling sketch and the fold feature from `part.Group`, though again their Shapes
  stayed valid) - the reliable close-out move is to just **re-assert the intended final state
  of every affected Group in one explicit assignment** (`part.Group = [body, sketch2,
  fold_feature]`) rather than trying to trust incremental add/remove calls to leave the rest
  of the list alone. Verify `part.Group`, `body.Group`, and `body.Tip` all read back exactly
  as intended, in that order, as the last step before saving - every time a `Part`/`Body`
  hierarchy gets touched via script.

## 2026-09-11 (same session) - Fold's NULL-shape error can also mean "wrong face", not just "bad bend line"

Rebuilding the exact same h12 trim-plate sketch/pad/fold chain a second time (inside the new
`PartDesign::Body`-based container), the Fold threw the same `Standard_NullObject:
BRepCheck_Analyzer::Init() - NULL shape` that the bend-line-past-the-edge fix had already
resolved once this session - looked like a regression at first. **It wasn't the bend line
this time - it was which of the pad's two large flat faces got passed as the Fold's reference
face.** The two candidate faces (`Face29`/`Face30`, normals `(0,0,-1)`/`(0,0,+1)`) swapped
index and even swapped which-is-which between builds of what should have been identical
geometry (a `PartDesign::Pad`'s face numbering is not guaranteed stable/predictable run to
run the way one might assume). The face selected by "first Z-normal face found" failed
consistently; **the other large flat face worked immediately, first try.** If `Fold` gives a
NULL-shape error and the bend line already extends past the edge (confirmed correct), **try
the other large flat face on the base solid before assuming the sketch geometry itself is at
fault** - cheap to test (just swap the `selFaceNames` argument), and was the actual fix both
times this came up.

## 2026-09-11 (same session, building the second module h11) - object-name collisions across sibling parts in one multi-part document, and it corrupted an already-finished part

Starting the second trim-plate part (h11) in the same `70_asm_production.FCStd` that already
had h12's finished `TrimPlate_h12` (whose sketch is internally named `Sk_flat_full`), created
h11's new sketch the right way - `body.newObject("Sketcher::SketchObject", "Sk_flat_full")` -
and FreeCAD correctly auto-suffixed it to `Sk_flat_full001` to avoid the name collision, no
error raised. **The bug was entirely on this end**: a later line fetched the sketch back with
`sk = doc.getObject("Sk_flat_full")` (matching the pattern used all session for single-part
files) to apply the corner fillets - and in a document with more than one part, that name is
no longer unique, so it silently returned **h12's original, already-finished sketch** instead
of the new h11 one. The fillet loop then ran against h12's real 32-geometry shape (not a
plain 4-line rectangle), corrupting it (`Wire is not closed`, geometry count jumped to 36)
- **and this happened with no exception pointing at the actual mistake** - the error surfaced
downstream (a fillet failure) with nothing indicating the object was wrong, not just the
operation.

**In any document holding more than one part/module, never reference an object by a bare name
that another part in the same document might also use** (`Sk_flat_full`, `Pad_flat_full`,
`Foot_folded` are exactly the names every trim-plate module will want). Two real options,
both fine: (a) keep a direct Python handle to the object returned by `addObject`/`newObject`
and use that variable for the rest of the build instead of re-fetching by name at all - safest,
since a handle can't silently resolve to the wrong object; or (b) give every module's objects
a genuinely unique name from creation (`Sk_flat_full_h11`, not `Sk_flat_full`) so a same-session
`doc.getObject(name)` lookup is unambiguous even across many sibling parts. **Given this project
will have 12 near-identical trim-plate parts in one file, per-module unique names (b) is the
safer default going forward** - a bare `Sk_flat_full` lookup will keep silently finding whichever
module happened to claim that name first.

**Recovery, when this does happen: don't try to patch the corrupted object back - just verify
nothing was saved, close the document without saving, and reopen it fresh from disk.** Confirmed
clean in under a minute this way (`h12`'s sketch back to exactly 32 geometries, fold valid,
volume matching) - far faster and safer than trying to manually undo 4 bad fillets on a complex
32-geometry sketch. This is also why saving only at clear checkpoints (not after every small
step) matters: it's what made this instant, free recovery possible.

## 2026-09-11 (same session) - two real CAD-native techniques, and the index-tracking trap that cascaded when combining them carelessly

**Erik's teaching, directly applicable and now proven-correct in this sketcher API:**
`sk.trim(GeoId, refPoint)` and `sk.extend(GeoId, increment, PosId)` are real, callable tools.
The CAD-native way to connect two pieces of a sketch (e.g. a tab's edge reaching a plate
boundary) is **draw generously past where they need to meet, then trim/extend to the real
intersection** - not hand-compute the meeting coordinate and place a short separate connector
segment there. The latter is exactly what produced the "too little meat" pinched-wedge junction
earlier this session: a short connector landing near, but not exactly tangent-clean with, the
tab's own edge. Redone properly - `extend(tabEdgeGeoId, 15.0, 2)` to overshoot the edge past
the plate boundary, then rebuilding that one edge to run directly from its real far corner to
`PointOnObject` the plate's boundary edge (Parallel-constrained to the construction line) - gave
a single continuous edge with no separate connector and no pinch. **This is the right default
for any tab/flange edge meeting a boundary: one edge, extended/trimmed to the true intersection,
not a computed-coordinate stub.**

**`trim()`'s reference-point convention did not behave as expected** - passing a point on the
"far" portion of the curve (the side to be kept, near the far corner) resulted in the "far"
portion being discarded and the tiny overshoot sliver being kept instead - backwards from the
apparent rule. Not fully diagnosed (didn't chase it further after redoing the edge manually
worked cleanly) - **verify the kept-vs-discarded side immediately after every `trim()` call**
rather than assuming the reference-point convention, and prefer rebuilding a single clean edge
by hand (delete + fresh `LineSegment` + `PointOnObject`/`Coincident`/`Parallel`) over fighting
`trim()` if the first attempt goes the wrong way - re-diagnosing the tool's exact convention
costs more than just rebuilding the one edge correctly.

**The costly mistake: chaining multiple `delGeometries()` calls (or a delete followed by new
`addGeometry` calls) while continuing to reference OTHER geometry by integer index computed
BEFORE the delete.** `sk.delGeometries([N])` silently shifts every geometry index above `N`
down by one, with no warning - a later call using an index remembered from before the delete
(even a variable holding what was believed to be a stable GeoId, e.g. "geo6 is the far-left-to-
far-right tab edge") silently resolves to a DIFFERENT, wrong piece of geometry after the shift.
This cascaded badly this session: a second `delGeometries()` call (fixing the near-left
connector) shifted indices again while a stale index reference from the FIRST fix was still in
play, mis-linking a new edge to an unrelated point, and independently corrupted an already-built
slot into degenerate zero-length arcs (a `PointOnObject`/`Distance` constraint that had been
correctly targeting the slot's own arc center silently re-targeted a different geometry index
after the shift, and the solver "solved" it into a collapsed, valid-looking-but-wrong state with
no error raised). **After ANY `delGeometries()` call, never trust a previously-recorded index
for anything at or above the deleted index - re-fetch every subsequent target by its actual
current geometry/position (loop `sk.Geometry` and match by coordinates/type), every single time,
even for geometry that "shouldn't have been affected."** This is expensive to do every time, which
is exactly why deleting-and-reinserting mid-sketch is worth avoiding in the first place -
prefer the extend/trim approach (which modifies geometry in place, keeping its GeoId stable)
over delete-and-rebuild wherever the CAD-native tool can do the same job.

**Recovery, same pattern as before: don't try to untangle multi-step index-shift corruption by
more patching - verify nothing was saved, close without saving, reopen fresh, rebuild the
affected sketch from scratch with the lesson applied.** Confirmed clean again in under a minute;
the sibling part (`TrimPlate_h12`, fully saved and untouched by any of this) was unaffected.

## Slot (2-arc, 2-tangent-line) construction: use POINT tangent, not generic edge tangent + Coincident (2026-09-11)

Building a stadium/slot from a direct line+3-point-arc loop (per the "either circles+tangent+trim
OR direct line/arc construction, never both" rule above): the natural instinct is one `Coincident`
constraint per joint (closes the loop) plus one generic `Sketcher.Constraint('Tangent', edge1, edge2)`
per joint (keeps the line tangent to the arc). **This is wrong and causes a silent, hard-to-diagnose
bug:** the generic edge-edge Tangent constrains the *underlying infinite circle vs infinite line*
(perpendicular distance from center = radius) - which, once the center is already pinned by
`PointOnObject`+`Distance` and the radius by a `Radius` constraint, is already fully implied and gets
flagged **redundant**. Deleting the flagged item doesn't fix it - a *different* constraint (radius,
one of the PointOnObject pair, one of the Distance pair) gets flagged redundant next, because the
generic Tangent + separate Coincident pair never actually pins the arc's start/end angle (which
endpoint of the full circle the trim uses) - that DOF stays genuinely free the whole time, so no
matter which "redundant" constraint you remove, the true fix (pinning the angle) never happens and
the flag just migrates. **While a sketch reports `RedundantConstraints`, FreeCAD does NOT rebuild
its `.Shape` outside the sketch editor** - inside edit mode the preview still renders live from the
constraint solver's working geometry, so it looks fine there, but the object's real `Shape` property
(what Pad/every downstream feature and every outside-edit-mode view actually uses) stays stuck on
the last successfully-computed state. This is exactly why a sketch can look right while editing it
and wrong (or stale/missing geometry) the moment you close the sketch - always suspect
`sk.getStatusString()` / `sk.RedundantConstraints` first when that happens.

**The fix:** use FreeCAD's *point-to-point* Tangent constraint at each joint instead -
`Sketcher.Constraint('Tangent', edge1, posId1, edge2, posId2)` where posId1/posId2 are the actual
matching endpoint positions (1=start, 2=end). This single constraint subsumes both the coincidence
AND the tangency/angle-continuity at that vertex in one non-redundant equation - drop the
separate `Coincident` constraint at that joint entirely, don't add both. Full correct recipe per
slot (2 lines + 2 arcs, closed loop): 4x point-to-point `Tangent` (one per joint, no separate
`Coincident`) + `Radius` on one arc + `Equal` between the two arcs + `PointOnObject`(arc center,
centerline) + `Distance`(centerline start point, arc center) for each of the two arc centers.
That is exactly determined (0 DOF, `RedundantConstraints`/`ConflictingConstraints` both empty,
`getStatusString()` == "Valid") and the `.Shape` updates correctly outside edit mode immediately.

## 2026-09-11 (later still) - Sub-micron rounding gap silently split a closed wire into two open ones; App::Part visibility gates children again; a downstream feature error masks as "looks fine" in the viewport

Fixing h12's top-corner 45° chamfer to a proper fillet (Erik: the old chamfer was a hand-built
one-off, not a spec - replace with real fillets, same as the new h11 corners) required editing
`Sk_flat_full`'s outer-boundary geometry directly. That sketch's whole boundary chain turned out
to have **zero Coincident constraints anywhere** - built entirely from raw hardcoded coordinates
that happened to numerically match at full float precision, closing the wire by coincidence, not
by constraint (the same "wrong workflow" pattern already fixed for slots/holes, just never
revisited for the outer boundary itself).

- **When replacing part of an unconstrained raw-coordinate boundary, re-use the neighbor's exact
  point object, don't retype a rounded printout of it.** I read the neighboring geometry's
  endpoint via a `round(x,2)`-style debug printout earlier in the session and typed that rounded
  value into the new replacement line's coordinates. The result was off from the true value by
  ~0.00001-0.00004mm - invisible at any sane rounding, but enough that FreeCAD's wire-builder
  treated the two pieces as separate open wires (`shape.Wires` returned an 11-edge open wire and a
  5-edge open wire instead of one closed 16-edge wire) rather than merging them. `sk.solve()`
  still returned 0, `getStatusString()` still said "Valid" (the *sketch's own* 2D solve doesn't
  care about wire closure) - only `Part.makeFace`/the Pad's own recompute caught it. **Fix: pull
  the neighbor's point with full float precision (`repr(g.StartPoint.x)`, not a rounded readout)
  and add a real `Coincident` constraint there**, not just matching numbers - this also makes the
  joint robust to the next edit instead of being one more silent-gap trap.
- **A sketch's own `fillet()` call can still "trim the wrong segment" even when using two distinct
  reference points** (the existing documented mitigation) - it happened again on this session's
  *second* fillet call in a row, producing two open wires from what should have been one closed
  loop, while the sketch itself stayed `Valid`/non-redundant/non-conflicting throughout. Don't
  trust `sk.solve()==0` + `getStatusString()=="Valid"` alone as proof a fillet pass worked -
  always also check `sk.Shape.Wires` count/closure (or `Part.makeFace` on them) right after a
  `fillet()` call on boundary/outline geometry, before moving on to the next corner.
- **When a downstream PartDesign feature (Pad, Fold, ...) fails to recompute, it keeps its LAST
  GOOD shape and the 3D view keeps showing that stale shape - not an error, not a blank, a
  perfectly normal-looking but out-of-date part.** Erik looked at the live viewport mid-edit and
  said "see how the part is now correct... confirm visually" - the part he was looking at was
  genuinely fine, but it was the pre-edit `Foot_folded`, not a reflection of the just-made (at
  that point still broken) sketch edit. Checked and found `Pad_flat_full.getStatusString()` ==
  `"Wire is not closed."` while `Foot_folded.getStatusString()` == `"Valid"` (holding its old
  shape). **The tell: check every object's own `getStatusString()` along the dependency chain,
  not just the one you just edited** - a clean-looking downstream feature can be silently stale
  while its upstream input is broken. Don't confirm "looks right" from a viewport screenshot
  alone when a recompute error is possible - check the actual status strings first.
- **`App::Part` group visibility gating a child regardless of the child's own `Visibility=True`**
  recurred a second time this session (first was `saveImage` producing a blank PNG earlier
  09-11) - this time as "I don't see anything" when the sketch's own `ViewObject.Visibility` was
  already `True` but its containing `TrimPlates` `App::Part` group had `Visibility=False`. Same
  fix as before: walk the object's ancestry (`InListRecursive` or check each named parent) and
  set every ancestor `App::Part`'s own `Visibility=True`, not just the leaf object's.
- **`get_screenshot` MCP tool is still broken** (`AttributeError: 'dict' object has no attribute
  '__name__'`, same as the earlier-documented failure) - use the manual `execute_python` +
  `view.viewIsometric()`/`view.viewTop()` + `FreeCADGui.SendMsgToActiveView("ViewFit")` +
  `view.saveImage(path, w, h, "White")` sequence instead, and set a strong `ViewObject.LineColor`
  (e.g. pure blue `(0,0,1)`) + `LineWidth` on the sketch being inspected - the default line
  rendering came out nearly invisible (very light grey on white) in one save, so don't trust a
  faint/blank-looking save as proof of an empty scene without also checking colors/line width.

## 2026-09-11 (even later) - Two real bugs disguised as one: a genuine solver-instability trap around `sk.fillet()` + generic edge-tangent slot constraints, and the fix that actually held (redraw over patch)

Getting h11's `Sk_flat_full_h11` from "redundant constraints, Pad throws `NULL shape`" to a clean
`Pad`/`Fold` took a rebuild, not a patch - worth recording precisely what failed and what worked,
because the failure mode looks exactly like ordinary redundant-constraint cleanup (covered above,
"THE sheet-metal workflow") but isn't safely fixable the same way once a sketch has accumulated a
specific bad combination.

- **A slot built with 4x `Coincident` (loop closure) + 1x generic edge-edge `Tangent` per joint
  (instead of the correct 4x point-to-point `Tangent`, see the entry above) leaves the sketch
  reporting exactly one "redundant constraint" per slot, and Pad/PartDesign refuses to build a
  solid from a sketch in that state** - `sk.Shape` / `Part.makeFace` on the sketch's own wires
  report perfectly valid, closed, correct geometry throughout, but `PartDesign::Pad.Shape` throws
  `Standard_NullObject: BRepCheck_Analyzer::Init() - NULL shape` and the Pad object's own
  `getStatusString()` shows the sketch's error text verbatim. **Confirmed by direct test: the same
  sketch, same geometry, Pad succeeds instantly once `RedundantConstraints` is genuinely empty, and
  fails identically at both "2 redundant" and "1 redundant" - Pad needs true zero, not "close
  enough".**
- **Deleting the generic Tangent (or its co-located Coincident) once, in an otherwise-untouched
  freshly-reopened sketch, resolved that slot's redundancy cleanly (wire stayed closed, 14-edge
  outer boundary, radii exact).** But **doing the identical, individually-correct fix to the
  SECOND slot in the same session - regardless of which slot went first - reliably corrupted the
  boundary's 7 corner fillets** (outer wire dropped from 14 edges to 7, with 6-7 formerly-fillet
  edges becoming disconnected open 1-edge "wires"), even though the edited constraints were on
  slot geometry with no direct topological relationship to the plate/tab boundary. Re-adding the
  deleted constraint did NOT undo the corruption - the geometry had already resolved to a
  different (wrong) numerical solution, not just lost a constraint. This reproduced 3 times with
  different specific constraints deleted, always on "the second such edit this session" regardless
  of order. **Saving and reopening the document between the two edits did NOT prevent the second
  one from corrupting the boundary either** - so this is not an in-memory-only solver cache issue,
  it's something that reproduces from the saved file state too once the first fix's geometry is
  baked in. Suspect a Newton-Raphson bifurcation in the
  sketch's 2D solver that a slightly different starting seed (post-first-edit) pushes onto a
  different, unstable branch for the WHOLE sketch, not a local effect - not fully root-caused.
- **What actually worked, on Erik's direction ("perhaps redraw h11 sketch?"): `sk.deleteAllGeometry()`
  and rebuild the entire sketch from scratch in one clean pass, building the slots with the correct
  4x point-to-point `Tangent` recipe from the very first `addConstraint` call** (never creating a
  generic edge-tangent at all, so there's nothing to later discover as redundant). Applying all 7
  corner fillets one at a time with a wire/status check after each (rather than batching them) kept
  every single step at `Valid`/0-redundant/0-conflicting, and the resulting sketch padded on the
  first try. **When a sketch has been patched many times across a long session and starts showing
  this kind of "fix one thing, something unrelated breaks" behavior, stop patching and redraw** -
  a fresh, clean build of the same geometry is faster and more reliable than continuing to fight
  accumulated solver state, even when every individual patch is provably correct in isolation.
- **Corollary process lesson: verify a downstream feature's OWN status, not just the upstream
  sketch's, before reporting something "looks correct" from a live viewport** - mid-fix, the 3D
  view still showed the pre-edit `Foot_folded` (last good shape, PartDesign features hold their
  last successful Shape when recompute fails) while the just-edited sketch was actually broken;
  `Pad_flat_full.getStatusString()` said `"Wire is not closed."` at the exact moment the viewport
  looked fine. Check every object's status along the chain, not just the one just touched.
- **A `sk.RedundantConstraints` (or similar diagnostic property) report can itself go stale/stale-index
  after a `delConstraint`/`addConstraint` pair** - re-run `sk.solve()` before trusting the property,
  and treat an out-of-range index in the reported list (e.g. "28" when `len(sk.Constraints)==28`,
  i.e. only indices 0-27 exist) as a 1-based constraint number, not a 0-based Python index - check
  `sk.Constraints[reported_number - 1]` when the literal index errors.

## 2026-09-11 (later still) - SheetMetal `Fold`'s face reference is a baked name, not a live link - it goes stale on every sketch edit, silently

`SMFoldWall`'s `baseObject` property is an `App::PropertyLinkSub` storing `(pad_obj, ["FaceN"])` -
a face **index baked in at creation time**, not something that re-resolves when the Pad's shape
changes. Editing the flat sketch upstream (adding/removing any geometry) commonly changes the
Pad's total face count/ordering, so `"Face29"` silently stops being the flat face it used to be.
**The symptom is exactly the kind of silent-stale-shape trap documented above**: the Fold object's
`getStatusString()` reports `"GeneralFuse failed"` (or similar), but `fold_obj.Shape` still returns
the OLD, LAST-GOOD shape (valid, with a volume) rather than erroring or going blank - so a viewport
screenshot or a bare `Shape.isValid()` check both lie. **After ANY edit to a flat sketch that
already has a Fold built on top of it, always check the Fold's own `getStatusString()` explicitly**
- don't trust that `doc.recompute()` alone propagated correctly just because no exception was
raised.

Erik's framing (citing general FreeCAD dependency-chain practice - sketches should attach to
origin/datum planes, not to a named generated face, precisely because generated-face names shift):
the Fold's face reference is exactly that anti-pattern, just one level below where it usually
happens (Fold→Pad's face, not Sketch→Pad's face) - and SheetMetal's `Fold` genuinely has no
datum-plane-attachment alternative, it requires a face selection on a solid. **The fix that
respects the same principle without fighting the tool: re-resolve the target face by a stable
GEOMETRIC signature (not a name) and update the existing Fold object's `baseObject` in place**,
rather than deleting and recreating the Fold (which was tried first and works, but churns the
object's identity/name in the tree for no reason - `Foot_folded_h11` became a stale reference
anywhere else it might have been used, and any GUI selection/view settings on it are lost). Minimal
reusable pattern:

```python
def resolve_and_fix_fold_face(pad_obj, fold_obj):
    faces = pad_obj.Shape.Faces
    candidates = []
    for i, f in enumerate(faces):
        n = f.normalAt(0, 0)
        if abs(abs(n.z) - 1.0) < 1e-6 and f.Area > 1000:  # big flat face, normal along Z
            candidates.append((i + 1, f.Area, n.z))
    bottom = [c for c in candidates if c[2] < 0]           # convention: fold acts on the -Z face
    face_idx = max(bottom, key=lambda c: c[1])[0]
    new_ref = f"Face{face_idx}"
    if fold_obj.baseObject[1][0] != new_ref:
        fold_obj.baseObject = (pad_obj, [new_ref])          # in-place update, no object recreated
doc.recompute()
resolve_and_fix_fold_face(pad_obj, fold_obj)
doc.recompute()
```

**Standing rule for every future module trim plate (and any other sheet-metal part built with
this Fold pattern): after any edit to the flat sketch, call this resolver and re-recompute before
trusting the Fold's shape** - treat it as a required step of the edit, not a symptom to debug
after the fact when something looks wrong downstream.

## 2026-09-12 - a broken expression leaves the shape silently stale, not visibly errored

Deleting a piece of geometry (e.g. a hole) that another constraint's expression still
references (`.Constraints.clamp2_v - ...` after `clamp2_v` itself was deleted) does not throw
a visible error in the 3D view or block the document from looking "normal." The sketch's
`.Shape`/downstream Pad simply **stops updating and keeps showing the last successfully
computed geometry** - a render taken after the break looks identical to one taken before it,
even though the actual current state is broken. This is the same class of silent-staleness
bug as the `RedundantConstraints` case above (FreeCAD does not rebuild `.Shape` outside the
sketch editor when something's wrong) - just triggered by a dangling expression instead of a
redundant constraint.

**After deleting any geometry that other constraints might reference by name, always check
`sk.solve()` (expect `0`) and `sk.getStatusString()` (expect `"Valid"`) explicitly** - a
non-"Valid" status (e.g. `"No attribute named 'X'"`) means the shape is stale and must be
fixed (rebind the dangling expression to a real, remaining feature) before trusting any
screenshot or downstream measurement taken against it.

## 2026-09-12 - a recursively-copied VarSet can break EXTERNAL expression bindings into it, while its OWN internal cross-references keep working

Migrating `bench_capstan_drum.FCStd`'s drum (`doc.copyObject(BenchCapstanDrum, True)`) into
the master assembly, then moving the copied `VS_BenchDrum` out of the copied part into the
shared `Params` group (same pattern used successfully for the bracket's `VS_Bracket` and the
gearmotor's `VS_Gearmotor`) - this one didn't work. Symptom: 5 sketch constraints referencing
`<<VS_BenchDrum>>.OD` by expression never recomputed, no matter the value set (tested with a
wildly different value, 200 vs the original 100, zero change) - yet the SAME VarSet's own
internal property `RimRadius` (itself expression-bound to `OD`, `= OD/2 - 9`) tracked the
change correctly every time. Ruled out: sketch lock (a plain `setDatum()` on the same
constraint, after clearing its expression, worked immediately); label collision (only one
object had that Label); redundant/conflicting/malformed constraints (none); document-wide
recompute not running (`doc.recompute()` returned real touch counts). Also tried, still
broken: fully clearing the expression (`setExpression(path, None)`) and rebuilding it from
scratch - the fresh binding still didn't evaluate.

**Conclusion: a VarSet's OWN internal expressions survive a recursive `copyObject` +
re-parenting, but something in the compiled dependency-graph edge for EXTERNAL objects
binding INTO it does not** - the expression text is correct and re-settable, but the
recompute dependency link stays broken. Not fully root-caused; not worth re-attempting the
same copy-then-move sequence a second time once hit. **The fix that worked: delete the
copied part and its VarSet entirely, rebuild both natively in the target document** (fresh
`App::Part`/`Body`/sketch/VarSet, not copied) - a native VarSet in the same document that
things were always going to bind to doesn't have this problem. If a future migration needs
to bring in an existing part WITH ITS OWN driving VarSet, either (a) keep the VarSet inside
the copied part's own group rather than moving it to a shared location afterward, and test
one external binding before trusting the rest, or (b) just rebuild the VarSet+driving
sketch natively from the start and treat the copy as reference geometry only.

## 2026-09-12 - App::Link to an Assembly::AssemblyObject mirrors the CHILDREN's own visibility, not the assembly container's top-level Visibility flag

`MotionUnit` (`Assembly::AssemblyObject` holding real `Gearmotor`/`Bracket`/`Drum` + joints)
linked into `Module_H10_Real` via `App::Link` (`MotionUnit_h10`). Two separate toggles,
easy to conflate, confirmed by direct test AND by Erik watching the live GUI in real time:

- **Hiding `MotionUnit` itself (the assembly object's own `Visibility`) does NOT hide the
  link.** Erik confirmed this directly - the link stayed visible in his session.
- **Hiding the assembly's own CHILDREN (`Gearmotor`/`Part`/`Drum` individually) DOES hide
  the link** - confirmed both by Erik seeing `MotionUnit_h10` disappear right after a
  script set the three children invisible, and by the bridge's own before/after
  `.ViewObject.Visibility` reads.

So the link mirrors "does the linked assembly currently have anything visible inside it",
not the assembly's own top-level flag. Practical consequence: the shared master parts
inside an Assembly-type sub-assembly need to stay individually visible for ANY link to
that sub-assembly to render anything, anywhere it's linked - there's no clean
"master fully hidden, link still shows" state for this object type the way there is for a
plain `Part::Feature`/`PartDesign::Body` link (confirmed working differently for the
drum's own body earlier the same session). A mid-diagnosis version of this entry wrongly
blamed the bridge's screenshot pipeline instead - corrected once the real pattern (parent
toggle vs. children toggles) was actually isolated.

## 2026-09-12 - SheetMetal SMBendWall: chaining a second wall off the first one's OWN shape can corrupt both; base independent walls off the same original solid instead

Forming a U-channel from a flat strip needs two flange walls, one per long edge. First
attempt: `SMBendWall(obj2, obj1, ['Edge9'])` where `obj1` was the already-created first
flange (chaining the second off the first's own current shape, picking an edge index that
was numerically valid on `obj1`'s shape at the time). This threw `RuntimeError: Property
'radius' not found` on `obj2` **and** silently corrupted `obj1` (`obj1.Shape.isNull()`
became `True`, `hasattr(obj1, 'radius')` became `False`, even though `obj1.isValid()` and
`obj1.State` still reported clean) - not an isolated failure, a shared-state corruption.
Retried the identical `SMBendWall(obj2, obj1, ['Edge9'])` call afterward in isolation and
it worked fine, so the exact trigger wasn't fully isolated - treat chaining one
`SMBendWall` off another's live shape as fragile in this bridge/session regardless.
**Fix that worked cleanly and matches how the existing reference part
(`50_lib_frame_member.FCStd`) actually does it: base BOTH flange walls independently off
the SAME original pad/extrusion** (`SMBendWall(obj1, pad, ['Edge3'])` and
`SMBendWall(obj2, pad, ['Edge9'])`, not `obj2` based on `obj1`) - `obj2`'s own resulting
shape correctly includes both flanges regardless (SheetMetal composes them), so nothing is
lost by not chaining. Also needed each time: `obj.ViewObject.Proxy = 0` after
`addObject("Part::FeaturePython", ...)` for the shape to display, and the resulting
objects must live directly in the `App::Part`, NOT inside a `PartDesign::Body` -
`doc.addObject` into a Body's `Group` raises `ValueError: Body: object is not allowed`
for non-PartDesign feature types like `Part::FeaturePython`.

## 2026-09-12 - a sketch left mid-rebuild with the driving VarSet deleted does NOT error on recompute - it silently freezes at the last cached value, and a hand-patched literal beside expression-bound siblings goes unnoticed

Following the fix above (native rebuild after the copyObject failure), the drum's
`Sketch_RevolveProfile`/`Pad_Flat`/`Pocket_Grub`/chamfer chain got built correctly in a
session that was never logged, but the actual `VS_Drum` VarSet was never (re)created before
the file was saved and the session ended. Two silent failure modes found on reopening,
neither of which FreeCAD flagged as an error:

1. **A dangling `<<VS_Drum>>` expression with no such object in the document does not error
   on `doc.recompute()`.** It just keeps the last successfully-computed value forever. Here
   that value was `OD=100` (a stale intermediate from the diameter saga), while the decided
   value was `Ø70` - the file looked clean (no red marks, `isValid()==True` everywhere) and
   was quietly wrong.
2. **One sketch constraint (`v1_x`) had been hand-patched to a bare literal (`35`, correct
   for Ø70) instead of an expression**, while its four sibling constraints (`v2_x`..`v5_x`,
   same profile) stayed expression-bound to the dead VarSet at the stale `50` (Ø100/2). The
   sketch computed a real, valid-looking solid shape from this mismatched profile - a stepped
   revolve that was silently wrong, not a solver error.

**Check for this whenever reopening a sketch you didn't just build**: dump every named
constraint's `.Value` next to its `ExpressionEngine` entry (or lack of one) and confirm the
values that share a design relationship (e.g. all four corners of one profile radius) actually
agree, rather than trusting `Shape.isValid()` - a self-inconsistent profile still produces a
"valid" shape.

**A second, unrelated defect surfaced in the same document by the same check**: a chamfer
feature (`Chamfer_Flanges`) had lost its `Base`/`BaseFeature` link entirely (`Base: None`,
error `"No Base object linked"`) but still reported `Shape.isValid()==True` with a stale
cached shape - PartDesign keeps the last good shape on a compute failure rather than clearing
it. `isValid()` on a feature with a broken base link tells you nothing; check
`getStatusString()` / `o.isValid()` (the document-level validity list) directly. Fix: delete
the orphaned feature and everything downstream of it, and reset `Body.Tip` to the last real
good feature - don't try to repair `BaseFeature` via the API, PartDesign manages that link
internally and it isn't meant to be hand-set.

**Expression-engine gotcha hit while fixing this**: `<<VarSet>>.Length / 2 + 5` throws
`Quantity::operator +(): Unit mismatch in plus operation` - a bare number can't be added to a
`Length` result. Write `+ 5mm`, not `+ 5`.

## 2026-09-12 - SheetMetal SMFoldWall: `invertbend` controls WHICH SIDE folds, `invert` only controls rotation SENSE - two separate properties, easy to conflate

Building a multi-flange sheet (a flat strip folded into a U-channel, web in the middle,
a flange on each long edge) via two `SMFoldWall` operations. The requirement: the web must
stay exactly on its original sketch plane; only the flanges move. Confirmed by direct
measurement across many failed attempts that **`invert` does NOT control which side of the
bend line is fixed vs moving** - toggling it only flips the rotation direction of whichever
side the tool has already decided is "the moving one" (by default, seemingly the larger-area
side). The actual property is **`invertbend`** - a separate, easy-to-miss property on the
same `Part::FeaturePython` Fold object (`sorted(fold_obj.PropertiesList)` reveals both
`invert` and `invertbend` side by side). Verified by direct measurement of a known point
(the web's own center, which should stay at its original Z) before and after toggling each
property independently - `invertbend=True` kept the correct side fixed for the first fold;
the **second** fold on the same strip needed `invertbend=False` (the opposite value) for the
same "keep the web fixed" outcome, because it operates on the mirrored edge of the same
piece. **Don't assume both folds on a symmetric part share the same `invertbend` value -
verify each one independently by checking a known-should-not-move point's real coordinates,
not by eye and not by bounding-box extent alone** (a bounding box can look "roughly right"
while the wrong side has actually moved - this cost several rounds of the wrong diagnosis
in this session, including once wrongly declaring a Z-shaped defect "fixed" based on a
misleading axis-aligned box-clip cross-section of geometry that had rotated off the main
axes - always re-derive the real cross-section by finding actual faces/vertices at the
expected coordinates, never trust a box-boolean clip once anything has rotated).

## 2026-09-13 (Erik) - toggle the top-level container's Visibility, not every child in the tree
**Correction 2026-09-14: the core claim below is wrong for a plain `App::Part` in this
FreeCAD build - see the entry below this one for the empirically-verified real behavior.**

When hiding or showing a set of objects that already live under one `App::Part` group (or
other container), set `Visibility` on the **container itself** - not on every individual
child object inside it. ~~FreeCAD's viewport respects a parent container's visibility as a
gate over its children~~ (**false, see 2026-09-14 correction**), so toggling the one top-level
flag is both sufficient and far less code than looping over every object in the tree. This
session looped over 6+ individual segment objects plus their group to hide them, when
setting the group's own `Visibility` alone would have done it. **Caveat, from the two
entries above**: this only holds for a plain container whose children's own visibility
hasn't been separately forced - an `App::Link` to an `Assembly::AssemblyObject` mirrors the
**children's** visibility rather than the container's (2026-09-12 entry above), and
re-parenting into a group can silently flip a child back to visible (2026-09-11 entry
above) - so this shortcut is the right default, but re-verify with a real screenshot or
`obj.Visibility` check when the container is anything other than a plain
`App::Part`/`PartDesign::Body` you built yourself.

## 2026-09-14 (Erik) - SUPERSEDED, kept for the record: an earlier same-day pass wrongly
concluded "hiding a container does NOT gate its children" from a false-positive test (a
non-blank screenshot that was actually a stale, un-moved camera frame left over from a
prior successful fit - see the final corrected entry below, which supersedes this one
entirely). Left here struck through rather than deleted, per "walk edges back out": ~~setting
`Frame.ViewObject.Visibility = False` (an `App::Part`) left `HexRingSegments` (Visibility=
True) fully rendering in a screenshot, proving the parent's flag isn't a gate~~ - **false**;
re-tested properly (camera-moved check, see below) and `Frame` hidden DOES stop
`HexRingSegments`/`RingSeg0` from rendering, round-tripped both ways. The blank-screenshot
diagnosis in the original version of this entry (small-scene theory, "leave an anchor object
visible") was also incomplete - the real, precise mechanism is documented below.

## 2026-09-14 (Erik) - View/camera rendering: visibility gating, verifying a screenshot is real, and the infinite-bbox fit-poisoning bug

Three findings from the same session, all interlocking - the second was needed to catch that
the first attempt at the first finding was wrong, and the third is a distinct mechanical bug
found while chasing the first two.

**1. Visibility gating is real and uniform - but there are two different mechanisms
depending on whether the thing you're hiding is a container or a shared leaf.**
- **A container (`App::Part`, `PartDesign::Body`, `Assembly::AssemblyObject`) gates its own
  branch of the scenegraph.** Hide it, and everything beneath it on THAT branch stops
  rendering, regardless of each child's own `Visibility` flag (which stays whatever it was -
  the flag isn't changed, it's just not consulted while an ancestor above it is hidden).
  This holds for plain `App::Part` exactly the same as for `Assembly::AssemblyObject` - there
  is no special case; an earlier version of this entry claimed otherwise and was wrong (see
  the superseded entry above).
- **A shared/linked leaf object's own `Visibility` is a single global property, because
  there is only one instance of it in the document.** An `App::Link` (e.g. `MotionUnit_h10`
  linking to `MotionUnit`) does not duplicate `MotionUnit`'s children (`Gearmotor`, `Part`,
  `Drum`) - it displays the SAME objects through its own, independent branch of the
  scenegraph. So: hiding the LINK'S OWN ancestor chain (the link object itself, or its
  container) only gates that one branch, leaving the master's branch (and any other link's
  branch) completely unaffected. But hiding the LEAF ITSELF (`Drum.ViewObject.Visibility =
  False`) hides it on every branch that reaches it - master included - because every branch
  ultimately reads that one shared flag. Confirmed by direct test: hid `Drum` while looking
  at the master `MotionUnit` branch, then switched to viewing purely through
  `Module_H10_Real -> MotionUnit_h10` (master hidden entirely) and `Drum` was gone there too.
- **Practical consequence - two different tools for two different intents:**
  - To hide ONE instance/branch only (e.g. just the H10 motion unit) without touching the
    master or any other instance: hide that instance's own ancestor/container (the link
    object, or its containing assembly) - never the shared leaf.
  - To hide something everywhere, in every instance at once: hide the leaf/component object
    itself.
  - Hiding the wrong one for your intent is the exact bug this session kept tripping on:
    hiding the master's container looks like "the geometry disappeared" only if you forgot a
    link elsewhere still has its own independent, visible path to the same shared data.
- **Set visibility at the coarsest node that achieves the intended result, and stop there -
  don't recurse into children, and don't leave visibility state split across multiple depths
  (some children individually toggled while an ancestor is also toggled).** A leaf flipped
  off independently of its container becomes a landmine: the container's own flag no longer
  fully describes what's actually shown, so a later toggle of the container alone won't
  reveal what you expect. This is the practical meaning of "coarsest grain" - not just less
  code, but keeping exactly one flag authoritative for any given piece of geometry's
  visibility at the depth you're actually working at.

**2. Never trust "the screenshot isn't blank" as proof a fit actually worked - verify the
camera position itself changed.** `Std_ViewFitSelection` (and, less often, `fitAll()`) can
silently no-op - selection succeeds, the command reports nothing wrong, but the camera never
moves - typically because the selected/fitted object isn't actually part of the rendered
scenegraph right now (its ancestor chain is hidden per the gating rule above). The result is
a perfectly normal-looking, non-blank screenshot that is actually just the LAST frame that
WAS successfully rendered, showing completely unrelated geometry. This produced multiple
wrong conclusions in this session before being caught. **The fix: capture
`av.getCamera()` before the fit call (ideally after first moving to a neutral/different view
like `viewFront()`, so a truly-identical camera can't be a coincidence), run the fit, capture
`getCamera()` again, and compare the two strings for equality.** Only trust the resulting
image if the camera actually changed. A `PIL.Image.getcolors()` check for "is it blank"
(below) is a separate, necessary-but-not-sufficient check - non-blank does not mean correct.

**3. Real, distinct cause of blank-white `saveImage`/`get_screenshot` output: a VISIBLE
object with an infinite bounding box poisons the camera-fit calculation.** Every
`PartDesign::Body`'s default `Origin` (its `X_Axis`/`Y_Axis`/`Z_Axis` lines and
`XY_Plane`/`XZ_Plane`/`YZ_Plane` planes) has a bounding box of literally `+-1e100` by design
(they're meant to be visually infinite reference geometry) and often default to
`Visibility=True`. If `fitAll()` or `Std_ViewFitSelection` computes a scene/selection bbox
that includes even ONE such object, the camera zooms out to frame "infinity," and every real
(mm-scale) object renders as a sub-pixel dot - i.e. a pure white frame. This is NOT the same
bug as the 2026-08-31/2026-09-07 Coin redraw-cache blank-render entries above (those need a
visibility-toggle+recompute+updateGui loop) and is NOT primarily about "scene too sparse"
(an earlier version of this entry guessed that, from the correlation that isolating down to
few objects made it more likely one of the remaining visible ones was a poisoned datum) -
the real, precise, mechanical cause is the infinite bbox, confirmed by finding 138 such
visible objects document-wide via the scan below, hiding them, and getting a normal
`fitAll()` (no anchor-object workaround, no selection tricks) to render cleanly immediately
after.
**Diagnostic/fix, worth running before any screenshot session on an unfamiliar document:**
```python
def find_poisoned_visibles(doc):
    poisoned = []
    for o in doc.Objects:
        if hasattr(o, "ViewObject") and o.ViewObject and o.ViewObject.Visibility and hasattr(o, "Shape"):
            try:
                bb = o.Shape.BoundBox
                if not bb.isValid():
                    continue
                if max(abs(bb.XMin), abs(bb.XMax), abs(bb.YMin), abs(bb.YMax), abs(bb.ZMin), abs(bb.ZMax)) > 1e50:
                    poisoned.append(o)
            except Exception:
                pass
    return poisoned

bad = find_poisoned_visibles(doc)
for o in bad:
    o.ViewObject.Visibility = False  # pure visibility change, touches no real geometry
doc.recompute()
```

**Working screenshot recipe, consolidated:**
1. Run the infinite-bbox scan above once per document/session before relying on any fit.
2. `Gui.getDocument(doc.Name).ActiveView` (or `Gui.ActiveDocument.ActiveView` once
   `Gui.ActiveDocument` is confirmed set) is the reliable view handle.
   `mcp__freecad__get_screenshot` is still broken in this bridge build (`'dict' object has
   no attribute '__name__'`, confirmed again this session) - use `execute_python` +
   `saveImage` instead.
3. To focus on one object: `Gui.Selection.clearSelection()`,
   `Gui.Selection.addSelection(doc.Name, "TheObject", "")`,
   `Gui.runCommand("Std_ViewFitSelection", 0)`, `Gui.updateGui()`, then
   `Gui.Selection.clearSelection()` before `saveImage` (selection highlight color can mask
   small holes/slots, producing a false "looks fine" read).
4. **Verify the camera actually moved** (finding 2 above) before trusting the result.
5. **Verify the image isn't blank**: `PIL.Image.open(path).getcolors(maxcolors=2000000)`
   returning a single-entry list is a blank frame. Necessary but not sufficient - do this
   in addition to, never instead of, the camera-moved check.

## 2026-10-01 - Model sheet parts as made: flat blank, then Add Wall. Make Bend and solid-first fillets both fail on mitred parts
Tested on a 2 mm L-angle frame segment with mitred ends (KinSculpter `RingSeg1`), SheetMetal 0.8.22 on FreeCAD 1.1.4.
- **Make Bend crashes on any solid that already has holes**: `smFindEdgeByVerts` indexes `edge.Vertexes[1]` on every edge, and a circle edge has one vertex -> `IndexError: list index out of range`. Bend before cutting holes, or do not use it.
- **It must be given the OUTER corner edge.** Given the inner edge it fillets the wrong pair (seen: inner R 2, outer R 5.055, axes 7 mm apart).
- **On a mitred end it measures thickness across the mitre**: it takes the distance from the edge's end vertex to a neighbouring edge, which on a 30 deg mitre is t / cos 30 (2.309 for 2 mm). Outer radius and wall come out wrong (wall 1.87-2.02).
- **It needs one continuous corner**: a relief or step at the ends (web proud of the bend edge) means no matching inner edge -> `UnboundLocalError: resultSolid`.
- **A fillet pair on the solid gets the bend thickness right but not the part:** `makeFillet(R, [inner])` then `makeFillet(R + t, [outer])` gives one axis and a constant wall (check by sampling `Part.Vertex(outer.valueAt(u, v)).distToShape(inner)`), but on a mitred end the flat comes out with a bevelled web end and a kinked bend-zone end that a press brake never makes. Shapes are not made that way; do not model them that way.
- **Use instead - PROVEN 2026-10-01 on a test hex-ring segment (2 mm, 25 x 25 L, 30 deg mitres), step by step with Erik:**
  1. **One sketch = the whole flange blank:** outline with the in-plane mitred ends, plus every hole and notch, fully constrained, named dims. Trapezoid recipe: 4 lines, `Coincident` chain, `Horizontal` on the two long sides, outer edge start `Coincident` to the root point (-1,1), `DistanceX` (0,1,0,2) = outer length, `DistanceY` (0,1,2,1) = width, and the end angles as `Angle(0,1,3,2, 60deg)` and `Angle(1,1,0,2, 60deg)` - the angle is taken at the SHARED corner (line 3's end meets line 0's start); the other end-pairing gives `solve() == -3`. Holes: circle + `DistanceX`/`DistanceY` from the root + `Diameter`.
  2. `Part::Extrusion` of the sketch, `Dir (0,0,1)`, `LengthFwd = t`, `Solid = True` (holes come through with the default face maker).
  3. **Add Wall:** `w = doc.addObject("Part::FeaturePython", ...)`; `SheetMetalCmd.SMBendWall(w, plate, ["EdgeN"])`; `SheetMetalCmd.SMViewProviderTree(w.ViewObject)`; set `length`, `radius`, `BendType = "Material Outside"`, `angle = 90`. EdgeN = the outer long edge on the plate's BOTTOM face (z 0) folds the wall down. Bend comes out true: inner R, outer R + t, wall t all through.
  4. **Material Outside dimensioning:** the wall sits (t + R) outboard of the edge and its length runs from the plate's underside, so for an outer section D x W: sketch width = W - (t + R), wall `length` = D - (t + R). Measure the result's BoundBox - it must equal the outer section.
  5. **Wall ends:** `extend1`/`extend2` lengthen the wall and bend past the plate corners, but that puts a step in the flat. Keep them 0 (Erik): the web ends at the plate corners. At a 120 deg ring corner that leaves an open gap between the two webs (4.62 mm at the outer face, 2.31 mm at the inner, for 2 mm sheet + R 2) - the weld fills it. Do NOT cut the webs square at the OUTER corner: square-cut plates meeting at 120 deg cannot both reach it - they overlap (48.5 mm3 measured).
  6. **Corner check without the neighbour existing:** mirror the segment across the mitre plane (through the outer corner, along the bisector) - `s.mirror(corner, normal)` - and take `s.common(mirror).Volume`; it must be ~0.
  7. **Unfold:** `SheetMetalUnfoldCmd.SMUnfold(u, wall, ["FaceN"])` with FaceN = the plate's top face, `KFactorStandard = "ansi"`, `KFactor = 0.38`, recompute. Check: one valid solid, thickness t, no cylinders left, flat width = (W - (t+R)) + (D - (t+R)) + pi/2 * (R + k t) (46.335 for 25/25/2/2/0.38), every non-sheet face square to the sheet (0 deg), end outline = one angled flange edge meeting one square web edge, no jog.
- **Unfold scripted:** `SheetMetalUnfoldCmd.SMUnfold(obj, src, ["FaceN"])` on a `Part::FeaturePython`, set `KFactorStandard` and `KFactor`, recompute - works on a plain `Part::Feature` source as long as the bends are true cylinders of constant thickness. Default K is 0.4 ANSI.
- **Laser reality check on the flat:** every non-sheet face of the flat must be square to the sheet (normal z = 0). A mitre modelled across a leg's thickness unfolds as a 30 deg bevel a laser cannot cut - cut that leg square in the solid instead.
