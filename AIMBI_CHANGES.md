# AIMBI FreeCAD Modifications

This document tracks all modifications made to FreeCAD's source code for AIMBI.
These changes should be applied to a fresh FreeCAD clone before building.

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

# The built FreeCAD will be in build/bin/
```

See also: https://wiki.freecad.org/Compile_on_Windows
