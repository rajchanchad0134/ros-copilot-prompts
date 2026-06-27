# Architecture JSON Construction Prompt (Full Format)

---

## [Inputs]

### 1. preprocess_output_json

**Purpose:** Contains all node instances and launch file instances with their relationships, namespaces, and execution metadata.

**Structure:**
```json
{
  "launch_file_instances": [
    {
      "id": "lf1",
      "launch_file_path": "/path/to/launch/file.py",
      "type": "root.launch.py",
      "nodes": ["n1"],
      "included_launch_files": ["lf2", "lf3"],
      "namespace": {
        "/front": ["lf2"],
        "/rear": ["lf3"],
        "/core": ["n1"]
      }
    },
    {
      "id": "lf2",
      "launch_file_path": "/path/to/launch/camera.py",
      "type": "camera.launch.py",
      "nodes": ["n2"],
      "included_launch_files": [],
      "namespace": {
        "/driver": ["n2"]
      }
    },
    {
      "id": "lf3",
      "launch_file_path": "/path/to/launch/camera.py",
      "type": "camera.launch.py",
      "nodes": ["n3"],
      "included_launch_files": [],
      "namespace": {
        "/driver": ["n3"]
      }
    }
  ],
  "node_instances": [
    {
      "id": "n1",
      "node_kind": "executable",
      "type": "image_processor_node",
      "node_name": "processor_main",
      "namespace": "/local"
    },
    {
      "id": "n2",
      "node_kind": "composable",
      "type": "CameraDriverComponent",
      "node_name": "camera_driver_front",
      "namespace": "/cam"
    },
    {
      "id": "n3",
      "node_kind": "executable",
      "type": "camera_driver_node",
      "node_name": "camera_driver_rear",
      "namespace": "/cam"
    }
  ]
}
```

**Key Fields:**
- `launch_file_instances[]`: Array of launch file definitions
  - `id`: Unique identifier for the launch file instance
  - `launch_file_path`: Absolute file path to the launch file (used for remapping parsing)
  - `type`: Launch file type/name (e.g., "root.launch.py", "camera.launch.py")
  - `nodes`: List of node IDs directly launched by this file
  - `included_launch_files`: List of sub-launch file IDs included by this file
  - `namespace`: Dictionary mapping namespace paths to node/launch IDs that belong to them

- `node_instances[]`: Array of ROS node definitions
  - `id`: Unique identifier for the node instance
  - `node_kind`: Either "executable" or "composable"
  - `type`: The executable name (for executable) or component plugin class name (for composable)
  - `node_name`: Runtime node name as declared in launch file
  - `namespace`: The namespace context for this node

---

### 2. Crew1_json

**Purpose:** Contains AtomicRosNodeClassifier definitions that map executables/plugins to their class implementations, source files, and metadata.

**Structure:**
```json
{
  "packages": [
    {
      "package_name": "my_pkg",
      "atomic_ros_node_classifiers": [
        {
          "id": "arc_1",
          "class_name": "Apple",
          "node_name": "runtime_node_name",
          "header_file_paths": ["include/a.hpp"],
          "source_file_paths": ["src/a.cpp"],
          "description": "example node",
          "compile_type": "composable",
          "execution": "Apple"
        },
        {
          "id": "arc_2",
          "class_name": "Dog",
          "node_name": "runtime_node_name_2",
          "header_file_paths": ["include/d.hpp"],
          "source_file_paths": ["src/d.cpp"],
          "description": "another node",
          "compile_type": "executable",
          "execution": "dog_exec"
        }
      ]
    }
  ]
}
```

**Key Fields:**
- `packages[]`: Array of ROS packages
  - `package_name`: Name of the ROS package
  - `atomic_ros_node_classifiers[]`: Array of node classifiers in this package

- `atomic_ros_node_classifiers[]`: Node classifier entries
  - `id`: Unique identifier for this classifier
  - `class_name`: C++ class name of the ROS node
  - `node_name`: Runtime node name (from launch configuration)
  - `header_file_paths`: Array of header files where rclcpp::Node or rclpy.node.Node is defined
  - `source_file_paths`: Array of source files:
    - For composable nodes: where `RCLCPPCOMPONENTSREGISTER_NODE` macro exists
    - For executable nodes: where the Node is instantiated (main.cpp)
  - `description`: Human-readable description of the node
  - `compile_type`: Either "composable" or "executable"
  - `execution`: The execution name used in launch files (e.g., executable name or plugin class name)

**Search Rules:**
- To find a classifier for an executable node with `type: "image_processor_node"`:
  - Search for entry where `compile_type == "executable"` AND `execution == "image_processor_node"`
  - Extract the `class_name` field (e.g., "ImageProcessor")

- To find a classifier for a composable node with `type: "CameraDriverComponent"`:
  - Search for entry where `compile_type == "composable"` AND `execution == "CameraDriverComponent"`
  - Extract the `class_name` field

---

### 3. Crew2_json

**Purpose:** Contains port definitions for each AtomicRosNodeClassifier, detailing all publishers, subscribers, service clients, and service servers.

**Structure:**
```json
{
  "classifier_name": "AtomicRosNodeClassifier_name1",
  "ports": [
    {
      "port_type": "Publisher",
      "identifier": "pub_1",
      "message_type": "sensor_msgs/msg/Image",
      "original_topic_name": "/camera/image_raw"
    },
    {
      "port_type": "Publisher",
      "identifier": "pub_2",
      "message_type": "std_msgs/msg/String",
      "original_topic_name": "/system/status"
    },
    {
      "port_type": "Subscriber",
      "identifier": "sub_1",
      "message_type": "geometry_msgs/msg/Twist",
      "callback_function": "cmdCallback",
      "original_topic_name": "/cmd_vel"
    },
    {
      "port_type": "Client",
      "identifier": "client_1",
      "service_type": "std_srvs/srv/Empty",
      "original_service_name": "/reset"
    },
    {
      "port_type": "Service",
      "identifier": "service_1",
      "service_type": "std_srvs/srv/SetBool",
      "service_function": "handleEnable",
      "original_service_name": "/enable"
    }
  ]
}
```

**Key Fields:**
- `classifier_name`: Name of the classifier (matches AtomicRosNodeClassifier.class_name)
- `ports[]`: Array of port definitions

- `ports[].Publisher/Subscriber/Client/Service`:
  - `port_type`: Type of port ("Publisher", "Subscriber", "Client", or "Service")
  - `identifier`: Unique identifier for this port within the classifier
  - `message_type`: ROS message type (for Publisher/Subscriber)
  - `service_type`: ROS service type (for Client/Service)
  - `callback_function`: Function name for Subscriber ports (optional)
  - `service_function`: Function name for Service ports (optional)
  - `original_topic_name`: Original topic name before remapping
  - `original_service_name`: Original service name before remapping

**Indexing Rule:**
- Access Crew2_json entries by classifier_name: `Crew2_json[class_name]`
- Example: If class_name is "Apple", find the Crew2_json entry with `classifier_name: "Apple"`

---

### 4. existing_Arch_json

**Purpose:** Previously generated Architecture JSONs from this process, used for recursive/hierarchical architecture construction.

**Structure & Usage:**
```json
{
  "launch_file_name": "example_launch.py",
  "RosNodeParts": [
    {
      "id": "1",
      "node_name": "camera_node",
      "class_name": "CameraDriver",
      "full_namespace": "/aa/bb",
      "ports": [
        {
          "is_connected": true,
          "message_or_service_type": "sensor_msgs/msg/Image",
          "connected_topic_or_service_name": "/camera/image_raw",
          "port_type": "Publisher",
          "identifier": "pub_image"
        },
        {
          "is_connected": false,
          "message_or_service_type": "std_srvs/srv/SetBool",
          "connected_topic_or_service_name": "/camera/enable",
          "port_type": "Service",
          "identifier": "srv_enable"
        }
      ]
    },
    {
      "node_name": "perception_node",
      "class_name": "PerceptionProcessor",
      "ports": [
        {
          "is_connected": true,
          "message_or_service_type": "sensor_msgs/msg/Image",
          "connected_topic_or_service_name": "/camera/image_raw",
          "port_type": "Subscriber",
          "identifier": "sub_image"
        }
      ]
    }
  ],
  "topics": [
    {
      "message_type": "sensor_msgs/msg/Image",
      "topic_name": "/camera/image_raw"
    },
    {
      "message_type": "std_msgs/msg/String",
      "topic_name": "/diagnostics"
    }
  ],
  "services": [
    {
      "service_type": "std_srvs/srv/SetBool",
      "service_name": "/camera/enable"
    }
  ],
  "namespaces": [
    "/camera",
    "/perception"
  ]
}
```

**Key Fields:**
- `launch_file_name`: Name of the launch file that generated this architecture
- `RosNodeParts[]`: Previously constructed node parts with full port information
- `topics[]`: Defined topics with message types
- `services[]`: Defined services with service types
- `namespaces[]`: List of all namespaces present in the architecture

**Indexing Rule:**
- Access existing architectures by type: `existing_Arch_json[launch_file_instance.type]`
- Example: If processing launch type "camera.launch.py", retrieve `existing_Arch_json["camera.launch.py"]`

---

## [Output]

### Architecture_json

**Purpose:** Complete architecture representation with all RosNodeParts, ports, topics, services, and namespace groupings.

**Structure:**
```json
{
  "launch_file_name": "example_launch.py",
  "RosNodeParts": [
    {
      "id": "unique_id",
      "node_name": "node_runtime_name",
      "class_name": "ClassName",
      "full_namespace": "/path/to/namespace",
      "ports": [
        {
          "is_connected": true,
          "message_or_service_type": "pkg/msg/Type",
          "connected_topic_or_service_name": "/full/topic/path",
          "port_type": "Publisher",
          "identifier": "pub_1"
        },
        {
          "is_connected": false,
          "message_or_service_type": "pkg/srv/Type",
          "connected_topic_or_service_name": "/service/path",
          "port_type": "Service",
          "identifier": "srv_1"
        }
      ]
    }
  ],
  "topics": [
    {
      "message_type": "pkg/msg/Type",
      "topic_name": "/full/topic/path"
    }
  ],
  "services": [
    {
      "service_type": "pkg/srv/Type",
      "service_name": "/service/path"
    }
  ],
  "namespaces": [
    "/namespace1",
    "/namespace2"
  ]
}
```

**Required Fields:**
- `launch_file_name`: Name of the launch file being processed
- `RosNodeParts[]`: Array of all constructed node parts
  - `id`: Unique identifier
  - `node_name`: Runtime node name
  - `class_name`: ROS node class name
  - `full_namespace`: Complete namespace path
  - `ports[]`: All ports (publisher, subscriber, client, service)
    - `is_connected`: Boolean — true if the port's topic/service name is resolved
        and non-empty (even if the counterpart is an external node outside this system);
        false ONLY if the topic/service name is empty or could not be determined at all
    - `message_or_service_type`: Message or service type
    - `connected_topic_or_service_name`: Final resolved topic/service name
    - `port_type`: "Publisher", "Subscriber", "Client", or "Service"
    - `identifier`: Unique port identifier

- `topics[]`: Array of all discovered topics
  - `message_type`: ROS message type
  - `topic_name`: Full resolved topic path

- `services[]`: Array of all discovered services
  - `service_type`: ROS service type
  - `service_name`: Full resolved service path

- `namespaces[]`: Array of all namespace paths discovered

---

## STEP 1: Architecture Initialization

### [1.1 Determine Architecture Name]

**Rule:**
Extract the architecture name from the current launch_file_instance being processed.

**Instructions:**
1. Locate the launch_file_instance to be processed from preprocess_output_json.launch_file_instances
2. Extract the `type` field from launch_file_instance
3. Assign this value to `Arch_json.launch_file_name`

**Example:**
```
Input: launch_file_instance.type = "root.launch.py"
Output: Arch_json.launch_file_name = "root.launch.py"

Input: launch_file_instance.type = "camera.launch.py"
Output: Arch_json.launch_file_name = "camera.launch.py"
```

**CrewAI Tool Usage:**
Use the `ExtractFieldTool` from CrewAI to extract the `type` field:
```python
# Pseudo-code for agent usage
tool.extract_field(
  source_object=launch_file_instance,
  field_name="type",
  target_field="launch_file_name"
)
```

---

### [1.2 Determine Number of RosNodeParts]

**Rule:**
Calculate the total count of RosNodeParts that will be created by adding direct nodes and included launch files.

**Instructions:**
1. Count direct nodes: `count_nodes = len(launch_file_instance.nodes)`
2. Count included launch files: `count_includes = len(launch_file_instance.included_launch_files)`
3. Total RosNodeParts: `total = count_nodes + count_includes`
4. Create RosNodePart entries for each:
   - For each ID in `launch_file_instance.nodes`: create one RosNodePart (Case A - node_instance)
   - For each ID in `launch_file_instance.included_launch_files`: create one RosNodePart (Case B - launch_file_instance)

**Example:**
```
Input:
  launch_file_instance.nodes = ["n1"]
  launch_file_instance.included_launch_files = ["lf2", "lf3"]

Calculation:
  count_nodes = 1
  count_includes = 2
  total_RosNodeParts = 3

Output:
  Create RosNodeParts for: n1 (Case A), lf2 (Case B), lf3 (Case B)
```

**CrewAI Tool Usage:**
Use `CountAndMapTool` to count and create ID mappings:
```python
tool.count_and_map(
  nodes_list=launch_file_instance.nodes,
  includes_list=launch_file_instance.included_launch_files,
  case_mapping={"nodes": "Case_A", "includes": "Case_B"}
)
```


---

### [1.3 Namespace Information]

**Rule:**
Extract and organize all namespace information from the launch_file_instance.

**Instructions:**
1. Locate `launch_file_instance.namespace` dictionary
2. For each namespace key (e.g., "/front", "/rear", "/core"):
   - Initialize `Arch_json.namespace[namespace_key] = []`
3. For each ID in the namespace's value list:
   - Store the ID for later assignment to RosNodeParts

**Data Structure:**
```
Arch_json.namespace = {
  "/front": ["lf2"],
  "/rear": ["lf3"],
  "/core": ["n1"]
}
```

**Example Processing:**
```
Input: launch_file_instance.namespace = {
  "/front": ["lf2"],
  "/rear": ["lf3"],
  "/core": ["n1"]
}

Output Arch_json.namespace:
{
  "/front": [],  // Will be populated with RosNodePart IDs
  "/rear": [],
  "/core": []
}

Later (Step 2.1.4): Assign RosNodePart IDs to these namespace lists
```

**Purpose:**
This namespace structure is critical for:
- Tracking which RosNodeParts belong to which namespace
- Computing full namespace paths during port resolution
- Assigning topics/services to appropriate namespace groups

---

## STEP 2: Construct RosNodePart

### [2.1 Case A: ID corresponds to node_instance]

#### [2.1.1 Basic Attributes]

**Rule:**
Assign fundamental attributes to the RosNodePart from the node_instance.

**Instructions:**
1. Look up the node_instance using the ID from launch_file_instance.nodes
2. Assign attributes:
   ```
   RosNodePart.node_name = node_instance.node_name
   RosNodePart.id = generate_unique_id()  // e.g., "rnp_1", "rnp_2", etc.
   ```

**Example:**
```
Input: node_instance (id="n1")
{
  "id": "n1",
  "node_kind": "executable",
  "type": "image_processor_node",
  "node_name": "processor_main",
  "namespace": "/local"
}

Output: RosNodePart
{
  "id": "rnp_1",
  "node_name": "processor_main",
  ...
}
```

**ID Uniqueness Rule:**
- Generate IDs sequentially: "rnp_1", "rnp_2", "rnp_3", etc.
- Ensure no ID is reused within the same Arch_json
- Maintain ID consistency if processing is interrupted and resumed

---

#### [2.1.2 Classifier Resolution]

**Rule:**
Resolve the RosNodePart class_name based on the node_kind (composable or executable).

**Instructions:**

**For composable nodes:**
1. If `node_instance.node_kind == "composable"`:
   - Direct assignment: `RosNodePart.class_name = node_instance.type`

**For executable nodes:**
1. If `node_instance.node_kind == "executable"`:
   - Search Crew1_json for an AtomicRosNodeClassifier matching:
     - `compile_type == "executable"` AND
     - `execution == node_instance.type`
   - If found: `RosNodePart.class_name = AtomicRosNodeClassifier.class_name`
   - If not found: Log warning and use `node_instance.type` as fallback

**Examples:**

```
Example 1 - Composable:
Input: node_instance
{
  "node_kind": "composable",
  "type": "CameraDriverComponent"
}

Search result: Direct mapping
Output: RosNodePart.class_name = "CameraDriverComponent"
```

```
Example 2 - Executable:
Input: node_instance
{
  "node_kind": "executable",
  "type": "image_processor_node"
}

Search Crew1_json:
Find: atomic_ros_node_classifiers entry where
  compile_type = "executable"
  execution = "image_processor_node"
  class_name = "ImageProcessor"

Output: RosNodePart.class_name = "ImageProcessor"
```

**CrewAI Tool Usage:**
Use `ClassifierLookupTool` to search Crew1_json:
```python
tool.lookup_classifier(
  crew1_json=crew1_json_input,
  node_kind=node_instance.node_kind,
  search_value=node_instance.type,
  search_field="execution" if node_kind=="executable" else None
)
```

---

#### [2.1.3 Namespace Resolution]

**Rule:**
Compute the complete namespace path for the RosNodePart by accumulating all parent namespaces.

**Instructions:**

1. Call `get_namespace_scope(preprocess_output_json, node_instance)` function
2. This function must:
   - Traverse the node's namespace hierarchy in preprocess_output_json
   - Start from the launch_file_instance that contains this node
   - Accumulate namespaces from parent launches upward to the root
   - Follow ROS2 namespace propagation semantics

**Algorithm:**
```
function get_namespace_scope(preprocess_output_json, node_instance):
    result_namespace = ""
    
    // Find which launch file contains this node
    for launch_file in preprocess_output_json.launch_file_instances:
        if node_instance.id in launch_file.nodes:
            current_launch = launch_file
            break
    
    // Accumulate namespaces by traversing namespace hierarchy
    for namespace_key in current_launch.namespace:
        if node_instance.id in current_launch.namespace[namespace_key]:
            result_namespace = namespace_key
            break
    
    // Combine with node's own namespace
    if node_instance.namespace:
        if not result_namespace.endswith("/"):
            result_namespace += "/"
        result_namespace += node_instance.namespace.strip("/")
    
    // Ensure starts with "/"
    if not result_namespace.startswith("/"):
        result_namespace = "/" + result_namespace
    
    return result_namespace
```

**Example:**
```
Input:
  launch_file_instance.namespace["/core"] = ["n1"]
  node_instance.namespace = "/local"

Process:
  namespace_scope = "/core"
  combined = "/core" + "/local" = "/core/local"

Output: RosNodePart.full_namespace = "/core/local"
```

---

#### [2.1.4 Namespace Assignment]

**Rule:**
Assign the RosNodePart to the appropriate namespace group in Arch_json.

**Instructions:**

1. Check if `node_instance.id` exists in any namespace of `launch_file_instance.namespace`
2. For the matching namespace:
   - Append `RosNodePart.id` to `Arch_json.namespace[namespace_key]`

**Example:**
```
Input:
  launch_file_instance.namespace = {
    "/front": ["lf2"],
    "/rear": ["lf3"],
    "/core": ["n1"]
  }
  node_instance.id = "n1"
  RosNodePart.id = "rnp_1"

Process:
  Find n1 in namespace["/core"]
  Append rnp_1 to Arch_json.namespace["/core"]

Output: Arch_json.namespace["/core"] = ["rnp_1"]
```

---

#### [2.1.5 Port Construction (from Crew2_json)]

**Rule:**
Extract all ports from the corresponding Crew2_json entry and add them to the RosNodePart.

**Instructions:**

1. Use `RosNodePart.class_name` to find the Crew2_json entry
2. Locate `Crew2_json[class_name]` from the Crew2_json collection
3. For each port in `Crew2_json[class_name].ports`:
   - Create a port object with the following mapping:
     ```
     port.port_type = Crew2_json_port.port_type
     port.identifier = Crew2_json_port.identifier
     port.message_or_service_type = Crew2_json_port.message_type OR service_type
     port.original_topic_or_service_name = Crew2_json_port.original_topic_name OR original_service_name
     port.connected_topic_or_service_name = "" (will be filled in Step 2.1.6)
     port.is_connected = false
     ```

4. Add all ports to `RosNodePart.ports[]`

**Example:**
```
Input:
  RosNodePart.class_name = "Apple"
  Crew2_json["Apple"].ports = [
    {
      "port_type": "Publisher",
      "identifier": "pub_1",
      "message_type": "sensor_msgs/msg/Image",
      "original_topic_name": "/camera/image_raw"
    },
    {
      "port_type": "Service",
      "identifier": "srv_1",
      "service_type": "std_srvs/srv/SetBool",
      "original_service_name": "/enable"
    }
  ]

Process:
  For each Crew2_json port, create RosNodePart.port:
  
Output: RosNodePart.ports[] = [
  {
    "port_type": "Publisher",
    "identifier": "pub_1",
    "message_or_service_type": "sensor_msgs/msg/Image",
    "original_topic_or_service_name": "/camera/image_raw",
    "connected_topic_or_service_name": "",
    "is_connected": false
  },
  {
    "port_type": "Service",
    "identifier": "srv_1",
    "message_or_service_type": "std_srvs/srv/SetBool",
    "original_topic_or_service_name": "/enable",
    "connected_topic_or_service_name": "",
    "is_connected": false
  }
]
```

**CrewAI Tool Usage:**
Use `PortExtractionTool` to extract and map ports from Crew2_json:
```python
tool.extract_ports(
  crew2_json_collection=crew2_json_input,
  classifier_name=rosnode_part.class_name,
  create_port_mapping=True
)
```

---

#### [2.1.6 Remapping Resolution]

**Rule:**
Apply launch file remappings to ports to get their initial topic/service names.

**Instructions:**

1. Parse the launch file at `launch_file_instance.launch_file_path`
2. Find the node instantiation section for the current node:
   - Search for the executable name (for executable nodes) OR plugin class name (for composable nodes)
   - The search target is `node_instance.type`
3. Locate remapping rules associated with this node instantiation:
   - Look for `remap` tags/entries with `from` and `to` fields
4. For each port in RosNodePart.ports:
   - If a remap rule exists where `remap.from == port.original_topic_or_service_name`:
     - Set `port.connected_topic_or_service_name = remap.to`
   - Otherwise:
     - Set `port.connected_topic_or_service_name = port.original_topic_or_service_name`

**Example:**

```
Input:
  Launch file contains:
  <node pkg="my_pkg" exec="image_processor_node" name="processor_main">
    <remap from="image_raw" to="/processed/image"/>
  </node>

  Port (from Crew2_json):
  {
    "port_type": "Publisher",
    "original_topic_or_service_name": "image_raw"
  }

Process:
  Find remap: from="image_raw" to="/processed/image"
  Match found for port.original_topic_or_service_name

Output: port.connected_topic_or_service_name = "/processed/image"
```

```
Input:
  Port (from Crew2_json):
  {
    "port_type": "Subscriber",
    "original_topic_or_service_name": "/sensor/data"
  }
  No matching remap rule found

Process:
  No remap match

Output: port.connected_topic_or_service_name = "/sensor/data"
```

**CrewAI Tool Usage:**
Use `RemappingParserTool` to extract remappings from launch files:
```python
tool.parse_remappings(
  launch_file_path=launch_file_instance.launch_file_path,
  node_type=node_instance.type,
  ports_to_remap=rosnode_part.ports
)
```

---

### [2.2 Case B: ID corresponds to launch_file_instance]

#### [2.2.1 Retrieve Sub-Architecture]

**Rule:**
Locate and retrieve the previously generated Architecture JSON for the sub-launch file.

**Instructions:**

1. Look up the launch_file_instance using the ID from `launch_file_instance.included_launch_files`
2. Extract the `type` field from this sub-launch_file_instance
3. Search existing_Arch_json for entry where `launch_file_name == sub_launch_file_instance.type`
4. If found: store this as `sub_architecture`
5. If not found: Log error - sub-architecture must be generated first (recursive dependency)

**Example:**
```
Input:
  Current: launch_file_instance.id = "lf1"
  Included: launch_file_instance.included_launch_files = ["lf2", "lf3"]
  Sub-launch: lf2.type = "camera.launch.py"

Process:
  Search existing_Arch_json for:
    launch_file_name == "camera.launch.py"

Output:
  sub_architecture = existing_Arch_json["camera.launch.py"]
```

**Recursive Processing Note:**
This architecture construction should be performed recursively:
1. First pass: Process all leaf-level launches (no included_launch_files) → generates base Arch_json
2. Second pass: Process parent launches that include the leaf launches → uses existing Arch_json
3. Continue until root launch is processed

---

#### [2.2.2 Construct RosNodePart]

**Rule:**
Create a RosNodePart that represents the sub-architecture as a black-box component.

**Instructions:**

1. Create a new RosNodePart with:
   ```
   RosNodePart.id = generate_unique_id()
   RosNodePart.node_name = ""  // Empty for wrapped architectures
   RosNodePart.class_name = sub_architecture.launch_file_name
   RosNodePart.full_namespace = get_namespace_scope(preprocess_output_json, launch_file_instance)
   ```

2. The `full_namespace` should include:
   - Namespace from parent launch_file_instance.namespace
   - Combined with any sub-launch namespace context

**Example:**
```
Input:
  sub_architecture.launch_file_name = "camera.launch.py"
  launch_file_instance in namespace["/front"] = ["lf2"]

Process:
  RosNodePart.node_name = ""
  RosNodePart.class_name = "camera.launch.py"
  RosNodePart.full_namespace = "/front"

Output: RosNodePart representing wrapped camera.launch.py
```

---

#### [2.2.3 Extract Unconnected Ports]

**Rule:**
Promote unconnected ports from the sub-architecture to the current RosNodePart.

**Instructions:**

1. Iterate over `sub_architecture.RosNodeParts[*].ports[*]`
2. For each port, check `port.is_connected`
3. Select only ports where `is_connected == false`
4. For each unconnected port:
   - Create a new port in RosNodePart.ports[] with all attributes copied
   - This represents the port as exposed at the parent level

**Example:**
```
Input:
  sub_architecture.RosNodeParts[0].ports = [
    {
      "identifier": "pub_image",
      "port_type": "Publisher",
      "is_connected": true,
      "message_or_service_type": "sensor_msgs/msg/Image"
    },
    {
      "identifier": "srv_enable",
      "port_type": "Service",
      "is_connected": false,
      "message_or_service_type": "std_srvs/srv/SetBool"
    }
  ]

Process:
  Select only srv_enable (is_connected == false)

Output: RosNodePart.ports[] = [
  {
    "identifier": "srv_enable",
    "port_type": "Service",
    "is_connected": false,
    "message_or_service_type": "std_srvs/srv/SetBool"
  }
]
```

**Purpose:**
Unconnected ports are those that:
- Do not have a matching publisher/subscriber pair
- Do not have a matching client/service pair
- Represent external connections needed by the sub-architecture
- Should be promoted to the parent level for connection at higher levels

---

#### [2.2.4 Topic/Service Name Handling]

**Rule:**
Apply namespace resolution to relative topic and service names from sub-architecture.

**Instructions:**

For each port from Step [2.2.3]:

1. Get the port's `connected_topic_or_service_name`
2. If it starts with "/":
   - Keep unchanged (absolute path)
   - `result = connected_topic_or_service_name`
3. Otherwise (relative path):
   - Prepend the RosNodePart's full_namespace
   - `result = RosNodePart.full_namespace + "/" + connected_topic_or_service_name`
   - Simplify double slashes: replace "//" with "/"
4. Assign result to `port.connected_topic_or_service_name`

**Example:**
```
Example 1 - Absolute path:
Input:
  port.connected_topic_or_service_name = "/camera/image_raw"
  RosNodePart.full_namespace = "/front"

Process:
  Starts with "/" → keep as-is

Output: port.connected_topic_or_service_name = "/camera/image_raw"
```

```
Example 2 - Relative path:
Input:
  port.connected_topic_or_service_name = "status"
  RosNodePart.full_namespace = "/front"

Process:
  Does not start with "/" → prepend namespace
  result = "/front" + "/" + "status" = "/front/status"

Output: port.connected_topic_or_service_name = "/front/status"
```

---

## STEP 3: Port Connection & Topic/Service Construction

### [3.1 Compute runtime_full_name]

**Rule:**
Resolve the final runtime-resolved topic and service names for all ports.

**Instructions:**

For each port in each RosNodePart in Arch_json.RosNodeParts:

1. Get the port's current `connected_topic_or_service_name`
2. Apply namespace resolution:
   - If `connected_topic_or_service_name` starts with "/":
     - `runtime_full_name = connected_topic_or_service_name` (absolute, use as-is)
   - Otherwise:
     - `runtime_full_name = RosNodePart.full_namespace + "/" + connected_topic_or_service_name`
3. Validation:
   - Ensure `runtime_full_name` starts with "/"
   - Replace double slashes "//" with "/"
   - Trim trailing slashes if any
4. Assign `port.runtime_full_name = runtime_full_name`

**Example:**
```
Example 1:
Input:
  port.connected_topic_or_service_name = "/global/image"
  RosNodePart.full_namespace = "/cam/front"

Process:
  Starts with "/" → use as-is
  runtime_full_name = "/global/image"

Output: port.runtime_full_name = "/global/image"
```

```
Example 2:
Input:
  port.connected_topic_or_service_name = "processed"
  RosNodePart.full_namespace = "/cam/front"

Process:
  No "/" prefix → prepend namespace
  runtime_full_name = "/cam/front" + "/" + "processed"
  = "/cam/front/processed"

Output: port.runtime_full_name = "/cam/front/processed"
```

```
Example 3:
Input:
  port.connected_topic_or_service_name = "data"
  RosNodePart.full_namespace = "/" (root namespace)

Process:
  No "/" prefix → prepend namespace
  runtime_full_name = "/" + "/" + "data"
  = "//data" → normalize → "/data"

Output: port.runtime_full_name = "/data"
```

---

### [3.2 Define Port Tuple]

**Rule:**
Create port tuples that group ports by their connection characteristics for clustering.

**Instructions:**

For each port, create a tuple containing three elements:

```
port_tuple = (
  port.port_type,
  port.runtime_full_name,
  port.message_or_service_type
)
```

**Tuple Elements:**
1. **port.port_type**: "Publisher", "Subscriber", "Client", or "Service"
2. **port.runtime_full_name**: The fully resolved topic/service name (from Step 3.1)
3. **port.message_or_service_type**: The message type (for topics) or service type (for services)

**Purpose:**
These tuples are used in Step [3.3] to cluster matching ports for connection creation.

**Example:**
```
Port 1 (Publisher):
  port_tuple = (
    "Publisher",
    "/camera/image",
    "sensor_msgs/msg/Image"
  )

Port 2 (Subscriber):
  port_tuple = (
    "Subscriber",
    "/camera/image",
    "sensor_msgs/msg/Image"
  )

These two tuples match on (runtime_full_name, message_type)
and have complementary port_types → can be clustered
```

---

### [3.3 Clustering Logic]

**Rule:**
Group ports that represent the same connection (matching topics or services).

**Instructions:**

**Cluster Formation Conditions:**

A cluster is formed when:
1. **runtime_full_name is identical** across multiple ports
2. **message_or_service_type is identical** across multiple ports
3. **port_type combination is valid:**
   - For Topics: one "Publisher" + one or more "Subscriber"
   - For Services: one "Service" + one or more "Client" (fully connected), OR one or more "Client" without a "Service" (client-only), OR one "Service" without any "Client" (server-only)

**Clustering Algorithm:**

```
clusters = {}

for each port:
  key = (runtime_full_name, message_or_service_type)
  
  if key not in clusters:
    clusters[key] = {
      "publishers": [],
      "subscribers": [],
      "service_providers": [],
      "clients": [],
      "is_topic": (port.port_type in ["Publisher", "Subscriber"]),
      "is_service": (port.port_type in ["Service", "Client"])
    }
  
  if port.port_type == "Publisher":
    clusters[key]["publishers"].append(port)
  elif port.port_type == "Subscriber":
    clusters[key]["subscribers"].append(port)
  elif port.port_type == "Service":
    clusters[key]["service_providers"].append(port)
  elif port.port_type == "Client":
    clusters[key]["clients"].append(port)

return clusters
```

**Valid Cluster Patterns:**

```
Topic Cluster (fully internal):
  - Exactly one Publisher + one or more Subscribers
  - All have same runtime_full_name and message_type
  - ALL ports → is_connected: true

Topic Cluster (publisher-only — external subscribers):
  - One or more Publishers, no Subscribers in this launch file
  - The Subscriber(s) are external hardware drivers or nodes outside this system
  - VALID — topic is still a known, resolved interface
  - ALL ports → is_connected: true

Topic Cluster (subscriber-only — external publisher):
  - One or more Subscribers, no Publisher in this launch file
  - The Publisher is external (e.g. a hardware driver or an external system)
  - VALID — topic is still a known, resolved interface
  - ALL ports → is_connected: true

Service Cluster (fully connected):
  - Exactly one Service provider + one or more Clients
  - All have same runtime_full_name and service_type
  - ALL ports → is_connected: true

Service Cluster (client-only):
  - One or more Clients, no Service provider present in this launch file
  - The Service provider is external to this architecture
  - VALID — service is still part of the system architecture
  - ALL ports → is_connected: true

Service Cluster (server-only):
  - Exactly one Service provider, no Clients present in this launch file
  - The Clients are external to this architecture
  - VALID — service is still part of the system architecture
  - ALL ports → is_connected: true

Warning-only (do NOT mark as unconnected):
  - Multiple Publishers on same topic → log a warning, add topic to topics[],
    mark ALL ports as is_connected: true (topic is still known)
  - Multiple Service providers → log a warning, still mark as is_connected: true
```

**Example:**
```
Input ports:
Port A: (Publisher, "/camera/image", "sensor_msgs/msg/Image")
Port B: (Subscriber, "/camera/image", "sensor_msgs/msg/Image")
Port C: (Subscriber, "/camera/image", "sensor_msgs/msg/Image")

Clustering:
key = ("/camera/image", "sensor_msgs/msg/Image")
cluster = {
  "publishers": [Port A],
  "subscribers": [Port B, Port C],
  "is_topic": true,
  "is_service": false
}

Result: Valid topic cluster
```

---

### [3.4 Construct Topic/Service]

**Rule:**
Create Topic and Service objects from valid clusters.

**Instructions:**

**For ALL clusters (topic or service, any pattern):**

```
topic = {
  "topic_name": runtime_full_name,
  "message_type": message_or_service_type
}

Add topic to Arch_json.topics[] for every unique runtime_full_name found,
regardless of whether Publishers, Subscribers, or both are present.
Mark ALL ports in the cluster as is_connected = true.

service = {
  "service_name": runtime_full_name,
  "service_type": message_or_service_type
}

Add service to Arch_json.services[] for every unique service runtime_full_name found.
Mark ALL ports in the cluster as is_connected = true.

// KEY RULE: Any port whose connected_topic_or_service_name is resolved and non-empty
// gets is_connected = true — even if only Publishers exist (no Subscribers in this system)
// or only Subscribers exist (no Publisher in this system). External interfaces ARE valid.
// The missing side is external — the topic/service still belongs to this architecture.
```

**Example:**
```
Input cluster (valid topic):
key = ("/camera/image", "sensor_msgs/msg/Image")
publishers = 1, subscribers = 2

Output:
1. Create topic object:
   {
     "topic_name": "/camera/image",
     "message_type": "sensor_msgs/msg/Image"
   }

2. Add to Arch_json.topics[]

3. Mark ports as connected:
   All 3 ports (1 publisher + 2 subscribers).is_connected = true
```

**CrewAI Tool Usage:**
Use `TopicConstructorTool` to create topic/service objects:
```python
tool.construct_from_cluster(
  cluster=valid_cluster,
  cluster_type="topic" or "service",
  # For service clusters, specify the pattern:
  #   "full"        → Service provider + Clients both present
  #   "client_only" → Only Clients present (Service provider is external)
  #   "server_only" → Only Service provider present (Clients are external)
  # All present ports are marked is_connected = true in all three patterns.
  # For topic clusters, omit this parameter.
  service_cluster_pattern="full" or "client_only" or "server_only",
  add_to_architecture=Arch_json
)
```

---

### [3.5 Namespace Assignment for Topic/Service]

**Rule:**
Assign topics and services to namespace groups based on their name prefix.

**Instructions:**

For each Topic/Service created in Step [3.4]:

1. Extract the namespace prefix from the topic/service name:
   ```
   topic_name = "/aa/bb/cc/dd/speed"
   launch_file_namespace_scope = "/aa/bb/"
   
   name_result = topic_name - namespace_scope
   name_result = "/cc/dd/speed"
   
   Extract prefix = "cc"
   ```

2. Check if prefix matches any namespace in `Arch_json.namespace`:
   - For each namespace key in Arch_json.namespace:
     - If namespace contains the prefix:
       - Assign this topic/service to that namespace
       - Record the association

3. If no match found:
   - Topic/service remains unassigned to a specific namespace
   - It belongs to the global scope

**Example:**
```
Input:
  topic_name = "/front/camera/image_raw"
  Arch_json.namespace = {
    "/front": ["rnp_1"],
    "/rear": ["rnp_2"],
    "/core": ["rnp_3"]
  }
  launch_file_namespace_scope = "/" (root)

Process:
  name_result = "/front/camera/image_raw" - "/" = "/front/camera/image_raw"
  Extract first level prefix = "front"
  
  Check: Does "front" match any namespace?
  Yes: "/front" matches
  
  Assign topic to "/front" namespace

Output: Arch_json.namespace_assignments["/front"] = [topic_object]
```

---

### [3.6 Update Port Connection State]

**Rule:**
Mark ports as connected based on whether their topic/service name is resolved and non-empty.

**Instructions:**

For EVERY port in every RosNodePart:

1. If `port.connected_topic_or_service_name` is non-empty and non-null:
   - Set `port.is_connected = true`

2. If `port.connected_topic_or_service_name` is empty, null, or could not be resolved:
   - Set `port.is_connected = false`

**KEY RULE:**
`is_connected = true` does NOT require both sides of the topic (Publisher + Subscriber)
to exist within this system. It means only: "this port's topic name is resolved and known."
External interfaces — topics published here but subscribed by hardware drivers, or
subscribed here but published by external systems — are VALID connected ports.

**Tracking:**
Maintain a list of ports where `is_connected == false` (truly unresolvable) for Step [3.7].

**Example:**
```
Port A (Publisher, /camera/image_raw, internal subscriber exists):
  → is_connected = true ✓

Port B (Publisher, /alice/move_urx, no internal subscriber — external driver):
  → is_connected = true ✓  (topic name IS resolved; external subscriber)

Port C (Subscriber, /alice/io_states, no internal publisher — external hardware):
  → is_connected = true ✓  (topic name IS resolved; external publisher)

Port D (Publisher, topic name could not be determined → ""):
  → is_connected = false  (ONLY case where false is correct)
```

---

### [3.7 Handle Unresolvable Ports]

**Rule:**
Organize and preserve ports whose topic/service name could not be determined at all.

**Instructions:**

For ports where `is_connected == false` (topic/service name is empty or null — NOT simply
because the other side is external):

1. **Group by port_type:**
   ```
   unresolvable_publishers = []
   unresolvable_subscribers = []
   unresolvable_clients = []
   unresolvable_services = []
   
   for each unresolvable port (connected_topic_or_service_name is "" or null):
     add to appropriate group based on port_type
   ```

2. **Ensure identifier uniqueness within each group:**
   - Check that `port.identifier` is unique within each group
   - If duplicates found, rename by appending: "_1", "_2", etc.

3. **Preservation in output:**
   - Keep these ports in `RosNodePart.ports[]` with `is_connected: false`
   - They indicate nodes whose topic names could not be extracted from source files

**IMPORTANT — What is NOT in this step:**
External interfaces (topics only subscribed externally, or only published externally) are NOT
unresolvable. They go through the normal Step 3.4–3.6 flow and get `is_connected: true`.

**Example of truly unresolvable port:**
```
Port where the original_topic_name could not be read from source or crew2 JSON:
{
  "port_type": "Publisher",
  "identifier": "pub_unknown",
  "is_connected": false,
  "connected_topic_or_service_name": ""   ← empty = truly unknown
}
```

**Purpose:**
Only truly unresolvable ports (empty topic names) stay `is_connected: false`.
These are rare edge cases where source extraction failed entirely for a port.

---

## STEP 4: Finalization

### [4.1 Consistency Check]

**Rule:**
Validate that the generated architecture is complete and internally consistent.

**Instructions:**

Perform the following checks:

**1. RosNodeParts Completeness:**
- [ ] Each RosNodePart has a valid `id`
- [ ] Each RosNodePart has a `class_name` assigned
- [ ] Each RosNodePart has a `full_namespace` assigned
- [ ] All RosNodeParts count matches initial calculation (Step 1.2)

**2. Port Population:**
- [ ] Each RosNodePart has a `ports[]` array (may be empty)
- [ ] Each port has: `port_type`, `identifier`, `message_or_service_type`, `connected_topic_or_service_name`, `is_connected`
- [ ] Port identifiers are unique within each RosNodePart
- [ ] No port has `port_type` other than: "Publisher", "Subscriber", "Client", "Service"

**3. Namespace Correctness:**
- [ ] All namespaces from launch_file_instance.namespace are present
- [ ] Each namespace contains RosNodePart IDs correctly assigned
- [ ] Namespace paths start with "/"
- [ ] No circular namespace dependencies

**4. Topic/Service Construction:**
- [ ] Each topic has `topic_name` and `message_type`
- [ ] Each service has `service_name` and `service_type`
- [ ] Topic/service names are unique within Arch_json
- [ ] All topics/services have at least one connected port

**5. Connection Validity:**
- [ ] Every port with a non-empty connected_topic_or_service_name has is_connected = true
- [ ] is_connected = false only for ports with empty or null connected_topic_or_service_name
- [ ] topics[] includes ALL unique topic names from ALL ports (not only matched pairs)
- [ ] External-only topics (publisher-only or subscriber-only within this system) ARE valid
      and appear in topics[] with is_connected = true on their ports

**6. Namespace Assignment:**
- [ ] All topics/services are assigned to appropriate namespaces
- [ ] Namespace assignments are consistent with topic/service names

**Error Handling:**
If any check fails:
- Log the specific failure with details
- Report which RosNodePart, port, or connection is problematic
- Provide suggestions for remediation
- Do NOT proceed to output if critical failures exist

**Example Check Output:**
```
✓ RosNodeParts: 3 created (matches expected count of 3)
✓ Ports: 8 total ports across all RosNodeParts
✗ Topic "/camera/image": multiple publishers detected (ERROR)
  - RosNodePart[rnp_1].pub_image
  - RosNodePart[rnp_2].pub_image_alt
✓ Namespaces: 3 namespaces correctly assigned
⚠ Unconnected ports: 2 ports remain without connection
  - RosNodePart[rnp_3].srv_enable (Service)
  - RosNodePart[rnp_4].sub_config (Subscriber)
```

---

### [4.2 Output]

**Rule:**
Generate and output the final Architecture_json in the specified format.

**Instructions:**

**Final Output Structure:**

```json
{
  "launch_file_name": "<launch_file_type>",
  "RosNodeParts": [
    {
      "id": "<unique_id>",
      "node_name": "<runtime_node_name or empty>",
      "class_name": "<class_name>",
      "full_namespace": "<full_namespace_path>",
      "ports": [
        {
          "is_connected": <boolean>,
          "message_or_service_type": "<type>",
          "connected_topic_or_service_name": "<name>",
          "port_type": "<Publisher|Subscriber|Client|Service>",
          "identifier": "<identifier>"
        }
      ]
    }
  ],
  "topics": [
    {
      "message_type": "<message_type>",
      "topic_name": "<topic_name>"
    }
  ],
  "services": [
    {
      "service_type": "<service_type>",
      "service_name": "<service_name>"
    }
  ],
  "namespaces": [
    "<namespace_path_1>",
    "<namespace_path_2>"
  ]
}
```

**Output Generation Steps:**

1. **Start with launch_file_name:**
   - Copy from `Arch_json.launch_file_name` (determined in Step 1.1)

2. **Include all RosNodeParts:**
   - Iterate through all created RosNodeParts
   - For each, include all fields: id, node_name, class_name, full_namespace, ports
   - Ensure ports array is complete with all connection information

3. **Include all Topics:**
   - Include all topics created in Step [3.4]
   - Each with message_type and topic_name

4. **Include all Services:**
   - Include all services created in Step [3.4]
   - Each with service_type and service_name

5. **Include all Namespaces:**
   - Extract unique namespace keys from `Arch_json.namespace`
   - Sort alphabetically for consistency

6. **Format and Serialize:**
   - Output as valid JSON
   - Use 2-space indentation for readability
   - Ensure no trailing commas
   - Validate JSON structure

**Example Output:**

```json
{
  "launch_file_name": "root.launch.py",
  "RosNodeParts": [
    {
      "id": "rnp_1",
      "node_name": "processor_main",
      "class_name": "ImageProcessor",
      "full_namespace": "/core/local",
      "ports": [
        {
          "is_connected": true,
          "message_or_service_type": "sensor_msgs/msg/Image",
          "connected_topic_or_service_name": "/camera/image_raw",
          "port_type": "Subscriber",
          "identifier": "sub_image"
        },
        {
          "is_connected": true,
          "message_or_service_type": "std_msgs/msg/String",
          "connected_topic_or_service_name": "/core/local/processed",
          "port_type": "Publisher",
          "identifier": "pub_processed"
        }
      ]
    },
    {
      "id": "rnp_2",
      "node_name": "camera_driver_front",
      "class_name": "CameraDriver",
      "full_namespace": "/front/cam",
      "ports": [
        {
          "is_connected": true,
          "message_or_service_type": "sensor_msgs/msg/Image",
          "connected_topic_or_service_name": "/camera/image_raw",
          "port_type": "Publisher",
          "identifier": "pub_image"
        },
        {
          "is_connected": true,
          "message_or_service_type": "std_srvs/srv/SetBool",
          "connected_topic_or_service_name": "/front/camera/enable",
          "port_type": "Service",
          "identifier": "srv_enable"
        }
      ]
    }
  ],
  "topics": [
    {
      "message_type": "sensor_msgs/msg/Image",
      "topic_name": "/camera/image_raw"
    },
    {
      "message_type": "std_msgs/msg/String",
      "topic_name": "/core/local/processed"
    }
  ],
  "services": [
    {
      "service_type": "std_srvs/srv/SetBool",
      "service_name": "/front/camera/enable"
    }
  ],
  "namespaces": [
    "/core",
    "/front",
    "/rear"
  ]
}
// NOTE: srv_enable has is_connected=true because its service name "/front/camera/enable"
// is resolved and non-empty — even though no Client for this service exists in this system.
// The "/front/camera/enable" service also appears in services[] for the same reason.
```

**CrewAI Tool Usage:**
Use `JSONSerializerTool` to format and validate the output:
```python
tool.serialize_to_json(
  architecture_object=Arch_json,
  validate_schema=True,
  format_indent=2,
  ensure_unique_ids=True
)
```

---

## CrewAI Integration Guidelines

This prompt is designed to be executed by a CrewAI Agent. The following CrewAI tools are referenced throughout:

| Tool Name | Steps Used | Purpose |
|-----------|-----------|---------|
| `ExtractFieldTool` | [1.1] | Extract fields from structured objects |
| `CountAndMapTool` | [1.2] | Count and create ID mappings |
| `ClassifierLookupTool` | [2.1.2] | Search Crew1_json for classifiers |
| `PortExtractionTool` | [2.1.5] | Extract ports from Crew2_json |
| `RemappingParserTool` | [2.1.6] | Parse launch file remappings |
| `TopicConstructorTool` | [3.4] | Create topic/service objects |
| `JSONSerializerTool` | [4.2] | Validate and format JSON output |

**Tool Usage Pattern:**
```
For each tool invocation:
1. Gather required input data
2. Call tool with appropriate parameters
3. Validate tool output
4. Apply result to Arch_json structure
5. Continue to next step
```

**Agent Instructions for Tool Usage:**
- Use tools to parallelize independent operations (e.g., multiple classifier lookups)
- Cache results from expensive operations (launch file parsing)
- Validate tool output before proceeding
- Log tool operations for debugging
- Handle tool failures gracefully with fallbacks

---

## Processing Workflow Summary

```
START: Process launch_file_instance

STEP 1: INITIALIZE
  [1.1] Extract launch file name
  [1.2] Count nodes and includes → RosNodeParts count
  [1.3] Extract namespace information
  
STEP 2: CONSTRUCT ROSNODEPARTS
  For each node_instance ID in launch_file_instance.nodes:
    [2.1.1] Assign basic attributes (name, id)
    [2.1.2] Resolve class name from Crew1_json
    [2.1.3] Compute full namespace path
    [2.1.4] Assign to namespace group
    [2.1.5] Extract ports from Crew2_json
    [2.1.6] Apply launch file remappings
  
  For each included_launch_file ID:
    [2.2.1] Retrieve sub-architecture from existing_Arch_json
    [2.2.2] Create RosNodePart wrapping sub-architecture
    [2.2.3] Extract unconnected ports
    [2.2.4] Apply namespace resolution to ports
  
STEP 3: CONNECT PORTS & CREATE TOPICS/SERVICES
  [3.1] Compute runtime_full_name for all ports
  [3.2] Create port tuples for clustering
  [3.3] Cluster ports by (name, type, msg_type)
  [3.4] Create Topic/Service objects from valid clusters
  [3.5] Assign topics/services to namespaces
  [3.6] Mark ports as connected
  [3.7] Handle unconnected ports
  
STEP 4: FINALIZE
  [4.1] Perform consistency checks
  [4.2] Generate Architecture_json output
  
END: Return Architecture_json
```

---

## Notes and Best Practices

1. **Recursive Processing Order:**
   - Process leaf launches first (no included_launch_files)
   - Then process parent launches
   - Continue until root launch is reached

2. **Namespace Handling:**
   - Always prepend "/" to absolute paths
   - Normalize double slashes
   - Maintain consistency in namespace separators

3. **Port Matching:**
   - Match port types carefully (Publisher ↔ Subscriber, Service ↔ Client)
   - One-to-many connections are valid (1 publisher → N subscribers)
   - Many-to-one is NOT valid

4. **Error Recovery:**
   - Log all errors with context
   - Use fallbacks where available
   - Do not silently skip problematic elements

5. **Performance Considerations:**
   - Cache Crew1_json and Crew2_json lookups
   - Parallelize independent port extractions
   - Reuse namespace computations

---

END OF ARCHITECTURE JSON CONSTRUCTION PROMPT
