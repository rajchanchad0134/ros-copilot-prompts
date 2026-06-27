# ROS Co-Pilot Prompts

LLM prompts used in the ROS architecture co-pilot pipeline.

## Active Prompts

| File | Role |
|------|------|
| `architecture_json_construction_prompt.md` | Phase 3a — extract system architecture JSON from launch files + source code |
| `architecture_plantuml_flat_format_final.md` | Phase 3b — generate PlantUML diagram from architecture JSON |
| `diagram_fewshot_prompt_python_json.md` | Phase 2 — few-shot examples for per-node port extraction and diagram generation |

## Pipeline

```
Phase 1  →  extract ROS nodes from source
Phase 2  →  extract node ports + generate per-node PlantUML   [diagram_fewshot_prompt_python_json.md]
Phase 3a →  build system architecture JSON                     [architecture_json_construction_prompt.md]
Phase 3b →  generate system-level PlantUML                     [architecture_plantuml_flat_format_final.md]
```
