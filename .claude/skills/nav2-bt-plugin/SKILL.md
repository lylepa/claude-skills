---
name: nav2-bt-plugin
description: >
  Scaffolds complete, production-ready custom BehaviorTree.CPP plugin packages
  for Navigation2 (Nav2). Use this skill whenever the user wants to create a
  new Nav2 behavior tree node of any kind — Action nodes (wrapping ROS2 action
  servers), Service nodes (wrapping ROS2 services), Condition nodes, or
  Decorator nodes. Also triggers when the user mentions "BT plugin", "behavior
  tree node", "Nav2 plugin", "bt_navigator plugin", "BtActionNode",
  "BtServiceNode", or asks to scaffold, generate, or create any Nav2 BT
  component. Always use this skill for Nav2 BT work — it encodes all best
  practices for ports, blackboard usage, lifecycle, unit tests, XML examples,
  CMakeLists, and GitHub Actions CI.
---

# Nav2 BehaviorTree.CPP Plugin Scaffolder

Generates a complete, compilable scaffold for a custom Nav2 BT plugin including:
- Header + implementation files (with correct base class and overrides)
- `CMakeLists.txt` additions
- `package.xml` dependencies
- GTest unit test file (with ROS2 fixture, fake servers, and edge-case tests)
- BT XML usage example
- XSD schema entry
- GitHub Actions CI snippet

---

## Step 1 — Gather Intent

Extract from the conversation what you already know, then ask only for what's missing. Confirm before generating.

**Required information:**

| Field | Question to ask |
|---|---|
| **Node type** | Action (wraps a ROS2 action server), Service (wraps a ROS2 service), Condition (checks a boolean fact), or Decorator (wraps one child and modifies its result)? |
| **Node name** | What should the C++ class and BT XML tag be called? (e.g. `GlobalLocalizationService`, `RetryUntilPoseConverged`) |
| **ROS2 interface** | What action/service/message type does it use? (e.g. `std_srvs/srv/Empty`, `nav2_msgs/action/NavigateToPose`) |
| **Input ports** | What inputs does this node take from the blackboard? Name, C++ type, default value, description for each. |
| **Output ports** | What does it write back to the blackboard? |
| **ROS distro** | `humble`, `iron`, or `jazzy`? (affects `behaviortree_cpp_v3` vs `behaviortree_cpp`) |
| **Package name** | What is the ROS2 package name? (e.g. `my_bt_plugins`) |
| **Namespace** | C++ namespace (e.g. `my_bt_plugins`) |

**Optional but useful:**
- What does success/failure mean for this node?
- Are there timeout or retry requirements?
- Any specific blackboard keys it should read from (set by other nodes)?

---

## Step 2 — Select Template

Based on node type, read the corresponding reference file **before generating any code**:

| Node type | Reference file | Base class |
|---|---|---|
| Action | `references/action_node.md` | `nav2_behavior_tree::BtActionNode<ActionT>` |
| Service | `references/service_node.md` | `nav2_behavior_tree::BtServiceNode<ServiceT>` |
| Condition | `references/condition_node.md` | `BT::ConditionNode` |
| Decorator | `references/decorator_node.md` | `BT::DecoratorNode` |

Read the test template too: `references/test_template.md`

---

## Step 3 — Generate Scaffold

Produce **all** of the following in one response, clearly separated by filename headers.

### File generation checklist

- [ ] `include/<package_name>/<snake_case_name>.hpp`
- [ ] `src/<snake_case_name>.cpp`
- [ ] CMakeLists.txt additions (as a clearly-marked diff/snippet)
- [ ] package.xml dependency additions
- [ ] `test/test_<snake_case_name>.cpp`
- [ ] `bt_trees/<snake_case_name>_example.xml`
- [ ] XSD schema entry snippet (to add to `schemas/bt_tree.xsd`)
- [ ] GitHub Actions CI snippet (if the user has a `bt_validation.yml`)

### Naming conventions

| Concept | Convention | Example |
|---|---|---|
| C++ class | `PascalCase` | `GlobalLocalizationService` |
| BT XML tag | Same as C++ class | `<GlobalLocalizationService .../>` |
| File names | `snake_case` | `global_localization_service` |
| Port names | `snake_case` | `server_timeout`, `goal_pose` |
| Blackboard keys | `snake_case` in `{braces}` | `{nav_path}`, `{localized}` |

---

## Step 4 — Apply Best Practices

Every generated file **must** follow these rules. Do not skip any.

### Ports
- Always call `providedBasicPorts({...})` for Action/Service nodes (includes `server_name`, `server_timeout`)
- Input ports: use `BT::InputPort<T>(name, default, description)`
- Output ports: use `BT::OutputPort<T>(name, description)`
- Port types must exactly match the C++ types used in `getInput<T>()` / `setOutput()`
- Blackboard references in XML always use `{key}` syntax

### Lifecycle
- Action nodes: override `on_tick()`, `on_success()`, `on_aborted()`, `on_cancelled()`
- Service nodes: override `on_tick()`, `on_completion(response)`
- Decorator nodes: override `tick()` AND `halt()` — `halt()` must reset all state and halt the child
- Condition nodes: override `tick()` only
- Decorators must call `child_node_->executeTick()` (never `tick()` directly)

### Blackboard access
- Get the rclcpp node via: `config().blackboard->get<rclcpp::Node::SharedPtr>("node")`
- Use a dedicated `MutuallyExclusive` callback group with `false` (not added to node executor)
- Add the callback group to a `SingleThreadedExecutor` and call `spin_some()` in `tick()`

### Registration macro
- Always use `BT_REGISTER_NODES(factory)` at the bottom of the `.cpp` file
- Action/Service nodes require a builder lambda that passes the default service/action name
- Condition and Decorator nodes use `factory.registerNodeType<ClassName>("XmlTagName")`

### ROS distro compatibility
- `humble` / `iron`: `#include "behaviortree_cpp_v3/..."`, package `behaviortree_cpp_v3`
- `jazzy`+: `#include "behaviortree_cpp/..."`, package `behaviortree_cpp`

---

## Step 5 — Generate Tests

Every scaffold includes a test file. Follow `references/test_template.md` exactly.

**Minimum test cases for each node type:**

| Node type | Required test cases |
|---|---|
| Action | happy path, action server unavailable, output ports set, goal populated correctly |
| Service | happy path, service unavailable, output ports set, called exactly once |
| Condition | returns SUCCESS when true, returns FAILURE when false, blackboard value used |
| Decorator | child success → decorator behaviour, child failure propagation, max retries/limits, halt resets state, state tracked correctly |

---

## Step 6 — Present Summary

After generating all files, show a compact summary table:

```
Generated files:
  include/my_bt_plugins/my_node.hpp          ← C++ header
  src/my_node.cpp                            ← Implementation + BT_REGISTER_NODES
  test/test_my_node.cpp                      ← GTest unit tests
  bt_trees/my_node_example.xml               ← BT XML usage example
  [snippets for CMakeLists.txt, package.xml, bt_tree.xsd, bt_validation.yml]

Next steps:
  1. Add snippets to CMakeLists.txt and package.xml
  2. colcon build --packages-select <pkg> --cmake-args -DBUILD_TESTING=ON
  3. colcon test --packages-select <pkg>
  4. Open bt_trees/my_node_example.xml in Groot2 to verify layout
```

---

## Quick Reference — Key Nav2 BT Patterns

```cpp
// Get the node from blackboard (all Nav2 BT node constructors do this)
node_ = config().blackboard->get<rclcpp::Node::SharedPtr>("node");

// Read an input port
double threshold;
getInput("my_input_port", threshold);

// Write an output port
setOutput("my_output_port", result_value);

// Tick a decorator's child (never call child->tick() directly)
BT::NodeStatus status = child_node_->executeTick();

// Reset covariance / uncertainty tracking in halt()
void MyDecorator::halt() {
  if (child_node_->status() == BT::NodeStatus::RUNNING) {
    child_node_->halt();
  }
  my_state_ = initial_value_;
  BT::DecoratorNode::halt();
}
```
