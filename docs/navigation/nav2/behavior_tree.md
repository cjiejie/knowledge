# Behavior Tree (行为树)

在 Navigation 2 (Nav2) 中，行为树（Behavior Tree，简称 BT）取代了 ROS1 `move_base` 中硬编码的有限状态机（FSM）。它充当了整个导航系统的“大脑”，负责编排全局规划、局部控制、异常恢复等各个模块的执行顺序。

Nav2 使用了开源库 **BehaviorTree.CPP** 来实现这一功能。

## 为什么使用行为树？

- **灵活性**：通过修改 XML 文件即可改变机器人的导航逻辑，无需重新编译代码。
- **模块化**：各种动作（如规划路径、跟随路径、旋转）都是独立的插件。
- **可读性强**：树状结构天然适合表达复杂的条件分支和恢复策略。

## 核心概念

行为树由不同类型的节点（Node）组成，每次执行（Tick）都会从根节点自顶向下、从左到右遍历。节点会返回三种状态：`SUCCESS`（成功）、`FAILURE`（失败）或 `RUNNING`（运行中）。

### 1. 控制节点 (Control Nodes)
控制节点决定了其子节点的执行逻辑：

- **Sequence (顺序节点)**：按顺序执行子节点。只要有一个子节点返回 `FAILURE`，它就返回 `FAILURE`；所有子节点都 `SUCCESS` 才返回 `SUCCESS`。（类似于逻辑 `AND`）
- **Fallback / Selector (选择节点)**：按顺序执行子节点。只要有一个子节点返回 `SUCCESS`，它就返回 `SUCCESS`；只有全部子节点都 `FAILURE`，它才返回 `FAILURE`。（类似于逻辑 `OR`）
- **PipelineSequence (流水线节点)**：Nav2 自定义节点。它会持续 ticking 所有的子节点，通常用于实现“一边走，一边重新规划路径”的逻辑。
- **RecoveryNode (恢复节点)**：Nav2 自定义节点。包含两个子节点，如果第一个节点（通常是导航动作）失败，则触发第二个节点（通常是恢复动作，如清除代价地图或原地旋转）。

### 2. 动作节点 (Action Nodes)
与具体的 ROS2 Action/Service 交互的节点：

- `ComputePathToPose`：请求 Planner Server 计算全局路径。
- `FollowPath`：请求 Controller Server 沿路径移动。
- `Spin`, `BackUp`, `Wait`：触发具体的恢复动作。
- `ClearEntireCostmap`：请求清除代价地图。

### 3. 条件节点 (Condition Nodes)
用于检查状态或条件的布尔节点（不执行动作）：
- `IsBatteryLow`：电量是否过低。
- `GoalUpdated`：目标点是否发生了改变。
- `DistanceTraveled`：是否移动了指定距离。

## 典型结构示例

Nav2 默认提供了一套名为 `navigate_to_pose_w_replanning_and_recovery.xml` 的行为树，它的核心逻辑简述如下：

1. **根节点**是一个 `RecoveryNode`。
2. 它的**主分支**是一个 `PipelineSequence`，里面包含两个任务：
   - 周期性地 `ComputePathToPose`（每隔一定时间或距离重规划）。
   - 持续地 `FollowPath`。
3. 如果主分支失败（比如撞到障碍卡死），则执行**恢复分支**。
4. 恢复分支是一个 `Sequence`，可能依次包含：
   - 清除局部代价地图 (`ClearEntireCostmap`)
   - 原地旋转 (`Spin`)
   - 再次尝试规划

## 定制自己的行为树

1. **编写 XML**：你可以用纯文本或可视化工具（如 Groot）创建一个符合业务逻辑的 `.xml` 文件。
2. **配置启动参数**：在 Nav2 的 `bt_navigator` 节点配置中，将 `default_bt_xml_filename` 参数指向你的 XML 文件。
3. **开发自定义插件**：如果现有的节点无法满足需求，你可以继承 BTCPP 的基类（如 `BtActionNode` 或 `ConditionNode`），编写 C++ 插件，并在 XML 中注册使用。