# EdgeCraft STL backend API

EdgeCraft is a stateless triangle-mesh finishing engine. Its public API accepts
an indexed triangle mesh, never mutates it, and returns a replacement mesh. The
same API works in a browser module, Web Worker, Node service, Electron process,
or CAD plugin host; it has no DOM, React, WebGL, or Cloudflare dependency.

## Import surfaces

| Import | Contents | Intended use |
|---|---|---|
| edgecraft-stl | Core API plus STL I/O | Normal package integration |
| edgecraft-stl/core | Analysis, validation, feature discovery, finishing | Host already owns import/export |
| edgecraft-stl/stl | STL parsing, welding, and encoding | Thin STL adapter |
| edgecraft-stl/mesh-ops | Low-level geometry routines | Advanced usage; less stable |
| public/edgecraft-backend.mjs | Fully bundled ESM backend | Direct browser or worker import |

The root, core, and STL imports supply TypeScript declarations. The bundled
backend is self-contained and portable.

## Mesh contract

The mesh is an object with vertices and faces. A vertex may be a numeric
three-tuple, an object with x/y/z properties, or a Three Vector3. A face is
exactly three integer vertex indices. The default result uses serialisable
three-tuples.

    {
      vertices: [[0, 0, 0], [1, 0, 0], [0, 1, 0]],
      faces: [[0, 1, 2]]
    }

Coordinates are unitless. If the host uses millimetres, every depth, tolerance,
bound, and returned coordinate is in millimetres. Input is defensively copied.
Because every finish reconstructs and compacts topology, output vertex and face
indices are new identities. Do not retain them as persistent CAD subshape IDs.

## Core functions

### analyzeMesh(mesh, options)

Computes topology and shape diagnostics without altering the mesh. The optional
featureAngleDegrees, default 1 degree, controls only the diagnostic feature-edge
count. The result includes vertex, face and edge counts; boundary,
non-manifold and degenerate-face counts; manifold/closed/oriented/watertight
flags; signed and absolute volume; bounds; edge-use histogram; and face normals.

Use this to determine whether a host should enable an operation and to assert
the output policy after an operation. It diagnoses but does not repair.

### validateMesh(mesh, options)

Runs analysis then throws EdgecraftValidationError when the explicit policy does
not hold. Options are:

| Option | Default | Effect |
|---|---:|---|
| requireClosed | false | Rejects boundary edges |
| requireOriented | false | Rejects inconsistent shared-edge direction |
| allowDegenerate | false | Allows zero-area/repeated-index triangles |
| allowNonManifold | false | Allows edges used by over two faces |

Edge finishing requires a closed, oriented shell by default because it clips
and caps a solid. Vertex finishing has looser initial requirements, but unsafe
or non-convex fans are still skipped.

### findFeatureEdges(mesh, options)

Returns actual geometric crease records, independently of STL triangulation
diagonals. minimumCreaseDegrees defaults to 1. Boundary, non-manifold, and flat
records are hidden unless the corresponding include option is true. Each record
includes a canonical small-index:large-index key, endpoint IDs, incident face
IDs, length, crease angle, classification, and operable flag.

An operable record has two faces, is convex, and meets the angle threshold.
Concave, open, non-manifold, and flat edges can be shown in a host UI for
diagnostics, but the current finishing engine will not modify them.

### applyFinish(mesh, options)

This is the single production entry point for geometry modification.

| Option | Required | Meaning |
|---|---:|---|
| target | yes | edge or vertex |
| selection | yes | Edge keys/pairs/objects, or vertex indices |
| depth | yes | Positive model-space cut depth |
| operation | no | chamfer, fillet, Flatillet, or Chamfillet |
| angle | no | Edge chamfer bias in degrees |
| segments | no | Profile resolution; constrained to 1–64 |
| rounding | no | Chamfillet shoulder amount, 0–1 |
| clampDepth | no | Default true; reduces unsafe local cuts |
| output | no | plain (default) or vector3 |
| validateInput | no | Default true; skip only after host validation |
| includeAnalysis | no | Includes result analysis in the return value |

Edge targets accept chamfer, fillet, and Chamfillet. Vertex targets also accept
Flatillet. Edge angle is internally constrained to 5–85 degrees. A result is
an object containing vertices, faces, stats, and optional analysis. Stats report
processed targets, skipped targets, clamped cuts, and edge profile-plane count.

If clampDepth is false, a profile that would intersect nearby geometry throws.
If true, the kernel conservatively reduces the relevant cut group. A fatal
construction failure returns no partial mesh: retain the input transaction.

### createEdgecraftBackend(defaultOptions)

Creates a frozen service facade for dependency injection. It contains version,
analyze, validate, featureEdges, and apply methods. Per-call options override
defaults. The facade stores no mesh state and is safe to share across workers
or requests.

    const service = createEdgecraftBackend({ segments: 12, clampDepth: true });
    const result = service.apply(mesh, {
      target: "edge", selection: ["4:5"], operation: "fillet", depth: 2
    });

## STL I/O

parseStl detects ASCII or binary STL, parses repeated triangle corners, then
welds them into indexed topology. It returns the mesh with format and
sourceTriangleCount metadata. weldTolerance defaults to 0.00001, degenerate
triangles are removed by default, and the default maximum is 20 million
triangles. Set a tolerance greater than file noise but smaller than intentional
gaps.

weldTriangles performs just the welding stage for triangles supplied by another
importer. encodeBinaryStl returns an ArrayBuffer; encodeAsciiStl returns text;
encodeStl dispatches by requested format. Normals are recalculated by default
because STL stores redundant per-triangle normals.

## Errors

Input errors are EdgecraftValidationError values with code and details. Common
codes are INVALID_MESH_SHAPE, EMPTY_MESH, INVALID_VERTEX_SELECTION,
INVALID_EDGE_SELECTION, INVALID_OPERATION, INVALID_DEPTH, OPEN_MESH,
INCONSISTENT_WINDING, DEGENERATE_FACES, and NON_MANIFOLD_MESH.

Unexpected construction errors are EdgecraftOperationError values with code
OPERATION_FAILED, cause, and target/operation context. The host should leave
the document unchanged, report the message, and offer a smaller depth, full
feature-chain selection, or source-mesh repair.
