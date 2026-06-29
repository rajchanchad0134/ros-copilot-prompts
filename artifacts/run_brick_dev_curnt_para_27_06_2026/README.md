# Run: run_brick_dev_curnt_para_27_06_2026

Output artifacts from a full co-pilot pipeline run on the **Brick-by-Brick** ROS 2 system (27 June 2026).

---

## Artifact Classification

### Intermediate Artifacts

These files are produced by one pipeline phase and consumed as input by the next. They are not the end goal but are essential for traceability and debugging.

| File | Produced by | Consumed by | Purpose |
|------|-------------|-------------|---------|
| `src_all_nodes.json` | Phase 1 — `ros_code_analyst` | Phase 2a — `node_ports_extractor` | Lists every ROS 2 node class found in the source package with its file paths, runtime name, and description |
| `plantuml_diagrams/brickbybrick_nodes/node_*.json` | Phase 2a — `node_ports_extractor` | Phase 2b — `plantuml_diagram_generator` | Per-node ports JSON — every publisher, subscriber, service client, and service server with exact topic/service names and message types |
| `system_architecture/src_launch_deps.json` | Static analyzer (Python, no LLM) | Phase 3 pre — `preprocess_launch_analyzer` | Launch file include tree — which `.launch.py` files exist and which ones include which others |
| `system_architecture/src_preprocess_launch.json` | Phase 3 pre — `preprocess_launch_analyzer` | Phase 3a — `system_architecture_json_extractor` | Structured launch topology — all node instances with their runtime names, namespaces, and which launch file they come from |
| `system_architecture/src_architecture.json` | Phase 3a — `system_architecture_json_extractor` | Phase 3b — `system_architecture_plantuml_generator` | Complete flat architecture JSON — all nodes with resolved namespaces, all topics and services with message types, all connections marked |

---

### Final Results

These are the human-readable outputs the co-pilot is designed to produce. They require no further processing.

| File | Produced by | What it shows |
|------|-------------|---------------|
| `plantuml_diagrams/brickbybrick_nodes/node_*.puml` | Phase 2b — `plantuml_diagram_generator` | Per-node component diagram in PlantUML source — shows each node's publishers (green), subscribers (blue), service clients, and service servers with exact topic/service names |
| `plantuml_diagrams/brickbybrick_nodes/node_*.png` | Auto-rendered from `.puml` | Rendered PNG image of each per-node component diagram — ready to embed in documentation |
| `system_architecture/system_architecture.puml` | Phase 3b — `system_architecture_plantuml_generator` | Full system architecture in PlantUML source — all 9 nodes, all topics as queues, all services as circles, all connections, grouped by namespace |
| `system_architecture/system_architecture.png` | Auto-rendered from `.puml` | **The primary output** — single PNG showing the complete Brick-by-Brick ROS 2 runtime architecture |

---

## Nodes in This Run

9 nodes extracted from the `brickbybrick` package:

| Node Class | Role |
|-----------|------|
| `AliceConveyer` | Controls the conveyor belt for brick transport |
| `CoordinateTransformerNode` | Transforms coordinates between reference frames |
| `DisassemblyOrchestrator` | High-level orchestration of the disassembly sequence |
| `LegoDetectorNode` | Detects LEGO bricks using vision |
| `ObjectLocalizerNode` | Localizes detected objects in 3D space |
| `PickPlannerNode` | Plans pick-and-place motions |
| `PositionEndingSequence` | Executes the ending position sequence |
| `PositionStartingSequence` | Executes the starting position sequence |
| `RealSenseCamera` | Interfaces with the Intel RealSense camera |
| `Supervisor` | Top-level supervisor node managing overall system state |

---

## Pipeline Flow for This Run

```
brickbybrick source code
        │
        ▼
[Phase 1]  src_all_nodes.json                            ← INTERMEDIATE
        │
        ▼
[Phase 2a] node_*.json  (one per node)                   ← INTERMEDIATE
        │
        ▼
[Phase 2b] node_*.puml + node_*.png  (one per node)      ← FINAL RESULT
        │
        ▼ (also feeds in: launch files)
[Phase 3 pre] src_launch_deps.json                       ← INTERMEDIATE
              src_preprocess_launch.json                 ← INTERMEDIATE
        │
        ▼
[Phase 3a] src_architecture.json                         ← INTERMEDIATE
        │
        ▼
[Phase 3b] system_architecture.puml                      ← FINAL RESULT
           system_architecture.png                       ← FINAL RESULT (primary)
```
