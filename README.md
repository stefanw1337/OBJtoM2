# OBJtoM2
Standalone OBJ to World of Warcraft **Wrath (M2 version 264)** converter.
The original converter is credited to Garthog in the program; this repository
contains fixes to its OBJ geometry conversion. It does not depend on Blender.

## Build on Windows

Install Visual Studio with the Desktop development with C++ workload and CMake.
From the repository directory:

```powershell
cmake -S . -B build -A x64
cmake --build build --config Release
ctest --test-dir build -C Release --output-on-failure
```

Tests require Python 3. The executable is `build/Release/OBJtoM2.exe`.
The original `.vcxproj` targets the old v110 toolset; use CMake for modern builds.

## Convert a static prop with one texture

```powershell
.\build\Release\OBJtoM2.exe "oak.obj" "oak" --texture 'World\CustomTrees\oak.blp'
```

This writes `oak.m2` and `oak00.skin`. The output argument is a filename stem,
without `.m2`; its parent directory must already exist. `--texture` is the
**internal game archive path**, not the path to a local JPG or PNG. This mode
assigns the same opaque, single-sided material to all sections. It does not
read MTL files or convert images. Prepare the BLP texture separately and pack it
at exactly that archive path.

Omit `--texture` to use the original interactive material commands (`te`, `r`,
`tu`, `i`, `q`). Use `q` to save; an unexpected end of input aborts without saving.

## Separate trunk collision

```powershell
.\build\Release\OBJtoM2.exe "oak.obj" "oak" --texture 'World\CustomTrees\oak.blp' --collision "oak_trunk.obj"
```

`--collision` replaces full-mesh collision with a separate triangulated OBJ.
Its vertices and faces are used only for collision; UVs and normals are not
required. Face normals are calculated during export. Empty meshes, invalid
indices and degenerate triangles are rejected before saving. Author a closed
proxy with outward-facing triangles, in the **same coordinates and scale** as
the visible OBJ. For trees, omit foliage from that proxy. Without this option,
the original full-mesh collision behavior remains.

The converter does not silently rotate or scale either OBJ. Prepare both in
WoW model space (Z up) with the intended foot position at the origin. For a
Y-up source, the rotation `(x,y,z) -> (x,-z,y)` preserves handedness. Apply it to
both geometry and normals; scale both visible and collision meshes together.

## OBJ requirements and fixes

- Triangular faces with position, UV and normal indices (`v/vt/vn`) are required.
  Export UVs and normals and triangulate in your modeling application first.
- Whitespace, comments and negative relative indices are supported. Invalid
  indices, missing attributes and non-triangular faces produce an error.
- Export vertices are keyed by the complete position/UV/normal index tuple,
  preserving UV seams and hard normals. Shared positions can therefore produce
  more M2 vertices without increasing triangle count.
- Each nonempty group/material section has contiguous vertices and triangles.
  Faces before the first group and repeated faces are retained.
- UV V is flipped once, matching the original converter's convention.
- This writer conservatively accepts at most **65,535 exported vertices and
  65,535 triangle indices (21,845 triangles)**. Larger meshes are rejected, not
  truncated. These are limits of this implementation, not a universal M2 or UE
  polygon budget. Supporting larger meshes needs additional section handling.

## Validation and remaining limitations

The regression suite independently parses emitted M2/SKIN files and verifies
seams, normals, section ranges, index validity and rejected inputs.

This is an experimental static-model converter, not a full audited M2 exporter.
The writer generates collision from **the entire visible mesh** unless
`--collision` is supplied. Section centers/radii now come from the visible
geometry, while collision bounds come from the selected collision mesh.
Regression tests verify that these remain independent. It does not export wind
animation, modern PBR materials, or automatically adjust scale and axes.
A successful conversion is not proof of correct appearance in a game.

Preview the model and validate scale, orientation, materials, bounds and collision
in the target client before deploying an MPQ replacement. No game data is modified
by the conversion command itself.
