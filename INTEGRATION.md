# Integrating EdgeCraft into a larger CAD package

## Recommended role

Treat EdgeCraft as a stateless mesh-feature service. The host owns the
document, units, undo history, persistent identities, rendering, constraints,
and user interface. EdgeCraft accepts an immutable mesh snapshot plus a finish
request and returns one replacement mesh with structured diagnostics.

This is suitable for mesh-native CAD, slicer preparation, scan cleanup, browser
CAD, and exact CAD applications after tessellation. It is not currently the
authoritative editor for a parametric B-rep tree because it has no analytic
surfaces or persistent topological names.

## Write one host adapter

Create a narrow boundary adapter between host mesh objects and the indexed
plain-mesh contract. Convert positions and triangles once, do not pass mutable
host buffers directly, and create a fresh host mesh from the result.

    const source = hostMeshToPlain(await document.meshSnapshot(bodyId));
    const before = analyzeMesh(source);
    if (!before.watertight && request.target === "edge") {
      throw new Error("Repair the shell before applying an edge finish.");
    }
    const result = applyFinish(source, { ...request, includeAnalysis: true });
    if (!result.analysis.watertight) throw new Error("Rejecting invalid result.");
    return plainToHostMesh(result);

Declare units at this boundary. EdgeCraft copies numeric values unchanged, so
host millimetres give millimetre depth and output. Establish a production policy
for minimum useful depth and required output quality before calling the core.

## Selections and persistent identity

Finishing reconstructs and compacts topology, so numeric vertex/face indices
apply only to the exact source snapshot. Never store an EdgeCraft index as a
permanent CAD subshape identity. Store a transaction record containing:

| Field | Purpose |
|---|---|
| body revision/checksum | Detects topology-changing edits |
| target | Edge or vertex |
| temporary selection | Canonical edge key or vertex index for the source snapshot |
| geometric anchor | Vertex position, or edge endpoints/midpoint/direction |
| feature signature | Adjacent normals, edge length, or neighbourhood |
| parameters | Operation, depth, angle, segments, rounding, clamp policy |

To replay against a changed body, first rediscover the matching geometric
feature within host-defined position and normal tolerances. If this is
ambiguous, require user confirmation rather than guessing.

## Transaction, preview, and undo

Run a finish as one immutable command:

1. Snapshot source mesh and document revision.
2. Analyse and validate it.
3. Execute the operation in a worker.
4. Analyse the returned mesh and enforce host acceptance policy.
5. Commit the precise returned mesh as one reversible command.
6. Undo restores the source snapshot; redo restores the recorded result.

Do not recompute an operation on redo: a changed mesh or changed tolerance could
produce a different result. Keeping both immutable snapshots prevents numerical
drift and makes failures naturally transactional.

For interactive sliders, debounce preview work and tag each request with a
revision/token. Discard stale responses. A host may preview with fewer segments
for responsiveness, but must label it approximate and run the requested final
resolution before commit.

## Browser and desktop worker pattern

The portable backend can be imported directly by a module Worker:

    import { applyFinish } from "./edgecraft-backend.mjs";

    self.onmessage = ({ data: { id, mesh, options } }) => {
      try {
        const result = applyFinish(mesh, { ...options, includeAnalysis: true });
        self.postMessage({ id, ok: true, result });
      } catch (error) {
        self.postMessage({
          id, ok: false,
          error: { name: error.name, code: error.code, message: error.message, details: error.details }
        });
      }
    };

Tuple arrays are structured-clone compatible. For very large models, an outer
host adapter can convert typed buffers at the worker boundary; the public
contract intentionally favours clear JSON-compatible data over sharing mutable
buffers. In Node/Electron, import the package normally and use a Worker Thread
or isolated process for large jobs.

For public uploads, retain maximum-triangle limits, impose a host file-size
limit, and isolate memory-heavy parsing. Do not publish an unbounded STL parser
as a request handler.

## UI division of responsibility

The supplied editor demonstrates the intended separation:

| Layer | Responsibilities |
|---|---|
| CAD host/UI | Picking, edge/vertex chains, highlighting, camera, themes, widgets, user help, document history |
| EdgeCraft core | Validation, crease classification, local clearance, profile construction, reconstructed topology, statistics |
| Codec layer | STL parsing, corner welding, and STL formatting |

Replace the editor's browser picker with the CAD package's own selection system.
Provide selected vertex indices or canonical edge pairs to the core. Chains are
a UI convenience: the core accepts many selected edges and groups compatible
collinear portions into one geometric run itself.

## Exact-kernel migration path

The modular split supports an eventual exact CAD backend without changing UI
contracts:

1. Keep the request schema and host adapter.
2. Add a B-rep adapter mapping selections to persistent kernel subshapes.
3. Implement the same operations using native blends/chamfers.
4. Tessellate the result for the existing renderer/export path.
5. Return equivalent statistics/diagnostics from the service facade.

The present engine remains a portable fallback for mesh-only deployments; an
exact build can substitute the implementation behind the same backend factory.
