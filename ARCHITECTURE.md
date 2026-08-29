# Architecture and geometry

## Design boundary

EdgeCraft is a triangle-mesh finishing kernel, not an exact B-rep or NURBS
kernel. It works on a closed piecewise-planar surface and returns another
triangle mesh. The system is deliberately layered:

STL or host mesh -> STL I/O and mesh-data adapter -> core API analysis,
validation and selection -> mesh operations -> replacement indexed mesh ->
STL encoder or CAD host adapter.

The browser editor uses the same STL I/O and core API for its own work. Only
core-api and stl-io are the supported integration boundary. mesh-ops is
exported for advanced users but is an implementation layer. The editor owns
selection gestures, camera, highlighting, themes, local file UX and history; it
calls applyFinish rather than retaining a separate copy of the geometry logic.

## Indexed topology

An STL repeats the corners of every triangle. The import layer first spatially
welds nearby corners, including points in neighbouring hash cells, yielding one
position per topological vertex and triangles of integer indices. Topology
construction then derives face normals, vertex-to-face links, neighbour sets,
and undirected edge-to-face links.

This is how the system separates a real rectangular crease from a hidden
coplanar STL diagonal. A coplanar diagonal has almost identical incident face
normals and is classified flat; it is not selected as a default feature.

## Core invariants

1. Input positions and face arrays are never mutated.
2. Emitted positions must be finite; repeated/zero-area triangles are omitted.
3. Output faces retain outward winding where an outward hint is available.
4. A successful finish of a closed, oriented input remains closed.
5. Local cuts must not cross nearby geometry: they clamp or fail.
6. Edge profile planes stay inside the original solid envelope.
7. Coplanar triangulation does not change an otherwise valid feature result.

The host should run analyzeMesh after a commit and enforce its own watertight
policy. EdgeCraft does not silently repair an invalid result.

## Edge reconstruction

### Geometric runs

Selected two-face convex creases are grouped by their underlying 3D line and
the pair of support-face normals. Consequently, a visible edge represented by
multiple triangulated subedges is treated as one run. Terminal support faces are
also identified so a Chamfillet can round supported ends instead of stopping
abruptly.

### Half-space profiles

An edge operation creates a sequence of support half-space planes. The kernel
clips the whole closed mesh successively: each step clips triangles, welds
intersections, identifies boundary loops on the plane, caps them, then compacts
the result.

| Operation | Profile |
|---|---|
| Chamfer | One planar cut between the two support faces; angle biases it from the equal 45 degree case |
| Fillet | Tangent plane sequence whose normals spherically interpolate between support normals |
| Chamfillet | A central chamfer plane, two tangent shoulder sequences, and suitable terminal shoulder planes |

Fillet centre positions are solved from tangent support constraints. Increasing
segments refines the same radius rather than growing a bulbous radius.
Chamfillet rounding scales shoulder radius, while retaining a measurable planar
chamfer section between them.

Before use, every plane group is checked against nearby original vertices.
Unsafe groups are uniformly reduced when clamping is enabled; otherwise the
request fails. If usable clearance collapses to almost nothing, the operation
fails rather than emitting bad triangles.

## Vertex reconstruction

Vertex finishing replaces a local incident-face fan. The engine gathers unique
support normals, establishes a weighted outward direction, orders neighbour
directions around that normal, rejects non-convex/unusable fans, and intersects
all incident edges with one shared cut plane. One shared plane makes the chamfer
symmetric and prevents individual triangle diagonals from pulling cut points.

Surrounding triangles become clipped polygons. The generated cut loop receives
one of the following patches:

| Operation | Patch |
|---|---|
| Chamfer | Planar loop triangulated from a planar centroid |
| Flatillet | Intentionally shallow convex spherical radial cap |
| Fillet | Compound spherical-cap approximation subdivided around and across the corner |
| Chamfillet | Central flat with rounded outward shoulders |

Chamfillet ring spacing uses cosine edge bias, concentrating facets near the
outer shoulder and the central-flat boundary instead of spending them through
the low-curvature middle. Support-plane travel bounds keep a cap from projecting
through its incident faces. Adjacent selected corners reserve roughly half of a
shared edge, with optional clamping for oversized requests.

## Tolerances and resolution

Clipping tolerances derive from mesh bounding-box diagonal, with a minimum
scale of one, so plane tests adapt to model size. STL welding uses its own
explicit tolerance because import noise is an application/data choice.

Segments are constrained to 1–64. More segments give a visually smoother
faceted approximation but not an analytic surface. Choose resolution based on
manufacturing/slicing tolerance rather than assuming large values are always
better.

## Current limitations

- This is not an exact-radius or G1/G2 B-rep feature engine.
- Edge finishing accepts only closed, consistently wound, convex two-face
  manifold creases.
- Concave, open, non-manifold, and flat edges are diagnostic-only.
- Non-convex or very irregular vertex fans can be skipped/rejected.
- Local clamping is conservative and cannot prove no distant self-intersection
  in arbitrary complex geometry.
- The result is a replacement mesh, not a persistent parametric feature.

For exact CAD semantics, use EdgeCraft after tessellation, or implement the
same high-level request contract against an exact kernel and retain this engine
as a portable mesh fallback.
