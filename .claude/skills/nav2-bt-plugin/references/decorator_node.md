# Decorator Node Template

Use when the node **wraps a single child** and modifies when/how that child
is ticked or what result it returns. Examples: `Retry`, `Timeout`,
`RetryUntilPoseConverged`.

Base class: `BT::DecoratorNode`

---

## Critical Decorator Rules

1. **Exactly one child** — enforced by the framework and the XSD schema.
2. **Always call `child_node_->executeTick()`** — never `child_node_->tick()`.
3. **Always implement `halt()`** — reset ALL member state and halt the child.
4. **Return RUNNING to stay in control** — prevents parent from advancing prematurely.
5. **Never hold the BT executor thread** — use `spin_some()` + sleep, not `spin()`.

---

## Header Template

```cpp
// include/<package_name>/<snake_name>.hpp
#pragma once

#include <string>
#include <memory>
#include <limits>

#include "rclcpp/rclcpp.hpp"
#include "behaviortree_cpp_v3/decorator_node.h"  // behaviortree_cpp for jazzy+
#include "nav2_behavior_tree/bt_utils.hpp"

// Include any message type headers needed for subscriptions:
// #include "<msg_package>/msg/<MsgType>.hpp"

namespace <namespace>
{

/**
 * @brief <OneLiner description>
 *
 * Wraps its single child and re-ticks it according to <logic description>.
 *
 * Input ports:
 *   - <port_name> (<type>, default=<default>): <description>
 *
 * Output ports:
 *   - <port_name> (<type>): <description>
 */
class <ClassName> : public BT::DecoratorNode
{
public:
  <ClassName>(
    const std::string & name,
    const BT::NodeConfiguration & conf);

  static BT::PortsList providedPorts();

private:
  BT::NodeStatus tick() override;
  void halt() override;

  // Example: subscription callback to track external state
  void topicCallback(const <MsgType>::SharedPtr msg);

  // ROS2 infrastructure
  rclcpp::Node::SharedPtr node_;
  rclcpp::CallbackGroup::SharedPtr callback_group_;
  rclcpp::executors::SingleThreadedExecutor callback_group_executor_;
  rclcpp::Subscription<<MsgType>>::SharedPtr subscription_;

  // State that must be reset in halt()
  int    retry_count_{0};
  double current_value_{std::numeric_limits<double>::max()};
  bool   is_initialized_{false};
};

}  // namespace <namespace>
```

---

## Implementation Template

```cpp
// src/<snake_name>.cpp
#include "<package_name>/<snake_name>.hpp"
#include "behaviortree_cpp_v3/bt_factory.h"

namespace <namespace>
{

<ClassName>::<ClassName>(
  const std::string & name,
  const BT::NodeConfiguration & conf)
: BT::DecoratorNode(name, conf)
{
  // Grab the rclcpp node Nav2 stores on the blackboard
  node_ = config().blackboard->get<rclcpp::Node::SharedPtr>("node");

  // Dedicated callback group — false = NOT added to node's own executor
  callback_group_ = node_->create_callback_group(
    rclcpp::CallbackGroupType::MutuallyExclusive, false);

  rclcpp::SubscriptionOptions opts;
  opts.callback_group = callback_group_;

  subscription_ = node_->create_subscription<<MsgType>>(
    "<topic_name>",
    rclcpp::SystemDefaultsQoS(),
    std::bind(&<ClassName>::topicCallback, this, std::placeholders::_1),
    opts);

  callback_group_executor_.add_callback_group(
    callback_group_, node_->get_node_base_interface());
}

BT::PortsList <ClassName>::providedPorts()
{
  return {
    BT::InputPort<TYPE>("port_name", DEFAULT, "Description"),
    BT::OutputPort<TYPE>("port_name", "Description"),
  };
}

BT::NodeStatus <ClassName>::tick()
{
  // 1. Spin the subscriber to receive any waiting messages
  callback_group_executor_.spin_some();

  // 2. Read input ports
  TYPE threshold;
  int  max_retries;
  getInput("threshold_port", threshold);
  getInput("max_retries",    max_retries);

  // 3. Check completion condition BEFORE ticking child
  //    (short-circuit if already done)
  if (current_value_ < threshold) {
    setOutput("output_port", true);
    setOutput("retry_count", retry_count_);
    return BT::NodeStatus::SUCCESS;
  }

  // 4. Hard limit check
  if (max_retries >= 0 && retry_count_ >= max_retries) {
    RCLCPP_WARN(node_->get_logger(),
      "[<ClassName>] Max retries (%d) reached.", max_retries);
    setOutput("output_port", false);
    return BT::NodeStatus::FAILURE;
  }

  // 5. Tick the child
  const BT::NodeStatus child_status = child_node_->executeTick();

  switch (child_status) {
    case BT::NodeStatus::RUNNING:
      return BT::NodeStatus::RUNNING;

    case BT::NodeStatus::FAILURE:
      setOutput("retry_count", ++retry_count_);
      return BT::NodeStatus::FAILURE;

    case BT::NodeStatus::SUCCESS:
      retry_count_++;
      setOutput("retry_count", retry_count_);
      // Child succeeded — now wait/check for external condition
      // Return RUNNING to keep control; will check again next tick
      return BT::NodeStatus::RUNNING;

    default:
      return BT::NodeStatus::FAILURE;
  }
}

void <ClassName>::halt()
{
  // ── CRITICAL: reset ALL member state ────────────────────────────────────
  retry_count_    = 0;
  current_value_  = std::numeric_limits<double>::max();
  is_initialized_ = false;

  // ── CRITICAL: halt the child if it's running ─────────────────────────────
  if (child_node_->status() == BT::NodeStatus::RUNNING) {
    child_node_->halt();
  }

  // ── CRITICAL: call the base class halt() last ─────────────────────────────
  BT::DecoratorNode::halt();
}

void <ClassName>::topicCallback(const <MsgType>::SharedPtr msg)
{
  // Update state from the topic — keep this lightweight
  current_value_ = <extract_value_from_msg>;
  RCLCPP_DEBUG(node_->get_logger(),
    "[<ClassName>] Received update: value=%.4f", current_value_);
}

}  // namespace <namespace>

// ── Plugin registration ───────────────────────────────────────────────────────
BT_REGISTER_NODES(factory)
{
  factory.registerNodeType<<namespace>::<ClassName>>("<ClassName>");
}
```

---

## State Machine Pattern

```
Entry
  │
  ├─ Completion condition met? ──YES──► SUCCESS
  │
  ├─ Hard limit hit?           ──YES──► FAILURE
  │
  └─ Tick child
          │
    ┌─────┴──────────┬─────────────┐
  RUNNING        SUCCESS        FAILURE
    │               │               │
  RUNNING     update state,    propagate or
              return RUNNING   return FAILURE
              (check again
               next tick)
```

---

## XML Usage Example

```xml
<root main_tree_to_execute="MainTree" BTCPP_format="4">
  <BehaviorTree ID="MainTree">

    <!-- Decorator wraps exactly ONE child -->
    <<ClassName>
      port_name="{blackboard_key}"
      max_retries="5"
      output_port="{result}">

      <SomeChildNode .../>

    </<ClassName>>

  </BehaviorTree>
</root>
```

---

## XSD Schema Entry

```xml
<!-- In the NodeGroup choice, add: -->
<xs:element name="<ClassName>" type="<ClassName>Type"/>

<!-- Add the type definition: -->
<xs:complexType name="<ClassName>Type">
  <xs:sequence>
    <!-- Exactly one child -->
    <xs:group ref="NodeGroup" minOccurs="1" maxOccurs="1"/>
  </xs:sequence>
  <xs:attribute name="name"       type="xs:string" use="optional"/>
  <xs:attribute name="port_name"  type="xs:string" use="optional"/>
  <xs:attribute name="max_retries" type="xs:string" use="optional"/>
  <!-- add one xs:attribute per port -->
</xs:complexType>
```
