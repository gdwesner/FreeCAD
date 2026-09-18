# AIMBI FreeCAD Modifications

This document tracks all modifications made to FreeCAD's source code for AIMBI.
These changes should be applied to a fresh FreeCAD clone before building.

**Current base**: FreeCAD 1.1.3 (tag `1.1.3`, released 2026-07-25). The `aimbi`
branch is the release tag plus the commits below. Rebased from the 1.0-era base
on 2026-07-30 with no conflicts (upstream still has not implemented the
rotation-lock feature; the `DisallowRotation` enum flags remain unwired in
1.1.3). The pre-rebase state is preserved in branch `aimbi-1.0-backup`.
Note: the 1.1.3 tag was fetched shallow (`--depth 1`); run
`git fetch upstream --unshallow` if full upstream history is needed.

## Change Log

### 1. View Rotation Lock (2024-XX-XX)

**Purpose**: Allow locking a 3D view to prevent rotation while still allowing
pan and zoom. Essential for level views where annotations are placed relative
to a fixed 2D orientation.

**Files Modified**:

#### src/Gui/View3DInventorViewer.h
- Added method declarations for navigation restrictions:
  ```cpp
  void setRotationDisabled(bool disable);
  bool isRotationDisabled() const;
  void setPanningDisabled(bool disable);
  bool isPanningDisabled() const;
  void setZoomingDisabled(bool disable);
  bool isZoomingDisabled() const;
  ```
- Added member variables:
  ```cpp
  bool rotationDisabled;
  bool panningDisabled;
  bool zoomingDisabled;
  ```

#### src/Gui/View3DInventorViewer.cpp
- Initialized flags in constructor (around line 453):
  ```cpp
  rotationDisabled = false;
  panningDisabled = false;
  zoomingDisabled = false;
  ```
- Added method implementations (after setPopupMenuEnabled):
  ```cpp
  void View3DInventorViewer::setRotationDisabled(bool disable) { ... }
  bool View3DInventorViewer::isRotationDisabled() const { ... }
  // etc.
  ```

#### src/Gui/Navigation/NavigationStyle.cpp
- Added rotation check at start of `spin()` method (around line 1108):
  ```cpp
  if (viewer->isRotationDisabled()) {
      return;
  }
  ```
- Added rotation check at start of `doRotate()` method (around line 1030):
  ```cpp
  if (viewer->isRotationDisabled()) {
      return;
  }
  ```

#### src/Gui/View3DViewerPy.h
- Added method declarations:
  ```cpp
  Py::Object setRotationDisabled(const Py::Tuple&);
  Py::Object isRotationDisabled(const Py::Tuple&);
  // etc.
  ```

#### src/Gui/View3DViewerPy.cpp
- Added method registrations in `init_type()`:
  ```cpp
  add_varargs_method("setRotationDisabled", ...);
  add_varargs_method("isRotationDisabled", ...);
  // etc.
  ```
- Added method implementations at end of file

**Python API**:
```python
# Get the viewer from a view
view = FreeCADGui.ActiveDocument.ActiveView
viewer = view.getViewer()

# Disable rotation (locks to current orientation)
viewer.setRotationDisabled(True)

# Check if rotation is disabled
is_locked = viewer.isRotationDisabled()

# Re-enable rotation
viewer.setRotationDisabled(False)

# Similar methods for panning and zooming:
viewer.setPanningDisabled(True/False)
viewer.isPanningDisabled()
viewer.setZoomingDisabled(True/False)
viewer.isZoomingDisabled()
```

**Note**: The `DisallowRotation`, `DisallowPanning`, and `DisallowZooming` flags
already existed in the `ViewerMod` enum but were never implemented. This change
wires them up properly with simple boolean flags (matching the pattern used for
`fpsEnabled`, `vboEnabled`, etc.).

---

### 2. View Clip Box (2026-09-17)

**Purpose**: Let a TechDraw part view draw only what lies inside a box, so
an elevation of a house on a site with outbuildings shows the house, a
section renders a chosen depth, and a long building stitches into pieces
with match lines. The outline is visible on the page while the box is
adjusted and never exported. Applies to every `DrawViewPart` subclass
(plans, elevations, sections, details, broken views).

**Properties added to `TechDraw::DrawViewPart` (group "Clip")**:

| Property        | Type            | Meaning                                                    |
|-----------------|-----------------|------------------------------------------------------------|
| `ClipEnabled`   | PropertyBool    | clip on/off                                                |
| `ClipCenter`    | PropertyVector  | box centre in model space; the view is centred on it       |
| `ClipWidth`     | PropertyLength  | extent along `XDirection` (0 = unbounded)                  |
| `ClipHeight`    | PropertyLength  | extent along the view's up axis, `Direction x XDirection`  |
| `ClipDepth`     | PropertyLength  | extent along `Direction`, centred on the box (0 = unbounded)|
| `ShowClipFrame` | PropertyBool    | draw the dashed outline on the page (never exported)       |

**Files Modified**:

#### src/Mod/TechDraw/App/DrawViewPart.h / .cpp
- The six properties above; `isClipped()`; `clipShape(shape)`.
- `getSourceShape()` returns `clipShape(...)` of the extracted shapes, so
  every consumer (execute, broken views, details, a section's base) is
  clipped. `clipShape` builds a `BRepPrimAPI_MakeBox` on the projection
  CS (`gp_Ax2(corner, Direction, XDirection)`), classifies each top-level
  piece of the source compound by its bounding box (inside: kept as is;
  outside: dropped; crossing: `FCBRepAlgoAPI_Common` with the box, kept
  uncut if the boolean fails) and returns a compound, or a null shape
  when nothing survives.
- `makeGeometryForShape()` uses `ClipCenter` as `m_saveCentroid` while
  clipped, so the view stays anchored on the box, not on whatever
  happens to be inside it.
- `mustExecute()` includes the clip properties; `onChanged()` requests a
  repaint when `ShowClipFrame` changes.

#### src/Mod/TechDraw/App/DrawViewSection.cpp
- `getShapeToCut()` returns `clipShape(shapeToCut)`: a section clips
  what it cuts with its own box (a cut plan over a hidden base included).
- `prepareShape()` anchors on `ClipCenter` while clipped.

#### src/Mod/TechDraw/Gui/QGIViewPart.h / .cpp
- `m_clipFrame` (`QGCustomRect`, dashed teal cosmetic pen) created in the
  constructor; `drawClipFrame()` called from `draw()` sizes it to
  `ClipWidth/ClipHeight * Scale` around the view origin (an unbounded
  extent follows the drawn geometry) and hides it when not clipped, when
  `ShowClipFrame` is false, or while exporting (`isExporting()`, like the
  view frame). It is a plain rect item, so `removePrimitives()` and
  `removeDecorations()` leave it alone.

**Python API** (properties are ordinary FreeCAD properties):
```python
view.ClipEnabled = True
view.ClipCenter = FreeCAD.Vector(2000, 3000, 1500)
view.ClipWidth, view.ClipHeight, view.ClipDepth = 4200, 0, 0   # 0 = unbounded
view.ShowClipFrame = False        # hide the outline
```
AIMBI wraps it in `aimbi.freecad.view_clip` (logged setters, fit to
components, split views with match lines, sync and rebuild).

**Notes**: the frame ignores the view's `Rotation`; a `DrawViewDetail`
inherits its base's clip but does not apply its own (its shape comes
from `getShapeForDetail()` on the base). Cosmetic edges removed after a
recompute stay in the drawn geometry until the next one - TechDraw
behaviour, not new.

---

## Applying Changes

To apply these changes to a fresh FreeCAD clone:

```bash
# Clone FreeCAD
git clone https://github.com/FreeCAD/FreeCAD.git
cd FreeCAD

# Copy modified files from AIMBI reference
cp /path/to/AIMBI/reference/FreeCAD/src/Gui/View3DInventorViewer.h src/Gui/
cp /path/to/AIMBI/reference/FreeCAD/src/Gui/View3DInventorViewer.cpp src/Gui/
cp /path/to/AIMBI/reference/FreeCAD/src/Gui/Navigation/NavigationStyle.cpp src/Gui/Navigation/
cp /path/to/AIMBI/reference/FreeCAD/src/Gui/View3DViewerPy.h src/Gui/
cp /path/to/AIMBI/reference/FreeCAD/src/Gui/View3DViewerPy.cpp src/Gui/
# change 2: view clip box
cp /path/to/AIMBI/reference/FreeCAD/src/Mod/TechDraw/App/DrawViewPart.h src/Mod/TechDraw/App/
cp /path/to/AIMBI/reference/FreeCAD/src/Mod/TechDraw/App/DrawViewPart.cpp src/Mod/TechDraw/App/
cp /path/to/AIMBI/reference/FreeCAD/src/Mod/TechDraw/App/DrawViewSection.cpp src/Mod/TechDraw/App/
cp /path/to/AIMBI/reference/FreeCAD/src/Mod/TechDraw/Gui/QGIViewPart.h src/Mod/TechDraw/Gui/
cp /path/to/AIMBI/reference/FreeCAD/src/Mod/TechDraw/Gui/QGIViewPart.cpp src/Mod/TechDraw/Gui/
```

Or create a git patch:
```bash
cd /path/to/AIMBI/reference/FreeCAD
git diff > ../patches/001-rotation-lock.patch

# Apply to fresh clone
cd /path/to/FreeCAD
git apply /path/to/patches/001-rotation-lock.patch
```

## Building

Prerequisites (Windows):
- Visual Studio 2019 or 2022
- CMake 3.16+
- Git
- FreeCAD LibPack (download from FreeCAD releases)

```bash
# Clone and build
git clone https://github.com/FreeCAD/FreeCAD.git
cd FreeCAD

# Apply AIMBI changes (copy files or apply patch)

# Configure
cmake -B build -S . -DFREECAD_LIBPACK_DIR=C:\path\to\LibPack

# Build
cmake --build build --config Release

# Rebuilding one workbench after a change (VS 2022 generator; cmake is
# the one bundled with Visual Studio if none is on PATH):
"C:\Program Files\Microsoft Visual Studio\2022\Community\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin\cmake.exe" \
    --build build --config Release --target TechDraw --target TechDrawGui -- -m:8 -v:m
# ~10 minutes for TechDraw + TechDrawGui; FreeCAD must not be running (the .pyd is locked)

# The built FreeCAD will be in build/bin/
```

See also: https://wiki.freecad.org/Compile_on_Windows
