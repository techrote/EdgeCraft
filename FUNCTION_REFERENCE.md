# Complete function reference

This document maps every top-level named function in the current source tree,
plus the important internal helper layer. Tiny single-use closures such as a
local addVertex or cap-loop callback are described by their owning reconstruction
function rather than treated as standalone contracts. Public entries are
integration contracts. Private entries are documented so maintainers can trace
the algorithm, but they are not compatibility promises.

## Aggregate backend

backend.js has no unique algorithm. It re-exports the stable core API and STL
I/O as the portable public module.

## mesh-data.js

| Function/class | Visibility | Purpose |
|---|---|---|
| EdgecraftValidationError | public | Input-policy error carrying stable code and details |
| edgeSelectionKey | public | Validates two unequal non-negative indices and makes canonical sorted key |
| parseEdgeSelectionKey | public | Parses canonical key or returns null for malformed input |
| toVector3 | public | Converts tuple, x/y/z object, or Vector3 into a finite cloned Vector3 |
| toVectorMesh | public | Validates indexed triangular mesh and returns defensive vector arrays |
| toPlainMesh | public | Converts accepted mesh into JSON-safe tuples and copied faces |
| triangleNormal | public | Returns unit cross-product normal, or zero for degenerate triangle |

## core-api.js

| Function/class/constant | Visibility | Purpose |
|---|---|---|
| EDGECRAFT_CORE_VERSION | public | Backend compatibility label |
| FinishTarget | public | Frozen edge and vertex target constants |
| FinishOperation | public | Frozen chamfer, fillet, Flatillet, Chamfillet constants |
| EdgecraftOperationError | public | Wraps unexpected geometry failure with context |
| collectEdgeRecords | private | Scans faces into edge-use records, winding balance, and normals |
| findFeatureEdges | public | Classifies geometric feature/flat/boundary/non-manifold edges |
| analyzeMesh | public | Calculates topology flags, bounds, volume, normals, and counts |
| validateMesh | public | Applies required topology policy and throws validation errors |
| normalizeVertexSelection | private | Validates and deduplicates vertex IDs |
| normalizeEdgeSelection | private | Accepts key/pair/object selections and canonicalises edge keys |
| applyFinish | public | Validates request, invokes engine, formats output, wraps errors |
| createEdgecraftBackend | public | Builds stateless bound facade for injection and workers |

## stl-io.js

| Function | Visibility | Purpose |
|---|---|---|
| bytesFrom | private | Normalises text, ArrayBuffer, and typed-array input to bytes |
| formatMesh | private | Selects tuple or Vector3 result representation |
| weldTriangles | public | Spatially welds repeated corners and optionally drops degenerates |
| parseStl | public | Detects binary/ASCII STL, parses triangles, then welds them |
| encodeBinaryStl | public | Writes STL header, triangles, recalculated normals, zero attributes |
| encodeAsciiStl | public | Produces readable precision-controlled ASCII STL |
| encodeStl | public | Dispatches to the chosen binary or ASCII encoder |

## mesh-ops.js: topology and vertex helpers

| Function | Visibility | Purpose |
|---|---|---|
| edgeKey | private | Local sorted undirected edge-key helper |
| buildTopology | private | Creates face normals, vertex faces/neighbours, and edge face links |
| angularNeighborOrder | private | Sorts neighbour directions around a supplied vertex normal |
| orderedVertexNeighbors | private | Builds cyclic vertex fan from topology with angular fallback |
| uniqueIncidentNormals | private | Removes near-duplicate support normals |
| featureNeighbors | private | Selects significant crease directions in a vertex fan |
| planarPolygonCentroid | private | Finds robust centroid of planar cut loop |
| maximumOutwardTravel | private | Limits vertex cap apex before crossing support faces |
| edgeBiasedCurveParameter | public | Cosine maps ring progress toward both curve-boundary edges |
| sampledBoundaryLoop | private | Resamples cut loop by edge length for compound cap patches |
| finishVertices | public/advanced | Rebuilds selected convex vertex fans into requested patches |

## mesh-ops.js: edge profile and clipping helpers

| Function | Visibility | Purpose |
|---|---|---|
| parseEdgeKey | private | Permissive low-level edge-key parser |
| normalSlerp | private | Safe spherical interpolation between support normals |
| solveOffsetCenter | private | Solves tangent profile centre from two support-plane constraints |
| profilePlanesForRun | private | Generates all chamfer/fillet/Chamfillet support planes |
| canonicalDirection | private | Removes arbitrary sign from normalised line direction |
| vectorSignature | private | Quantises vector into stable grouping signature |
| runSignature | private | Groups triangulated edges sharing one line and support pair |
| clipPolygonToPlane | private | Clips one polygon against one half-space |
| triangulatePlaneLoop | private | Caps a planar loop with a centroid fan |
| compactMesh | private | Removes unused generated positions and remaps faces |
| clipClosedMeshByPlane | private | Clips full shell, welds intersections, finds/caps boundary loops |
| finishEdges | public/advanced | Reconstructs selected convex geometric crease runs |
| meshEdgeUseCounts | public | Returns map of undirected edge use count for diagnostics |

## edgecraft.js browser controller

The controller is a UI layer, not a backend API. Geometry command functions call
the public applyFinish facade; the archived comment block is not live code and
exists only as source-history context.

| Group | Functions | Responsibility |
|---|---|---|
| Preferences and themes | readPreference, writePreference, applyUiTheme, applyRenderTheme, setThemeMenuOpen, installParameterTooltips | Persists local UI choices and creates accessible hover/focus explanations |
| Camera/layout | updateCamera, resizeRenderer, frameMesh, boundsOf | Maintains orbit camera and viewport sizing |
| Formatting/feedback | formatNumber, formatCount, toast | Formats values and reports non-blocking operation status |
| Mesh/editor state | edgeKey, cloneMeshData, demoMesh, buildTopology, setMesh, restoreSnapshot | Maintains UI-local topology, display state, and snapshots |
| Scene resources | disposeObject, meshGeometry, pointsGeometry, edgePair, edgeLinePositions, fatEdgeHighlight, updateHighlightResolutions, setHighlightDepthTest, rebuildSceneMesh, refreshSelectionObjects, applyDisplayState | Creates/disposes Three geometry and wide contrast-line overlays |
| Panels | updateMeshStats, updateSelectionUI, setSelectionMode, updateOperationControls, updateHistoryUI, updateDepthWarning, pairRange | Synchronises controls and side-panel information |
| Picking | pointerToNdc, projectedDistanceSq, pickVertex, projectedEdgeDistanceSq, isOperableFeatureEdge, pickEdge | Projects mesh primitives and chooses click/hover candidates |
| Chain gestures | featureNeighbors, directionFrom, chooseStartDirections, walkChain, selectChain, walkEdgeChain, selectEdgeChain, handleVertexClick, handleEdgeClick, handleSelectionClick | Implements click, Alt chain, Ctrl add, Shift remove behaviour |
| Navigation | endNavigation | Clears drag state on pointer release, capture loss, blur, visibility loss, and lost middle-button state |
| File I/O | loadFile, binaryStlBlob, asciiStlBlob, exportStl, downloadStandaloneApp | Uses public codecs for local import/export and saves the standalone app |
| Commands | undo, redo, applyVertexOperation, applyEdgeOperation, applySelectionOperation | Snapshot management and delegation to public finish API |
| Rendering | animate | Schedules frame rendering |

## How to navigate code changes

For a reported operation defect, begin at applyFinish, then follow target to
finishEdges or finishVertices. For import/export defects, begin at parseStl or
encodeStl. For feature picking problems, begin at findFeatureEdges in the core
and then the controller pick/chain group. A UI-only change should never require
modifying mesh-ops.
