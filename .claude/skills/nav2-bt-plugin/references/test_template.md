# Test Template Reference

All generated test files follow this structure. Copy the relevant sections
based on node type.

---

## File Structure

```cpp
// test/test_<snake_name>.cpp

// 1. Shared fixture include
#include "bt_test_fixture.hpp"

// 2. Node under test
#include "<package_name>/<snake_name>.hpp"

// 3. ROS2 message/service/action headers
#include "<msg_package>/<category>/<Type>.hpp"

// 4. Test namespace
namespace <namespace>::test { ... }
```

---

## Shared Fixture (bt_test_fixture.hpp)

Generate this file once per package — shared by all test files.

```cpp
// test/bt_test_fixture.hpp
#pragma once

#include <memory>
#include <string>
#include <chrono>
#include <thread>

#include "gtest/gtest.h"
#include "rclcpp/rclcpp.hpp"
#include "behaviortree_cpp_v3/bt_factory.h"
#include "behaviortree_cpp_v3/blackboard.h"

namespace <namespace>::test
{

class BtTestFixture : public ::testing::Test
{
protected:
  void SetUp() override
  {
    node_ = rclcpp::Node::make_shared(
      "bt_test_node_" + std::to_string(counter_++));

    blackboard_ = BT::Blackboard::create();
    blackboard_->set<rclcpp::Node::SharedPtr>("node", node_);
    blackboard_->set<std::chrono::milliseconds>(
      "server_timeout", std::chrono::milliseconds(2000));
    blackboard_->set<std::chrono::milliseconds>(
      "bt_loop_duration", std::chrono::milliseconds(10));
  }

  void TearDown() override
  {
    tree_.reset();
    node_.reset();
  }

  void buildTree(const std::string & xml)
  {
    tree_ = std::make_unique<BT::Tree>(
      factory_.createTreeFromText(xml, blackboard_));
  }

  BT::NodeStatus tickUntilDone(int max_ticks = 100)
  {
    BT::NodeStatus status = BT::NodeStatus::RUNNING;
    for (int i = 0; i < max_ticks && status == BT::NodeStatus::RUNNING; ++i) {
      status = tree_->tickRoot();
      rclcpp::spin_some(node_);
      std::this_thread::sleep_for(std::chrono::milliseconds(10));
    }
    return status;
  }

  rclcpp::Node::SharedPtr   node_;
  BT::Blackboard::Ptr       blackboard_;
  BT::BehaviorTreeFactory   factory_;
  std::unique_ptr<BT::Tree> tree_;

private:
  static inline int counter_ = 0;
};

class ROS2Environment : public ::testing::Environment
{
public:
  void SetUp()    override { rclcpp::init(0, nullptr); }
  void TearDown() override { rclcpp::shutdown(); }
};

}  // namespace <namespace>::test
```

---

## test_main.cpp

```cpp
// test/test_main.cpp
#include "gtest/gtest.h"
#include "bt_test_fixture.hpp"

int main(int argc, char ** argv)
{
  ::testing::InitGoogleTest(&argc, argv);
  ::testing::AddGlobalTestEnvironment(
    new <namespace>::test::ROS2Environment);
  return RUN_ALL_TESTS();
}
```

---

## Service Node Tests

```cpp
// ── Fake server ───────────────────────────────────────────────────────────────
class Fake<ServiceType>Server
{
public:
  explicit Fake<ServiceType>Server(rclcpp::Node::SharedPtr node)
  {
    server_ = node->create_service<<Service>>(
      "<service_name>",
      [this](
        const <Service>::Request::SharedPtr /*req*/,
        <Service>::Response::SharedPtr /*res*/)
      {
        call_count_++;
        if (should_fail_) {
          throw std::runtime_error("Simulated failure");
        }
        // Populate response fields if not Empty
      });
  }
  int  call_count_{0};
  bool should_fail_{false};
private:
  rclcpp::Service<<Service>>::SharedPtr server_;
};

// ── Fixture ───────────────────────────────────────────────────────────────────
class <ClassName>Test : public BtTestFixture
{
protected:
  void SetUp() override
  {
    BtTestFixture::SetUp();
    factory_.registerBuilder<<ClassName>>(
      "<ClassName>",
      [](const std::string & name, const BT::NodeConfiguration & cfg) {
        return std::make_unique<<ClassName>>(name, "<service_name>", cfg);
      });
    fake_server_ = std::make_shared<Fake<ServiceType>Server>(node_);
  }
  std::shared_ptr<Fake<ServiceType>Server> fake_server_;
};

// ── Tests ─────────────────────────────────────────────────────────────────────
TEST_F(<ClassName>Test, HappyPath_ReturnsSuccess)
{
  buildTree(R"(
    <root main_tree_to_execute="T">
      <BehaviorTree ID="T">
        <<ClassName>
          service_name="<service_name>"
          server_timeout="2000"
          output_port="{result}"/>
      </BehaviorTree>
    </root>
  )");

  EXPECT_EQ(tickUntilDone(), BT::NodeStatus::SUCCESS);
  EXPECT_EQ(fake_server_->call_count_, 1);
}

TEST_F(<ClassName>Test, OutputPortsSetOnSuccess)
{
  buildTree(/* xml with output_port="{result}" */);
  tickUntilDone();
  EXPECT_NO_THROW(blackboard_->get<TYPE>("result"));
}

TEST_F(<ClassName>Test, ServiceCalledExactlyOnce)
{
  buildTree(/* xml */);
  tickUntilDone();
  EXPECT_EQ(fake_server_->call_count_, 1);
}

TEST_F(<ClassName>Test, FailsWhenServiceUnavailable)
{
  // Use a nonexistent service name — no fake server
  buildTree(R"(
    <root main_tree_to_execute="T">
      <BehaviorTree ID="T">
        <<ClassName> service_name="/does_not_exist" server_timeout="300"/>
      </BehaviorTree>
    </root>
  )");

  EXPECT_EQ(tickUntilDone(50), BT::NodeStatus::FAILURE);
}
```

---

## Action Node Tests

```cpp
// ── Fake action server ────────────────────────────────────────────────────────
class Fake<ActionType>Server
{
public:
  using GoalHandle = rclcpp_action::ServerGoalHandle<<Action>>;

  explicit Fake<ActionType>Server(rclcpp::Node::SharedPtr node)
  {
    server_ = rclcpp_action::create_server<<Action>>(
      node, "<action_name>",
      [](const rclcpp_action::GoalUUID &, std::shared_ptr<<Action>::Goal>) {
        return rclcpp_action::GoalResponse::ACCEPT_AND_EXECUTE;
      },
      [](std::shared_ptr<GoalHandle>) {
        return rclcpp_action::CancelResponse::ACCEPT;
      },
      [this](std::shared_ptr<GoalHandle> gh) {
        call_count_++;
        auto result = std::make_shared<<Action>::Result>();
        // populate result fields
        if (should_abort_) { gh->abort(result); }
        else               { gh->succeed(result); }
      });
  }
  int  call_count_{0};
  bool should_abort_{false};
private:
  rclcpp_action::Server<<Action>>::SharedPtr server_;
};

// ── Tests ─────────────────────────────────────────────────────────────────────
TEST_F(<ClassName>Test, HappyPath_ReturnsSuccess) { /* ... */ }
TEST_F(<ClassName>Test, GoalPopulatedFromInputPorts) { /* verify goal fields */ }
TEST_F(<ClassName>Test, OutputPortsSetFromResult) { /* ... */ }
TEST_F(<ClassName>Test, ReturnsFailureWhenAborted) { /* set should_abort_=true */ }
TEST_F(<ClassName>Test, FailsWhenActionServerUnavailable) { /* no server */ }
```

---

## Decorator Node Tests

```cpp
// ── Helpers ───────────────────────────────────────────────────────────────────
// Publish a fake message to the topic the decorator subscribes to
void publishFake<MsgType>(
  rclcpp::Node::SharedPtr node, double value)
{
  auto pub = node->create_publisher<<MsgType>>("<topic>", 10);
  <MsgType> msg;
  msg.<field> = value;
  for (int i = 0; i < 5; ++i) {
    pub->publish(msg);
    rclcpp::spin_some(node);
    std::this_thread::sleep_for(std::chrono::milliseconds(10));
  }
}

// ── Required test cases ───────────────────────────────────────────────────────
TEST_F(<ClassName>Test, SucceedsImmediatelyWhenConditionAlreadyMet)
{
  publishFake<MsgType>(node_, GOOD_VALUE);
  buildTree(/* xml with AlwaysSuccess child */);
  EXPECT_EQ(tickUntilDone(), BT::NodeStatus::SUCCESS);
  EXPECT_TRUE(blackboard_->get<bool>("output_port"));
}

TEST_F(<ClassName>Test, RetriesUntilConditionMet)
{
  publishFake<MsgType>(node_, BAD_VALUE);
  buildTree(/* xml */);

  auto status = tree_->tickRoot();
  EXPECT_EQ(status, BT::NodeStatus::RUNNING);

  publishFake<MsgType>(node_, GOOD_VALUE);
  EXPECT_EQ(tickUntilDone(), BT::NodeStatus::SUCCESS);
}

TEST_F(<ClassName>Test, FailsAfterMaxRetries)
{
  publishFake<MsgType>(node_, BAD_VALUE);   // never converges
  buildTree(/* xml with max_retries="2" */);
  EXPECT_EQ(tickUntilDone(200), BT::NodeStatus::FAILURE);
  EXPECT_FALSE(blackboard_->get<bool>("output_port"));
}

TEST_F(<ClassName>Test, FailsImmediatelyIfChildFails)
{
  buildTree(/* xml with AlwaysFail child */);
  EXPECT_EQ(tickUntilDone(), BT::NodeStatus::FAILURE);
}

TEST_F(<ClassName>Test, HaltResetsState)
{
  publishFake<MsgType>(node_, BAD_VALUE);
  buildTree(/* xml max_retries="2" */);
  tickUntilDone(200);   // run to failure

  tree_->haltTree();    // reset

  publishFake<MsgType>(node_, GOOD_VALUE);
  EXPECT_EQ(tickUntilDone(), BT::NodeStatus::SUCCESS);
  // Retry count should have reset to 0
  EXPECT_EQ(blackboard_->get<int>("retry_count"), 0);
}

TEST_F(<ClassName>Test, RetryCountTracked)
{
  publishFake<MsgType>(node_, BAD_VALUE);
  buildTree(/* xml max_retries="3" */);
  tickUntilDone(200);
  const int retries = blackboard_->get<int>("retry_count");
  EXPECT_GT(retries, 0);
  EXPECT_LE(retries, 3);
}
```

---

## Condition Node Tests

```cpp
TEST_F(<ClassName>Test, ReturnsSucessWhenConditionMet)
{
  // Set topic / blackboard to triggering value
  publishFake<MsgType>(node_, TRIGGERING_VALUE);
  buildTree(/* xml */);
  EXPECT_EQ(tickUntilDone(), BT::NodeStatus::SUCCESS);
}

TEST_F(<ClassName>Test, ReturnsFailureWhenConditionNotMet)
{
  publishFake<MsgType>(node_, NON_TRIGGERING_VALUE);
  buildTree(/* xml */);
  EXPECT_EQ(tickUntilDone(), BT::NodeStatus::FAILURE);
}

TEST_F(<ClassName>Test, ReturnsFailureWhenNoDataReceived)
{
  // Don't publish anything — no data case
  buildTree(/* xml */);
  EXPECT_EQ(tree_->tickRoot(), BT::NodeStatus::FAILURE);
}

TEST_F(<ClassName>Test, ThresholdPortRespected)
{
  publishFake<MsgType>(node_, BORDER_VALUE);
  buildTree(/* xml with threshold just above/below border value */);
  // Check both sides of the threshold
}
```

---

## CMakeLists.txt Test Section

```cmake
if(BUILD_TESTING)
  find_package(ament_cmake_gtest REQUIRED)
  find_package(ament_lint_auto   REQUIRED)
  ament_lint_auto_find_test_dependencies()

  ament_add_gtest(test_<snake_name>
    test/test_main.cpp
    test/test_<snake_name>.cpp
  )

  target_include_directories(test_<snake_name> PRIVATE include test)

  ament_target_dependencies(test_<snake_name>
    rclcpp
    behaviortree_cpp_v3
    nav2_behavior_tree
    <msg_or_action_or_service_package>
  )

  target_link_libraries(test_<snake_name> <package_name>)
endif()
```
