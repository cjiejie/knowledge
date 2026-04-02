# Global Planner (全局规划器)

在 Nav2 中，全局规划器运行在 **Planner Server** 节点中。它的主要职责是：在接收到目标点（Goal Pose）后，利用 **Global Costmap（全局代价地图）**，计算出一条从机器人当前位置到目标点的最优（或次优、可行）路径。

计算出的路径通常是一系列离散的点（`nav_msgs/msg/Path`），随后交由行为树传递给局部规划器（Local Planner）进行跟踪。

## 核心职责

1. **路径搜索**：基于图搜索算法或采样算法，在二维栅格上寻找连通路径。
2. **规避已知障碍**：参考全局代价地图，避开 `Lethal` 和 `Inscribed` 区域。
3. **快速响应**：由于全局路径可能很长，规划算法必须保证足够的效率（通常要求在几十到几百毫秒内返回）。

## 常见的全局规划器插件

Nav2 采用了高度的插件化设计，官方库 `nav2_smac_planner` 和 `nav2_navfn_planner` 提供了多种主流算法供开发者选择。

### 1. NavFn Planner
- **算法基础**：Dijkstra 或 A* 算法。
- **特点**：
  - ROS1 `navfn` 的直接移植版。
  - 纯 2D 搜索，假设机器人是一个质点（Point-mass）。
  - 不考虑机器人的运动学约束（如阿克曼转向），生成的路径可能会有直角转弯。
- **适用场景**：全向移动机器人（如差速底盘、麦轮底盘），或者需要极高计算速度的简单场景。

### 2. Smac Planner - 2D
- **算法基础**：优化的 A* 算法。
- **特点**：
  - Nav2 专门优化的 2D 规划器，引入了更平滑的启发式函数和代价值感知。
  - 相比 NavFn，它能生成更合理、距离障碍物更均匀的路径。
  - 支持多分辨率搜索。
- **适用场景**：差速/全向机器人的默认首选全局规划器。

### 3. Smac Planner - Hybrid-A* (混合 A*)
- **算法基础**：Hybrid-A*。
- **特点**：
  - 考虑了机器人的 **运动学约束**（Kinematic Constraints）和 **非完整约束**（Non-holonomic）。
  - 在搜索时不仅考虑 X, Y 坐标，还考虑了朝向（Yaw / Theta）。
  - 生成的路径是平滑的曲线，满足车辆的最大转弯半径（Minimum Turning Radius）约束。
  - 支持前向和后向规划（倒车入库）。
- **适用场景**：阿克曼转向结构机器人（如自动驾驶汽车、乘用车）、带有大拖车的机器人。

### 4. Smac Planner - State Lattice
- **算法基础**：状态格栅（State Lattice）与 A*。
- **特点**：
  - 预先生成一组满足运动学约束的运动基元（Motion Primitives）。
  - 在线搜索时将这些基元拼接起来形成路径。
  - 同样支持阿克曼等非完整约束底盘，但相比 Hybrid-A* 更适合高速或特定曲线（如样条曲线）的场景。
- **适用场景**：对运动学要求严格的阿克曼车辆或高速行驶机器人。

### 5. Theta* Planner
- **算法基础**：Theta* (Any-angle Path Planning)。
- **特点**：
  - 突破了传统栅格搜索“只能走 8 个离散方向”的限制。
  - 能够规划出任意角度的直线，路径更加短、直且自然，无需进行极端的平滑后处理。
- **适用场景**：在开阔场地或需要直来直去的飞行器、机器人。

## 如何选择和配置

在 `nav2_params.yaml` 中，可以通过 `planner_server` 节点下的配置来指定使用的插件：

```yaml
planner_server:
  ros__parameters:
    planner_plugins: ["GridBased"]
    GridBased:
      plugin: "nav2_smac_planner/SmacPlanner2D"
      tolerance: 0.5                      # 目标点容差
      downsample_costmap: false           # 是否降采样代价地图以加速
      allow_unknown: true                 # 是否允许穿过未知区域
```