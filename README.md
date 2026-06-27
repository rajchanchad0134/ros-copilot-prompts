# ROS Architecture Co-Pilot — Prompt Library

This repository contains the three LLM prompt files that drive the **ROS Architecture Co-Pilot** pipeline. The co-pilot automatically extracts, analyzes, and visualizes the software architecture of a ROS 2 system from its source code — producing per-node component diagrams and a full system-level architecture diagram, all in PlantUML.

---

## Table of Contents

1. [System Overview](#system-overview)
2. [Full Pipeline Workflow](#full-pipeline-workflow)
3. [Agents and Their Roles](#agents-and-their-roles)
4. [Prompt Files](#prompt-files)
5. [How Prompts Are Fed to Agents](#how-prompts-are-fed-to-agents)
6. [Data Flow Diagram](#data-flow-diagram)

---

## System Overview

The co-pilot is built on **CrewAI** and runs five LLM agents across three sequential phases. Each phase takes the output of the previous one as input, progressively building a richer representation of the ROS 2 system:

```
Source Code
    │
    ▼
Phase 1  ──► Node Classifier JSON       (which nodes exist and where)
    │
    ▼
Phase 2  ──► Per-Node Ports JSON        (what each node publishes/subscribes)
         ──► Per-Node PlantUML Diagrams (component diagram per node)
    │
    ▼
Phase 3  ──► Preprocess Launch JSON     (how nodes are launched and namespaced)
         ──► System Architecture JSON   (full runtime topology)
         ──► System PlantUML Diagram    (one diagram showing everything)
```

---

## Full Pipeline Workflow

### Phase 1 — Node Extraction

**Agent:** `ros_code_analyst`  
**Crew:** `crew1_node_extractor`

The pipeline starts by scanning every `.cpp`, `.hpp`, `.h`, and `.py` file in the ROS 2 package. For each file, the agent calls the `collect_ros_file_context` tool, which returns the file's role and raw source code. The agent identifies every class that extends `rclcpp::Node` (C++) or `rclpy.node.Node` (Python), extracts its runtime name, source/header paths, and a short description, and aggregates everything into one JSON object.

**Output:** `<package>_all_nodes.json`

```json
{
  "packages": [{
    "package_name": "my_robot",
    "atomic_ros_node_classifiers": [
      {
        "id": "arc_1",
        "class_name": "CameraNode",
        "node_name": "camera",
        "source_file_paths": ["src/camera_node.cpp"],
        "header_file_paths": ["include/my_robot/camera_node.hpp"],
        "description": "Publishes camera frames on /camera/image_raw.",
        "compile_type": "python",
        "execution": "CameraNode"
      }
    ]
  }]
}
```

---

### Phase 2a — Per-Node Port Extraction

**Agent:** `node_ports_extractor`  
**Crew:** `crew2_node_diagrams` (first task)

For every node found in Phase 1, the agent reads its source file and performs a mandatory scan for every `create_publisher`, `create_subscription`, `create_client`, and `create_service` call. It extracts the exact topic/service name (or traces parameter-derived names back to `declare_parameter`), message types, and port direction.

**Output:** `node_<ClassName>.json` per node

```json
{
  "classifier_name": "CameraNode",
  "ports": [
    {
      "port_type": "Publisher",
      "identifier": "pub_1",
      "message_type": "sensor_msgs/msg/Image",
      "original_topic_name": "/camera/image_raw",
      "param_derived": false
    }
  ]
}
```

---

### Phase 2b — Per-Node PlantUML Diagram Generation

**Agent:** `plantuml_diagram_generator`  
**Crew:** `crew2_node_diagrams` (second task)  
**Prompt used:** `diagram_fewshot_prompt_python_json.md`

The agent loads the ports JSON produced in Phase 2a and converts it directly into a PlantUML component diagram using the few-shot examples in the prompt file as its style guide. It does not re-read source files — the ports JSON is the sole input.

Port mapping rules:
- `Publisher` → `portout` stub inside the component + `frame` outside + green arrow going OUT
- `Subscriber` → `portin` stub inside the component + `frame` outside + blue arrow coming IN
- `Client` → `folder` outside + lollipop connector `node --( service`
- `Service` → `folder` outside + filled lollipop `node --0 service`

**Output:** `node_<ClassName>.puml` per node

---

### Phase 3 Pre-Step — Launch File Preprocessing

**Agent:** `preprocess_launch_analyzer`  
**Crew:** `crew3_system_architecture` (first task)

Before the system architecture can be built, a static Python analyzer (`analyze_launch_dependencies`) walks the launch file include tree and produces a launch dependency JSON. The agent then reads every reachable launch file, extracts all `Node(...)` and `ComposableNode(...)` declarations, tracks `PushRosNamespace` groupings, and maps out which launch file includes which.

**Output:** `<subsystem>_preprocess_launch.json`

```json
{
  "launch_file_instances": [
    { "id": "lf1", "type": "robot.launch.py", "nodes": ["n1", "n2"], "included_launch_files": [] }
  ],
  "node_instances": [
    { "id": "n1", "node_kind": "composable", "type": "CameraNode", "node_name": "camera", "namespace": "/sensors" }
  ]
}
```

---

### Phase 3a — System Architecture JSON Extraction

**Agent:** `system_architecture_json_extractor`  
**Crew:** `crew3_system_architecture` (second task)  
**Prompt used:** `architecture_json_construction_prompt.md`

This agent synthesizes three inputs it did NOT produce itself — it never reads source files or launch files directly:

| Input | Source | Contains |
|-------|--------|----------|
| Preprocess JSON | Phase 3 pre-step | Which nodes are launched, in which namespaces |
| Node Classifier JSON | Phase 1 output | Class name → source file mapping |
| Per-Node JSON files | Phase 2a outputs | Exact topic/service names and message types per node |

The agent builds a node mapping (launch instance → class name → ports JSON), then assembles a single flat architecture JSON following the schema defined in `architecture_json_construction_prompt.md`.

**Output:** `<subsystem>_architecture.json`

```json
{
  "launch_file_name": "robot.launch.py",
  "RosNodeParts": [
    {
      "id": "rnp_1",
      "node_name": "camera",
      "class_name": "CameraNode",
      "full_namespace": "/sensors",
      "ports": [
        {
          "port_type": "Publisher",
          "identifier": "pub_1",
          "message_or_service_type": "sensor_msgs/msg/Image",
          "connected_topic_or_service_name": "/sensors/camera/image_raw",
          "is_connected": true
        }
      ]
    }
  ],
  "topics": [{ "message_type": "sensor_msgs/msg/Image", "topic_name": "/sensors/camera/image_raw" }],
  "services": [],
  "namespaces": ["/sensors"]
}
```

---

### Phase 3b — System Architecture PlantUML Generation

**Agent:** `system_architecture_plantuml_generator`  
**Crew:** `crew3_system_architecture` (third task)  
**Prompt used:** `architecture_plantuml_flat_format_final.md`

The final agent loads the architecture JSON from Phase 3a and renders one complete PlantUML system diagram. Every node becomes a `<<RosNodePart>>` component, every topic becomes a `queue` element, and every service becomes a `circle <<Service>>` element. Connection arrows are color-coded:

| Arrow color | Meaning |
|-------------|---------|
| Green `#Green,bold` | Publisher → topic queue |
| Blue `#Blue,bold` | Topic queue → Subscriber |
| Red `#Red,bold` | Client port → service circle |
| Purple `#Purple,bold` | Service-server port → service circle |

**Output:** `<subsystem_or_level_name>.puml` + rendered PNG

---

## Agents and Their Roles

| Agent | Phase | Prompt File Used | Key Tools |
|-------|-------|-----------------|-----------|
| `ros_code_analyst` | 1 | — (embedded in `agents.yaml`) | `collect_ros_file_context`, `file_reader`, `directory_reader` |
| `node_ports_extractor` | 2a | — (embedded in `agents.yaml`) | `collect_ros_file_context` |
| `plantuml_diagram_generator` | 2b | `diagram_fewshot_prompt_python_json.md` | `load_node_ports_json`, `render_puml_to_png` |
| `preprocess_launch_analyzer` | 3 pre | — (embedded in `agents.yaml`) | `file_reader` |
| `system_architecture_json_extractor` | 3a | `architecture_json_construction_prompt.md` | `file_reader`, `directory_reader` |
| `system_architecture_plantuml_generator` | 3b | `architecture_plantuml_flat_format_final.md` | `load_architecture_for_plantuml` |

---

## Prompt Files

### `diagram_fewshot_prompt_python_json.md`

**Used by:** `plantuml_diagram_generator` (Phase 2b)

This prompt provides **few-shot examples** that teach the agent the exact PlantUML structure and visual style to use when generating per-node component diagrams. It defines:

- The `skinparam` settings (colors, font, border style) for the `<<AtomicClassifierDiagram>>` diagram type
- How to declare the outer `component` for the node using `<<AtomicClassifierDiagram>>`
- The `portin` / `portout` stub conventions and their identifiers (`pub_1`, `sub_1`, etc.)
- How to wrap topic queues inside `frame` blocks and service interfaces inside `folder` blocks
- The exact arrow syntax and color conventions (green for publishers, blue for subscribers)
- Multiple concrete before/after examples showing input ports JSON → output PlantUML

The agent uses these examples as a style template — it never invents its own layout. It reads the JSON, matches each port to the corresponding few-shot pattern, and fills in the real values.

---

### `architecture_json_construction_prompt.md`

**Used by:** `system_architecture_json_extractor` (Phase 3a)

This prompt defines the **schema and construction rules** for the system architecture JSON. It is injected as the complete instruction set for how to map launch-time topology onto source-level port information. It covers:

- The exact JSON schema for `Architecture_json` including `launch_file_name`, `RosNodeParts`, `topics`, `services`, and `namespaces`
- How to map a `node_instance.type` (from the launch file) to a `class_name` (from the node classifier JSON) when names differ between runtime and source
- The `is_connected` rule: any port with a known topic/service name is `is_connected: true`, even if its counterpart is an external system
- The `topics[]` completeness rule: every topic that appears in any port (publisher or subscriber) must appear in `topics[]`, not only those with internal matches
- How to handle composable nodes vs. executable nodes differently
- Namespace resolution: how `PushRosNamespace` prefixes combine with node-level namespaces to form `full_namespace`
- The mandatory file inventory format the agent must write before producing any JSON output

---

### `architecture_plantuml_flat_format_final.md`

**Used by:** `system_architecture_plantuml_generator` (Phase 3b)

This prompt is the **complete PlantUML generation guide** for system-level diagrams. Unlike the per-node few-shot prompt (which shows small examples), this guide covers the full diagram structure for a multi-node system. It defines:

- The outer `component` wrapper for the launch file using `<<ComposedRosNodeClassifier>>`
- How each `RosNodePart` becomes an inner `component` with `<<RosNodePart>>` stereotype
- The port counter rule: the port index (`p1`, `p2`, `s1`, `s2`, …) continues across all nodes in the diagram and never resets between components
- How `topics[]` entries render as `queue "topic_name" as topic_alias` elements
- How `services[]` entries render as `circle "service_name" as service_alias <<Service>>` elements
- The exact arrow syntax for all four connection types (publisher→topic, topic→subscriber, client→service, server→service)
- Grouping of nodes by namespace using `rectangle` or `package` blocks
- Formatting and style rules consistent with the per-node diagrams

---

## How Prompts Are Fed to Agents

The prompts are not embedded directly in agent config files. Instead, they are loaded at runtime and injected as **task-level template variables** before the crew starts. The mechanism works as follows:

**Step 1 — Load prompt files** (`crew.py`, method `_read_knowledge_file`)

```python
# Constants pointing to the active prompt files
_KF_DIAGRAM_FEWSHOT = 'diagram_fewshot_prompt_python_json.md'
_KF_EXTRACTION      = 'architecture_json_construction_prompt.md'
_KF_PLANTUML        = 'architecture_plantuml_flat_format_final.md'

def _read_knowledge_file(self, filename: str) -> str:
    # Searches up to 5 parent directories for a knowledge/ or prompts/ folder
    # Reads and returns the file content as a plain string
```

**Step 2 — Pass as input variables** (`crew.py`, methods `generate_all_node_diagrams` and `build_system_architecture`)

```python
# Phase 2 — diagram_fewshot_prompt_python_json.md
diagram_fewshot_guidance = self._read_knowledge_file(self._KF_DIAGRAM_FEWSHOT)
all_node_inputs.append({
    ...
    'diagram_fewshot_guidance': diagram_fewshot_guidance,  # <-- injected here
})

# Phase 3 — architecture_json_construction_prompt.md + architecture_plantuml_flat_format_final.md
extraction_guidance = self._read_knowledge_file(self._KF_EXTRACTION)
plantuml_guidance   = self._read_knowledge_file(self._KF_PLANTUML)
combined_inputs = {
    ...
    'extraction_guidance': extraction_guidance,  # <-- injected here
    'plantuml_guidance':   plantuml_guidance,    # <-- injected here
}
```

**Step 3 — Resolved inside task descriptions** (`tasks.yaml`)

The task description for each affected task has a `{placeholder}` at the top that CrewAI fills in before the agent sees it:

```yaml
generate_plantuml_diagram:
  description: >
    ==================== FEWSHOT DIAGRAM GUIDANCE ====================
    {diagram_fewshot_guidance}          # <-- full content of diagram_fewshot_prompt_python_json.md
    ===================================================================
    ...rest of task instructions...

extract_system_architecture_json:
  description: >
    ==================== architecture_json_construction_prompt.md ====================
    {extraction_guidance}               # <-- full content of architecture_json_construction_prompt.md
    ========================================================================================
    ...rest of task instructions...

generate_system_architecture_plantuml:
  description: >
    ==================== PLANTUML GENERATION GUIDE ====================
    {plantuml_guidance}                 # <-- full content of architecture_plantuml_flat_format_final.md
    ====================================================================
    ...rest of task instructions...
```

The agent receives the **entire prompt file content** as the first section of its task description, followed by specific step-by-step instructions for that task. This means the prompt file acts as the agent's primary reference guide, and the task description tells it exactly how to use that guide on the current inputs.

---

## Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│  ROS 2 Source Code  (C++ / Python packages)                         │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                    ┌───────────▼────────────┐
                    │  ros_code_analyst       │  Phase 1
                    │  [no prompt file]       │
                    └───────────┬────────────┘
                                │  <package>_all_nodes.json
                    ┌───────────▼────────────┐
                    │  node_ports_extractor   │  Phase 2a
                    │  [no prompt file]       │
                    └───────────┬────────────┘
                                │  node_<ClassName>.json  (one per node)
                    ┌───────────▼────────────┐
                    │  plantuml_diagram_      │  Phase 2b
                    │  generator              │
                    │  [diagram_fewshot_      │  ◄── diagram_fewshot_prompt_python_json.md
                    │   prompt_python_json.md]│
                    └───────────┬────────────┘
                                │  node_<ClassName>.puml  (one per node)
                                │
    ┌───────────────────────────┤
    │  Launch Files             │
    │  (static analyzer)        │
    └───────────────┬───────────┘
                    │  <subsystem>_launch_deps.json
                    │
                    ┌───────────▼────────────┐
                    │  preprocess_launch_     │  Phase 3 pre
                    │  analyzer               │
                    │  [no prompt file]       │
                    └───────────┬────────────┘
                                │  <subsystem>_preprocess_launch.json
                    ┌───────────▼────────────┐
                    │  system_architecture_   │  Phase 3a
                    │  json_extractor         │
                    │  [architecture_json_    │  ◄── architecture_json_construction_prompt.md
                    │   construction_         │
                    │   prompt.md]            │
                    └───────────┬────────────┘
                                │  <subsystem>_architecture.json
                    ┌───────────▼────────────┐
                    │  system_architecture_   │  Phase 3b
                    │  plantuml_generator     │
                    │  [architecture_plantuml_│  ◄── architecture_plantuml_flat_format_final.md
                    │   flat_format_final.md] │
                    └───────────┬────────────┘
                                │
                    ┌───────────▼────────────┐
                    │  System Architecture    │
                    │  PlantUML + PNG         │
                    └────────────────────────┘
```
