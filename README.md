# @casys/mcp-dfm

Measured design-for-manufacturability checks for STEP geometry, exposed as an
MCP server. The server snapshots the exact STEP bytes, runs native geometry
analysis, and compares the resulting measurements with limits supplied by the
caller.

It does not declare a part "manufacturable" or select a printer, material,
orientation, or threshold.

| Tool                      | What it measures                                                      | Caller-declared comparison                                  |
| ------------------------- | --------------------------------------------------------------------- | ----------------------------------------------------------- |
| `dfm_check_envelope`      | Axis-aligned X/Y/Z extents, surface-mesh volume status, optional mass | `build_volume_mm: { x, y, z }` and optional `density_kg_m3` |
| `dfm_check_overhangs`     | Downward-facing triangle area and spatial zones                       | `build_direction` and `max_overhang_deg`                    |
| `dfm_check_min_thickness` | Sampled inward-normal wall thickness and thin zones                   | `min_thickness_mm`                                          |

Every registered check consumes an absolute `step_path`. Supplying
`expected_step_sha256` makes the call fail before Gmsh runs if the private input
snapshot does not match the expected bytes. Geometry is reported in mm, mm², and
mm³; angles are degrees; density is kg/m³; mass is kg.

## Measured DFM is not a documentary printability estimate

These tools execute against a STEP snapshot. An empty `violations` array means
only that the measured values did not cross the limits declared for that call.
It is not a general printability or supplier-acceptance verdict.

In the Casys Digital Thread integration, the measured
`industrialize.run-dfm-checks@1` path is deliberately separate from the older
documentary `industrialize.observe-printability@1` path. The measured case binds
an attested STEP, the three-axis `build_volume_mm` object, and any downstream
bed-contact filter. Results from the two paths are not interchangeable.

## Mesh evidence and bounded measurement states

Every result includes `mesh_topology` for the particular Gmsh surface mesh
consumed by that call: `closed`, `watertight`, `manifold`,
`orientation_consistent`, connected component count, and the detected boundary,
non-manifold, and degenerate counts. These are properties of the emitted
triangle mesh, not a claim that the source CAD is valid or that a part is
manufacturable. `manifold` checks both edge uses and each vertex link;
`watertight` is true only when that mesh is closed, manifold, consistently
oriented, and has no degenerate triangles.

`dfm_check_envelope` always returns the numerical divergence-theorem result, but
sets `measured.volume_status` to `computed` only for one connected watertight
shell. Its absolute value is invariant if that one shell is globally reversed,
and no second shell exists whose nesting could change the material volume.
Multiple closed components remain `unverified`: this server does not infer their
relative orientation or nesting. Derived `mass_kg` is emitted only when a caller
supplies density and that volume status is `computed`; otherwise `mass_status`
is `not_requested` or `unverified`.

`dfm_check_min_thickness` also returns `ray_coverage`: the sampled triangle
centres, rays that found an opposing surface, unresolved rays, and whether
coverage is complete. `minimum_thickness_status` is `unverified` unless the mesh
is watertight and every sampled ray completed. A `sampled` result is still an
inward-normal sample, not a proof of the global minimum wall thickness.

## Quick start: Docker image over HTTP

Version `0.4.1` packages Gmsh, Python, and NumPy for `linux/amd64` and
`linux/arm64`. The Docker workflow derives its OCI version from `deno.json` and
accepts a semantic image tag only when the pushed Git tag matches the package
version. The command below is the last published `0.4.0` image digest, kept
until the `0.4.1` image exists; it is not the 0.4.1 image. After that release
workflow completes, pin the new GHCR digest in a deployment manifest.

```bash
docker run --rm \
  -p 127.0.0.1:3018:3018 \
  -v /absolute/path/to/step-files:/data:ro \
  ghcr.io/casys-ai/mcp-dfm@sha256:564d3745fd4e0dd35c6eb6a3479292df83362d8a82d00d5812daaff8f23cbe9b http
```

The image's `http` mode provides stateless HTTP on `http://127.0.0.1:3018/mcp`,
protocol `2026-07-28`. Use paths as seen by the container, such as
`/data/bracket.step`, not host paths. Docker Desktop must be allowed to share
the mounted directory where applicable.

## HTTP call example

The following source-checkout call measures the committed 40 × 30 × 20 mm
fixture. `Mcp-Name` must mirror `params.name`:

```bash
curl -sS -X POST http://127.0.0.1:3018/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H 'MCP-Protocol-Version: 2026-07-28' \
  -H 'Mcp-Method: tools/call' \
  -H 'Mcp-Name: dfm_check_envelope' \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "dfm_check_envelope",
      "arguments": {
        "step_path": "/data/dfm_healthy_box.step",
        "expected_step_sha256": "0eeade1298710e8ceff66c52f2b7e51bbfb2b3856fafd955fd2cfa6107492693",
        "build_volume_mm": { "x": 200, "y": 200, "z": 200 },
        "mesh_size_mm": 5,
        "density_kg_m3": 2700
      },
      "_meta": {
        "io.modelcontextprotocol/protocolVersion": "2026-07-28",
        "io.modelcontextprotocol/clientCapabilities": {}
      }
    }
  }'
```

The measured result includes:

```json
{
  "violations": [],
  "measured": {
    "x_mm": 40,
    "y_mm": 30,
    "z_mm": 20,
    "volume_mm3": 23999.999999999967,
    "volume_status": "computed",
    "mass_status": "computed",
    "mass_kg": 0.06479999999999991
  },
  "mesh_topology": {
    "closed": true,
    "watertight": true,
    "manifold": true,
    "orientation_consistent": true,
    "connected_component_count": 1,
    "boundary_edge_count": 0,
    "non_manifold_edge_count": 0,
    "non_manifold_vertex_count": 0,
    "degenerate_triangle_count": 0
  },
  "limits_declared": {
    "build_volume_mm": { "x": 200, "y": 200, "z": 200 },
    "density_kg_m3": 2700
  },
  "input_artifact": {
    "sha256": "0eeade1298710e8ceff66c52f2b7e51bbfb2b3856fafd955fd2cfa6107492693",
    "bytes": 15456,
    "source_path": "/data/dfm_healthy_box.step"
  }
}
```

Use `structuredContent` as the machine-readable result. The text `content` is a
short summary for the model. Floating-point values are not rounded in the wire
result, so consumers should apply tolerances appropriate to their case.

`ghcr.io/casys-ai/mcp-dfm:latest` is a mutable convenience tag, not the
authority for a version or capability. The digest in the quick-start command is
the last published `0.4.0` image. Resolve the `0.4.1` image to its GHCR digest
after that image exists.

## Tool contracts

Every registered input object is closed (`additionalProperties: false`). Unknown
properties are rejected both by MCP schema validation and when a handler is
invoked directly. Numeric sizes and dimensions must be finite and strictly
positive: build-volume X/Y/Z, `mesh_size_mm`, optional `cluster_radius_mm`,
optional `density_kg_m3`, `min_thickness_mm`, and optional `timeout_ms`.
`sample_count` must be a finite positive integer. `max_overhang_deg` remains
0–90° inclusive; `build_direction` must be a non-zero three-vector with finite
components. These checks run before the STEP snapshot or any native subprocess.
The server still does not invent manufacturing defaults or arbitrary maximum
caps.

### `dfm_check_envelope`

Gmsh tessellates the STEP into a surface STL. The tool computes the bounding box
from vertex extrema, computes volume using the divergence theorem, and computes
mass only when density is supplied:

`mass_kg = volume_mm3 / 1e9 × density_kg_m3`

| Field                  | Required | Description                                                                                 |
| ---------------------- | -------- | ------------------------------------------------------------------------------------------- |
| `step_path`            | yes      | Absolute path to the STEP file on the server filesystem                                     |
| `build_volume_mm`      | yes      | Object with strictly positive X, Y, and Z extents in mm: `{ "x": 200, "y": 200, "z": 200 }` |
| `mesh_size_mm`         | yes      | Finite strictly positive Gmsh surface element size in mm                                    |
| `expected_step_sha256` | no       | Expected 64-character STEP SHA-256; mismatch aborts before Gmsh                             |
| `density_kg_m3`        | no       | Finite strictly positive caller-owned density used to add `mass_kg`; no density is inferred |
| `timeout_ms`           | no       | Finite strictly positive Gmsh subprocess timeout in ms; default 60000                       |

The comparison is axis by axis in the STEP coordinate system. The tool does not
rotate the part to find a tighter fit. A numerical `volume_mm3` is kept for
diagnosis, but it is explicitly `unverified` unless the returned mesh topology
establishes one connected watertight shell; derived mass is withheld on that
condition.

Measured native fixture result at `mesh_size_mm: 5`: 40.0 × 30.0 × 20.0 mm and
approximately 24000.0 mm³.

### `dfm_check_overhangs`

For each tessellated triangle, the tool measures the angle between its outward
normal and `-build_direction`. The server normalizes any non-zero three-vector.
With `[0, 0, 1]`, 0° points straight down, 90° is vertical, and 180° points
straight up. A triangle is included when its angle is strictly less than
`max_overhang_deg`; matching triangles are clustered into zones.

| Field                  | Required | Description                                                               |
| ---------------------- | -------- | ------------------------------------------------------------------------- |
| `step_path`            | yes      | Absolute path to the STEP file                                            |
| `build_direction`      | yes      | Non-zero finite `[x, y, z]` build direction; `[0, 0, 1]` is +Z            |
| `max_overhang_deg`     | yes      | Finite threshold in degrees from the downward direction, 0–90 inclusive   |
| `mesh_size_mm`         | yes      | Finite strictly positive Gmsh surface element size in mm                  |
| `expected_step_sha256` | no       | Expected STEP SHA-256                                                     |
| `cluster_radius_mm`    | no       | Finite strictly positive spatial merge radius; default `3 × mesh_size_mm` |
| `timeout_ms`           | no       | Finite strictly positive Gmsh timeout in ms; default 60000                |

Important bed-contact behavior: the provider does not remove the bottom face.
For the healthy-box fixture at +Z, the returned zone is the 1200 mm² face at Z =
-10 mm, with `bbox.min[2]` and `bbox.max[2]` both equal to -10. The output does
not separately declare the part's global minimum Z, and there is no `z_min`
input. A workflow may remove a bed-contact zone only from an explicit, reviewed
plane and tolerance; it must not silently infer one from the lowest reported
zone.

The tool also does not model support placement or cost, bridging rules, or
material-specific behavior. Curved surfaces are approximated by the mesh.

Measured L-bracket fixture result at `mesh_size_mm: 3`, 45°, build +Z: overhang
area greater than 1000 mm² and at least one zone, including the bed face.

### `dfm_check_min_thickness`

The tool tessellates with Gmsh, samples triangle centres, and invokes Python
with NumPy to cast an inward-normal Möller-Trumbore ray. It reports the first
opposing surface distance, the minimum sampled distance, sample coverage, and
clustered points below the declared threshold.

| Field                  | Required | Description                                                                  |
| ---------------------- | -------- | ---------------------------------------------------------------------------- |
| `step_path`            | yes      | Absolute path to the STEP file                                               |
| `min_thickness_mm`     | yes      | Finite strictly positive caller-declared violation threshold                 |
| `mesh_size_mm`         | yes      | Finite strictly positive Gmsh element size; start at most half the threshold |
| `expected_step_sha256` | no       | Expected STEP SHA-256                                                        |
| `sample_count`         | no       | Finite positive integer triangle centres to sample; default 500              |
| `cluster_radius_mm`    | no       | Finite strictly positive spatial merge radius; default `3 × mesh_size_mm`    |
| `timeout_ms`           | no       | Finite strictly positive total Gmsh and Python timeout in ms; default 120000 |

This is sampled inward-normal thickness, not a global exact minimum-distance
proof. Sampling can miss a small or diagonal thin region. The result reports the
exact mesh topology and ray coverage it obtained; `minimum_thickness_status` is
`unverified` when the mesh is not watertight or any sampled ray did not reach an
opposing surface. It does not apply FDM, SLA, SLS, or material-specific feature
rules.

Measured thin-wall fixture result at `mesh_size_mm: 0.5`, `sample_count: 300`,
and threshold 1.0 mm: minimum 0.8000 mm and 12 sampled violations.

## Input attestation

Before invoking a native process, each tool:

1. Copies `step_path` into a private temporary directory.
2. Hashes that private copy with SHA-256.
3. Compares it with `expected_step_sha256`, when supplied.
4. Makes the snapshot read-only.
5. Passes only the snapshot to Gmsh and returns its digest, byte count, and
   original source path in `input_artifact`.

The returned digest therefore identifies the bytes actually consumed. The
expectation is optional in the raw MCP schema so exploratory calls are possible;
provenance-sensitive workflows should require it.

## Run from source or JSR

The HTTP server requires native executables on `PATH`:

| Dependency             | Used by                | Typical install                                                                    |
| ---------------------- | ---------------------- | ---------------------------------------------------------------------------------- |
| `gmsh`                 | every registered check | `apt install gmsh` or `brew install gmsh`                                          |
| `python3` with `numpy` | minimum-thickness only | `apt install python3 python3-numpy` or an activated virtual environment with NumPy |

From a checkout:

```bash
deno task serve
deno task serve -- --port=3099 --hostname=0.0.0.0
```

Version `0.4.1` provides both stateless HTTP and native stdio. For stateless
HTTP through the JSR package:

```bash
deno run -A jsr:@casys/mcp-dfm@0.4.1/server --port=3018
```

The first two commands expose stateless HTTP.

### Native stdio from a checkout, JSR, or published image

Version `0.4.1` provides native stdio. Use an exact entrypoint:

```bash
# checkout
deno run -A server.ts --stdio

# JSR 0.4.1
deno run -A jsr:@casys/mcp-dfm@0.4.1/server --stdio
```

For native stdio from the published image, pass `stdio` to Docker and keep stdin
open with `-i`:

```bash
docker run --rm -i \
  -v /absolute/path/to/step-files:/data:ro \
  ghcr.io/casys-ai/mcp-dfm@sha256:564d3745fd4e0dd35c6eb6a3479292df83362d8a82d00d5812daaff8f23cbe9b stdio
```

## MCP App viewer

Version `0.4.1` includes the MCP App `io.casys.mcp-dfm.results@0.4.1`. The App
is a read-only projection of:

- a closed recorded session `io.casys.mcp-dfm.recorded-checks-session/1.0`
  delivered through `viewer.session.apply` (the Digital Thread path);
- optional single-tool raw results for the three existing checks, with no
  invented whole-case verdict.

It does not execute Gmsh, recompute evaluations, or declare a part
manufacturable. Recorded Digital Thread evaluations are labelled as Digital
Thread-owned. Quality fields that a historical `DfmCheckCapture` does not record
(`mesh_topology`, `volume_status`, `minimum_thickness_status`, `ray_coverage`)
display as unavailable. Private `stagedPath` / `source_path` values are stripped
or rejected.

The HTML bundle is part of the package layout. Rebuilding it uses the published
MCP View source commit `b08802df353bb25d25a1c8d64b22ea61b5287ae0`, with split
packages (`@casys/mcp-view@0.9.3`, `@casys/mcp-view-contracts@0.1.0`,
`@casys/mcp-view-components@0.9.0`) and does not fall back to a published SDK:

```bash
git clone https://github.com/superWorldSavior/mcp-platform.git .build/mcp-server
git -C .build/mcp-server checkout b08802df353bb25d25a1c8d64b22ea61b5287ae0
export MCP_VIEW_LOCAL_ROOT="$PWD/.build/mcp-server/packages/view"
export MCP_VIEW_CONTRACTS_LOCAL_ROOT="$PWD/.build/mcp-server/packages/view-contracts"
export MCP_VIEW_COMPONENTS_LOCAL_ROOT="$PWD/.build/mcp-server/packages/view-components"
deno task build:ui
deno task check:ui:bundle
```

The package and App identities are versioned together. The Docker quick-start
above retains the previously published 0.4.0 image digest; resolve the new 0.4.1
image after its release workflow succeeds. Consumer runtime qualification and
adoption remain separate from publishing this provider release.

## Development

```bash
deno task release:check
DFM_RUN_NATIVE=1 deno task test
```

`release:check` runs formatting, type checking, linting, non-native tests, the
stdio wire tests, viewer model tests, and the versioned UI bundle freshness
gate. The reusable CI quality workflow runs that fast job without
`DFM_RUN_NATIVE`; its native job installs Gmsh, Python, and NumPy, then runs
`DFM_RUN_NATIVE=1 deno task test`. A pull-request workflow calls the same
quality workflow, which must complete before JSR publication or GHCR image build
and push.

A workflow publishes a new JSR version only when the version in `deno.json` is
not already present. A separate workflow publishes the multi-arch GHCR image; a
semantic image tag is accepted only when it matches that package version.

## Security

This server invokes native parsers on caller-supplied files. Keep HTTP bound to
loopback unless it is protected by an appropriate trusted boundary. See
[`SECURITY.md`](SECURITY.md) for private vulnerability reporting.

## License

MIT
