# Condition Node Template

Use when the node **checks a boolean fact** and immediately returns SUCCESS or
FAILURE. No child nodes. Examples: `IsStuck`, `IsBatteryLow`, `GoalReached`.

Base class: `BT::ConditionNode`

Condition nodes are **synchronous** — they must return SUCCESS or FAILURE
immediately on each tick (never RUNNING).

---

## Header Template

```cpp
// include/<package_name>/<snake_name>.hpp
#pragma once

#include <string>
#include <atomic>

#include "rclcpp/rclcpp.hpp"
#include "behaviortree_cpp_v3/condition_node.h"  // behaviortree_cpp for jazzy+
#include "<msg_package>/msg/<MsgType>.hpp"

namespace <namespace>
{

/**
 * @brief Returns SUCCESS when <condition>, FAILURE otherwise.
 *
 * Subscribes to <topic_name> and evaluates <condition_logic>.
 *
 * Input ports:
 *   - <port_name> (<type>, default=<default>): <description>
 */
class <ClassName> : public BT::ConditionNode
{
public:
  <ClassName>(
    const std::string & name,
    const BT::NodeConfiguration & conf);

  static BT::PortsList providedPorts();

private:
  BT::NodeStatus tick() override;

  void topicCallback(const <MsgType>::SharedPtr msg);

  rclcpp::Node::SharedPtr node_;
  rclcpp::CallbackGroup::SharedPtr callback_group_;
  rclcpp::executors::SingleThreadedExecutor callback_group_executor_;
  rclcpp::Subscription<<MsgType>>::SharedPtr subscription_;

  // Condition state — updated in callback, read in tick()
  std::atomic<double> current_value_{0.0};
  std::atomic<bool>   has_data_{false};
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
: BT::ConditionNode(name, conf)
{
  node_ = config().blackboard->get<rclcpp::Node::SharedPtr>("node");

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
    BT::InputPort<double>("threshold", DEFAULT, "Description of the threshold"),
    // Add more ports as needed
  };
}

BT::NodeStatus <ClassName>::tick()
{
  // Drain any pending topic messages
  callback_group_executor_.spin_some();

  // Condition nodes must NEVER return RUNNING
  if (!has_data_) {
    RCLCPP_WARN_ONCE(node_->get_logger(),
      "[<ClassName>] No data received yet on <topic_name>, returning FAILURE.");
    return BT::NodeStatus::FAILURE;
  }

  double threshold;
  getInput("threshold", threshold);

  if (current_value_ < threshold) {
    RCLCPP_DEBUG(node_->get_logger(),
      "[<ClassName>] Condition MET: %.4f < %.4f", current_value_.load(), threshold);
    return BT::NodeStatus::SUCCESS;
  }

  RCLCPP_DEBUG(node_->get_logger(),
    "[<ClassName>] Condition NOT met: %.4f >= %.4f", current_value_.load(), threshold);
  return BT::NodeStatus::FAILURE;
}

void <ClassName>::topicCallback(const <MsgType>::SharedPtr msg)
{
  current_value_ = <extract_value_from_msg>;
  has_data_      = true;
}

}  // namespace <namespace>

// ── Plugin registration ───────────────────────────────────────────────────────
BT_REGISTER_NODES(factory)
{
  factory.registerNodeType<<namespace>::<ClassName>>("<ClassName>");
}
```

---

## Blackboard-Only Variant (no subscription)

When the value is already on the blackboard (set by another node):

```cpp
BT::NodeStatus <ClassName>::tick()
{
  double threshold, current;
  getInput("threshold", threshold);

  // Read from blackboard directly
  if (!config().blackboard->get<double>("some_shared_key", current)) {
    RCLCPP_WARN(node_->get_logger(),
      "[<ClassName>] Key 'some_shared_key' not set on blackboard.");
    return BT::NodeStatus::FAILURE;
  }

  return (current < threshold)
    ? BT::NodeStatus::SUCCESS
    : BT::NodeStatus::FAILURE;
}
```

---

## XML Usage Example

```xml
<root main_tree_to_execute="MainTree" BTCPP_format="4">
  <BehaviorTree ID="MainTree">
    <Sequence>
      <!-- Condition checked every tick — if FAILURE, Sequence stops -->
      <<ClassName>
        threshold="0.2"/>

      <SomeActionNode .../>
    </Sequence>
  </BehaviorTree>
</root>
```

---

## XSD Schema Entry

```xml
<!-- In the NodeGroup choice: -->
<xs:element name="<ClassName>" type="<ClassName>Type"/>

<xs:complexType name="<ClassName>Type">
  <xs:attribute name="name"      type="xs:string" use="optional"/>
  <xs:attribute name="threshold" type="xs:string" use="optional"/>
  <!-- one xs:attribute per port -->
</xs:complexType>
```
