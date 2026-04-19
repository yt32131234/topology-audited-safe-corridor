# tunnel-corridor-v52-core

`tunnel-corridor-v52-core` is a research-oriented Python implementation of a topology-audited safe corridor planner for static 3D environments. It builds a discrete topological reference from clearance-qualified free space, constructs a multi-scale inscribed-sphere cover, separates the geometric nerve from the executable planning graph, performs topology-driven cover correction, and exposes certified corridor queries together with a final audit stage.

The repository is intended for researchers who want a compact and readable implementation of the core planning pipeline. It can be used directly on the built-in synthetic tunnel scenes or on external voxel maps stored in a simple `npz` format.

## What the Planner Does

At a high level, the planner executes the following stages:

1. Build a clearance-qualified free-space representation from an ESDF map.
2. Extract a discrete topological reference, including Betti numbers, witness cycles, and cut-surface signatures.
3. Generate an initial multi-scale inscribed-sphere cover on discrete candidate centers and quantized radii.
4. Correct the cover through topology-driven editing operations such as shrink, split, remove, and insert.
5. Construct the full geometric nerve for topology verification and a separate planning graph for executable safe-ball-overlap queries.
6. Run a final audit that checks topology consistency, witness realization, signature independence, and safe-edge continuity.
7. Return a certified corridor for either an unconstrained best-safe query or a class-constrained query with a prescribed topological signature.

## Main Components

- `Map3D`, `TopoReference`, `PlanGraph`, and `CorridorResult` data structures for map representation, topological reference, query graph construction, and certified output.
- Synthetic 3D scene generation for tunnels, loops, cavities, and false-merge stress cases.
- External `npz` map loading and saving utilities.
- Topological reference construction on clearance-qualified free space.
- Initial inscribed-sphere cover generation on a discrete candidate set.
- Topology-preserving cover correction and witness realization.
- Query graph construction with safe-ball-overlap filtering and class-preserving constraints.
- Final audit and certified corridor queries.

## Installation

Create or activate a Python environment with Python `3.10+`, then install the package in editable mode:

```bash
pip install -e .[dev]
```

Optional GPU support is available for the candidate edge screening path in the geometric nerve construction:

```bash
pip install -e .[gpu]
```

To enable the GPU candidate-edge backend at runtime:

```bash
set TCV52_COMPLEX_BACKEND=gpu
```

On Linux or macOS:

```bash
export TCV52_COMPLEX_BACKEND=gpu
```

## Quick Start

### List the built-in synthetic scenes

```bash
tunnel-corridor-core list-scenes
```

### Run the full pipeline on a synthetic scene

```bash
tunnel-corridor-core run --scene double_tunnel --grid 48 --resolution 1.0 --clearance 2.0
```

This command runs the complete planning pipeline and prints a JSON summary containing:

- the reference topology
- the number of spheres in the corrected cover
- planning-graph statistics
- audit status
- corridor query metrics

### Export a synthetic scene as an `npz` map

```bash
tunnel-corridor-core save-scene --scene double_loop --out demo_scene.npz
```

### Run the planner on an external `npz` map

```bash
tunnel-corridor-core run-external --npz your_map.npz --clearance 2.0
```

## Python API

The package can also be used directly from Python.

### Minimal example

```python
from tunnel_corridor_v52 import generate_scene, run_pipeline, summarize_run

map3d = generate_scene(
    "double_tunnel",
    grid_shape=(48, 48, 48),
    resolution=1.0,
    clearance=2.0,
)

run = run_pipeline(map3d)
summary = summarize_run(run)

print(summary["beta_ref"])
print(summary["audit"]["success"])
print(summary["corridor"]["quality_metrics"]["min_clearance"])
```

### Explicit corridor query

```python
import numpy as np

from tunnel_corridor_v52 import CorridorQuery, generate_scene, run_pipeline

map3d = generate_scene("double_tunnel", clearance=2.0)

query = CorridorQuery(
    start=np.asarray(map3d.metadata["audit_start"], dtype=np.float32),
    goal=np.asarray(map3d.metadata["audit_goal"], dtype=np.float32),
    min_clearance=2.0,
)

run = run_pipeline(map3d, query=query)
result = run["corridor"]

print(result.certificate.topo_signature)
print(result.quality_metrics.energy_proxy)
```

### Running from an external map

```python
from tunnel_corridor_v52 import run_pipeline_from_npz, summarize_run

run = run_pipeline_from_npz("your_map.npz", clearance=2.0)
summary = summarize_run(run)

print(summary["scene"])
print(summary["audit"]["success"])
```

## Built-in Synthetic Scenes

The package ships with a compact set of synthetic scenes that are useful for debugging and algorithmic demonstrations:

- `single_tunnel`
- `double_tunnel`
- `tunnel_cavity`
- `multi_room_loop`
- `double_loop`
- `false_merge_trap`

These scenes are generated procedurally as voxel maps with ESDF values and metadata describing reference graph nodes, portal pairs, and default audit queries.

## External Map Format

External maps are loaded from compressed `npz` files. A minimal file can contain either:

- `occupancy_grid`: a 3D boolean array where `True` denotes occupied voxels
- or `esdf_grid`: a 3D floating-point ESDF array

The file should also contain:

- `resolution`: scalar voxel size
- `origin`: a 3-vector world origin

Optional fields:

- `clearance_threshold`
- `metadata`

If only `occupancy_grid` is provided, the loader computes the ESDF automatically. If explicit reference-graph metadata is not present, the loader attempts to infer a usable audit start/goal pair and a compressed reference skeleton from the clearance-qualified safe region.

## Typical Workflow

For a new project or a new map, a practical workflow is:

1. Prepare a voxel map with either occupancy or ESDF values.
2. Verify the resolution and clearance threshold you want to plan against.
3. Run the planner once and inspect the JSON summary.
4. If needed, call the Python API to access the corrected sphere cover, topological reference, planning graph, and final corridor object.
5. Integrate the returned `CorridorResult` into downstream evaluation, visualization, or robotics pipelines.

## Project Layout

```text
Github/
├─ .gitignore
├─ pyproject.toml
├─ README.md
├─ src/
│  └─ tunnel_corridor_v52/
│     ├─ __init__.py
│     ├─ cli.py
│     ├─ models.py
│     ├─ pipeline.py
│     ├─ utils.py
│     ├─ audit/
│     │  ├─ __init__.py
│     │  └─ report.py
│     ├─ cover/
│     │  ├─ __init__.py
│     │  ├─ correction.py
│     │  └─ initial.py
│     ├─ graph/
│     │  ├─ __init__.py
│     │  ├─ complex.py
│     │  └─ query.py
│     ├─ map/
│     │  ├─ __init__.py
│     │  ├─ external.py
│     │  └─ sim.py
│     └─ topology/
│        ├─ __init__.py
│        └─ reference.py
└─ tests/
   ├─ conftest.py
   ├─ test_core_pipeline.py
   └─ test_io_and_cli.py
```

## Entry Points

- CLI entry point: `tunnel-corridor-core`
- Python package: `tunnel_corridor_v52`

The CLI is designed for quick execution and JSON summaries, while the Python API exposes the intermediate representations needed for research workflows.

## Notes for Research Use

This repository is written as a compact algorithmic implementation rather than a polished application stack. The code is therefore especially suitable for:

- method inspection
- ablation-oriented development
- synthetic-scene debugging
- external-map adaptation
- downstream integration into custom evaluation scripts

If you use the code in a paper, it is a good idea to document the map preprocessing path, the chosen clearance threshold, and the exact scene or map identifiers used in the experiments.
