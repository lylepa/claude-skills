# Service Node Template

Use when the node calls a **ROS2 service** (request/response, short-lived).
Examples: `/reinitialize_global_localization`, `/clear_costmap`.

Base class: `nav2_behavior_tree::BtServiceNode<ServiceT>`

---

## Header Template

```cpp
// include/<package_name>/<snake_name>.hpp
#pragma once

#include <string>
#include <memory>

#include "<service_package>/srv/<service_type>.hpp"
#include "nav2_behavior_tree/bt_service_node.hpp"

namespace <namespace>
{

/**
 * @brief <OneLiner description>
 *
 * Calls the ROS2 service: <service_name>
 *
 * Input ports:
 *   - <port_name> (<type>): <description>
 *
 * Output ports:
 *   - <port_name> (<type>): <description>
 */
class <ClassName>
  : public nav2_behavior_tree::BtServiceNode<<pkg>::srv::<ServiceType>>
{
public:
  using Service = <pkg>::srv::<ServiceType>;

  <ClassName>(
    const std::string & xml_tag_name,
    const std::string & service_name,
    const BT::NodeConfiguration & conf);

  /// Populate request_ fields. Called once before the async call is sent.
  void on_tick() override;

  /// Called when the service response arrives.
  BT::NodeStatus on_completion(
    std::shared_ptr<Service::Response> response) override;

  static BT::PortsList providedPorts();
};

}  // namespace <namespace>
```

---

## Implementation Template

```cpp
// src/<snake_name>.cpp
#include "<package_name>/<snake_name>.hpp"
#include "behaviortree_cpp_v3/bt_factory.h"   // behaviortree_cpp for jazzy+

namespace <namespace>
{

<ClassName>::<ClassName>(
  const std::string & xml_tag_name,
  const std::string & service_name,
  const BT::NodeConfiguration & conf)
: BtServiceNode<Service>(xml_tag_name, service_name, conf)
{
  // BtServiceNode creates the rclcpp service client automatically.
}

BT::PortsList <ClassName>::providedPorts()
{
  // providedBasicPorts() adds "server_name" and "server_timeout"
  return providedBasicPorts({
    // --- Input ports ---
    BT::InputPort<TYPE>("port_name", DEFAULT, "Description"),

    // --- Output ports ---
    BT::OutputPort<TYPE>("port_name", "Description"),
  });
}

void <ClassName>::on_tick()
{
  // For std_srvs::srv::Empty the request has no fields.
  // For other services, populate request_ here:
  //
  // TYPE value;
  // getInput("port_name", value);
  // request_-><field> = value;

  RCLCPP_INFO(node_->get_logger(),
    "[<ClassName>] Sending request to %s", service_name_.c_str());
}

BT::NodeStatus <ClassName>::on_completion(
  std::shared_ptr<Service::Response> response)
{
  // Inspect response fields (empty for std_srvs::srv::Empty).
  // Set output ports.
  setOutput("port_name", <value_from_response>);

  RCLCPP_INFO(node_->get_logger(), "[<ClassName>] Service call succeeded.");
  return BT::NodeStatus::SUCCESS;

  // Return FAILURE here if the response indicates an error:
  // if (!response->success) {
  //   RCLCPP_WARN(...);
  //   return BT::NodeStatus::FAILURE;
  // }
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
        "<default_service_name>",   // e.g. "/reinitialize_global_localization"
        config);
    };

  factory.registerBuilder<<namespace>::<ClassName>>("<ClassName>", builder);
}
```

---

## Key Overrides Reference

| Override | When called | What to do |
|---|---|---|
| `on_tick()` | Once, before async request | Populate `request_` fields from input ports |
| `on_completion(response)` | Response received | Inspect response, set output ports, return SUCCESS/FAILURE |

**Do not** override `tick()` — `BtServiceNode` sends the request, spins until
the response arrives (or timeout), then calls `on_completion()`.

---

## Handling Empty Responses (std_srvs/srv/Empty)

`std_srvs::srv::Empty` has no fields in request or response.
`on_tick()` body can be empty (just log).
`on_completion()` receives an empty response — always return SUCCESS unless
the service threw an exception (BtServiceNode handles that as FAILURE).

---

## CMakeLists.txt snippet

```cmake
find_package(<service_package> REQUIRED)   # e.g. std_srvs

add_library(<package_name> SHARED
  src/<snake_name>.cpp
)

ament_target_dependencies(<package_name>
  nav2_behavior_tree
  behaviortree_cpp_v3
  pluginlib
  rclcpp
  <service_package>
)
```

---

## XML Usage Example

```xml
<root main_tree_to_execute="MainTree" BTCPP_format="4">
  <BehaviorTree ID="MainTree">
    <Sequence>
      <<ClassName>
        service_name="<ros_service_name>"
        server_timeout="3000"
        port_name="{blackboard_key}"
        output_port="{result_key}"/>
    </Sequence>
  </BehaviorTree>
</root>
```
