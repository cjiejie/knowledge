# Nav2 架构概述

Navigation 2 (Nav2) 是 ROS2 环境下的一套完整的移动机器人导航框架。它的核心目标是寻找安全且高效的路径，使机器人能够从起点移动到指定目标点。与 ROS1 中的 `move_base` 不同，Nav2 采用了高度模块化和分布式的架构设计，极大地提升了灵活性和可扩展性。

## 核心架构设计

Nav2 在设计上全面拥抱了 ROS2 的特性（如 Lifecycle Nodes、Action Servers 等），其架构主要由以下几个核心部分组成：

### 1. 行为树 (Behavior Tree / BT Navigator)

这是 Nav2 的“大脑”。Nav2 废弃了 ROS1 中的简单状态机，引入了行为树（通过 `BehaviorTree.CPP` 实现）来管理和编排整个导航任务。
- **bt_navigator**: 接收来自用户的导航目标（Action 请求），并根据配置的 XML 行为树文件调用相应的插件。
- 它可以非常方便地实现复杂的导航逻辑，比如：“先尝试生成路径，如果失败则执行原地旋转恢复，然后再重试；如果依然失败则报错”。

### 2. 核心导航服务器 (Action Servers)

这些服务器作为独立节点运行，通过 ROS2 Action 接口提供服务。主要包含：

- **Planner Server (全局规划器)**
  - 负责在全局代价地图上计算从当前位置到目标点的最优（或次优）路径。
  - 常见插件：`NavFn`, `SmacPlanner`, `ThetaStar` 等。
- **Controller Server (局部规划器/控制器)**
  - 负责跟踪 Planner 生成的全局路径，结合局部代价地图和当前传感器数据，输出底盘的速度指令 (`cmd_vel`)。
  - 常见插件：`DWB Controller`, `MPPI Controller`, `TEB Local Planner` 等。
- **Smoother Server (路径平滑器)**
  - 可选组件，用于对 Planner 生成的粗糙全局路径进行优化和平滑处理，使机器人的运动更加平稳。
- **Behavior Server (恢复与行为控制)**
  - 负责处理导航过程中的异常情况和特定的独立行为。
  - 例如：原地旋转清除代价地图 (`Spin`)、后退 (`BackUp`)、停车等待 (`Wait`) 等。

### 3. 代价地图 (Costmap2D)

用于将传感器数据（如雷达、深度相机）转换为机器人可以理解的障碍物信息。分为两个层级：
- **Global Costmap (全局代价地图)**：主要基于静态地图，用于全局路径规划。
- **Local Costmap (局部代价地图)**：基于实时传感器数据，通常以机器人为中心，用于局部避障和控制。
- 代价地图由多层组成（Plugins），如：静态层 (Static Layer)、障碍物层 (Obstacle Layer)、膨胀层 (Inflation Layer) 等。

### 4. 环境表示与定位

- **Map Server**: 负责加载和发布静态地图（栅格地图）。
- **AMCL (自适应蒙特卡洛定位)**: 负责在已知地图中估算机器人的位姿 (TF: `map` -> `odom`)。

### 5. 生命周期管理 (Lifecycle Manager)

Nav2 中的几乎所有核心节点都是基于 ROS2 的 **Lifecycle Node**（生命周期节点）实现的。
- `lifecycle_manager` 负责按照严格的确定性顺序启动、配置、激活或关闭这些节点。
- 这确保了在导航系统完全就绪之前，不会有组件提前运行并导致系统崩溃。

## 导航数据流向

1. 用户向 `bt_navigator` 发送 `NavigateToPose` 动作请求。
2. `bt_navigator` 解析行为树，通常首先请求 **Planner Server** 计算全局路径。
3. Planner 结合 **Global Costmap** 返回一条路径。
4. `bt_navigator` 将该路径发送给 **Controller Server**。
5. Controller 结合 **Local Costmap** 和里程计信息，计算出实时的运动控制指令并发布到 `/cmd_vel`。
6. 如果 Controller 遇到障碍物卡住，`bt_navigator` 会触发 **Behavior Server** 执行恢复动作。
7. 如果恢复成功，重新回到步骤 2 或 4；如果彻底失败，则返回失败状态给用户。

## 总结

Nav2 通过**行为树**将原本耦合的规划、控制、恢复等模块彻底解耦，并通过 **Lifecycle Node** 保证了系统的稳定性。开发者可以通过编写自定义插件或修改行为树 XML 文件，轻松适配不同底盘和不同应用场景的移动机器人。
