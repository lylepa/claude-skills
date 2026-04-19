# Action Node Template

Use when the node communicates with a **ROS2 action server** (long-running,
supports feedback and cancellation). Examples: `NavigateToPose`,
`ComputePathToPose`, `FollowPath`.

Base class: `nav2_behavior_tree::BtActionNode<ActionT>`

---

## Header Template

```cpp
// include/<package_name>/<snake_name>.hpp
#pragma once

#include <string>
#include <memory>

#include "<action_package>/action/<action_type>.hpp"
#include "nav2_behavior_tree/bt_action_node.hpp"

namespace <namespace>
{

/**
 * @brief <OneLiner description>
 *
 * Input ports:
 *   - <port_name> (<type>): <description>
 *
 * Output ports:
 *   - <port_name> (<type>): <description>
 */
class <ClassName>
  : public nav2_behavior_tree::BtActionNode<<pkg>::action::<ActionType>>
{
public:
  using Action = <pkg>::action::<ActionType>;

  <ClassName>(
    const std::string & xml_tag_name,
    const std::string & action_name,
    const BT::NodeConfiguration & conf);

  /// Populate goal_ fields from input ports. Called before goal is sent.
  void on_tick() override;

  /// Called when the action server reports SUCCESS.
  BT::NodeStatus on_success() override;

  /// Called when the action server reports ABORTED.
  BT::NodeStatus on_aborted() override;

  /// Called when the action is cancelled (tree halted mid-flight).
  BT::NodeStatus on_cancelled() override;

  /// Declare all ports this node exposes.
  static BT::PortsList providedPorts();
};

}  // namespace <namespace>
```

---

## Implementation Template

```cpp
// src/<snake_name>.cpp
#include "<package_name>/<snake_name>.hpp"
#include "behaviortree_cpp_v3/bt_factory.h"   // use behaviortree_cpp for jazzy+

namespace <namespace>
{

<ClassName>::<ClassName>(
  const std::string & xml_tag_name,
  const std::string & action_name,
  const BT::NodeConfiguration & conf)
: BtActionNode<Action>(xml_tag_name, action_name, conf)
{}

BT::PortsList <ClassName>::providedPorts()
{
  return providedBasicPorts({
    // --- Input ports ---
    BT::InputPort<TYPE>("port_name", DEFAULT, "Description"),

    // --- Output ports ---
    BT::OutputPort<TYPE>("port_name", "Description"),
  });
}

void <ClassName>::on_tick()
{
  // Populate goal_ fields from input ports.
  // goal_ is a pre-allocated Action::Goal — just set its fields.
  TYPE value;
  getInput("port_name", value);
  goal_.<field> = value;

  // Increment a counter, log, or check a precondition here if needed.
  RCLCPP_DEBUG(node_->get_logger(), "[<ClassName>] Sending goal.");
}

BT::NodeStatus <ClassName>::on_success()
{
  // result_.result is the action server's result message.
  // Write any output ports here.
  setOutput("port_name", result_.result-><field>);

  RCLCPP_INFO(node_->get_logger(), "[<ClassName>] Action succeeded.");
  return BT::NodeStatus::SUCCESS;
}

BT::NodeStatus <ClassName>::on_aborted()
{
  RCLCPP_WARN(node_->get_logger(), "[<ClassName>] Action aborted.");
  return BT::NodeStatus::FAILURE;
}

BT::NodeStatus <ClassName>::on_cancelled()
{
  RCLCPP_INFO(node_->get_logger(), "[<ClassName>] Action cancelled.");
  return BT::NodeStatus::SUCCESS;
}

}  // namespace <namespace>

// ── Plugin registration ───────────────────────────────────────────────────────
BT_REGISTER_NODES(factory)
{
  BT::NodeBuilder builder =
    [](const std::string & name, const BT::NodeConfiguration & config)
    {
      return std::make_unique<<namespace>::<ClassName>>(
        name,
        "<default_action_server_name>",  // e.g. "navigate_to_pose"
        config);
    };

  factory.registerBuilder<<namespace>::<ClassName>>("<ClassName>", builder);
}
```

---

## Key Overrides Reference

| Override | When called | What to do |
|---|---|---|
| `on_tick()` | Every BT tick while RUNNING | Populate `goal_` from input ports |
| `on_success()` | Action server returned SUCCEED | Set output ports from `result_.result`, return SUCCESS |
| `on_aborted()` | Action server returned ABORTED | Return FAILURE (usually) |
| `on_cancelled()` | Tree halted mid-action | Return SUCCESS (clean shutdown) |

**Do not** override `tick()` directly — `BtActionNode` manages the action
client lifecycle (goal sending, feedback callbacks, cancellation) for you.

---

## CMakeLists.txt snippet

```cmake
find_package(<action_package> REQUIRED)

add_library(<package_name> SHARED
  src/<snake_name>.cpp
  # ... other sources
)

ament_target_dependencies(<package_name>
  nav2_behavior_tree
  behaviortree_cpp_v3       # behaviortree_cpp for jazzy+
  pluginlib
  rclcpp
  <action_package>
)

install(TARGETS <package_name> DESTINATION lib)
install(DIRECTORY include/ DESTINATION include/)
```

---

## XML Usage Example

```xml
<root main_tree_to_execute="MainTree" BTCPP_format="4">
  <BehaviorTree ID="MainTree">
    <Sequence>
      <<ClassName>
        action_name="<server_name>"
        server_timeout="5000"
        port_name="{blackboard_key}"
        output_port="{result_key}"/>
    </Sequence>
  </BehaviorTree>
</root>
```
