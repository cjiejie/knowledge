# 强化学习 (Reinforcement Learning) 入门指南

强化学习（RL）是机器学习的三大分支之一（监督学习、无监督学习、强化学习）。在机器人领域，它被广泛应用于机械臂抓取、四足机器人行走（如波士顿动力机器狗）、自动驾驶轨迹规划等复杂决策任务。

本指南提供了一条从零基础到能够阅读并复现前沿 RL 论文的结构化学习路径。

---

## 阶段一：核心概念与理论基础

不要急于上手深度强化学习（Deep RL），先理解强化学习的本质框架：**马尔可夫决策过程（MDP）**。

### 必须掌握的知识点
1. **五大基本元素**
   - **Agent（智能体）**：做决定的主体（如机器人）。
   - **Environment（环境）**：智能体所处的外部世界（如迷宫、物理引擎）。
   - **State (S)**：环境当前的状态。
   - **Action (A)**：智能体可以采取的动作。
   - **Reward (R)**：环境对动作的即时反馈（奖励或惩罚）。
2. **马尔可夫决策过程 (MDP)**
   - 理解状态转移概率 $P(s' | s, a)$：当前状态只与上一状态和动作有关。
3. **核心函数**
   - **Policy (策略 $\pi$)**：在状态 $s$ 下选择动作 $a$ 的概率分布 $\pi(a|s)$。
   - **Value Function (状态价值函数 $V(s)$)**：在状态 $s$ 下，未来能获得的累积期望奖励。
   - **Q-Function (动作价值函数 $Q(s,a)$)**：在状态 $s$ 下，采取动作 $a$ 后，未来能获得的累积期望奖励。
   - **Bellman Equation (贝尔曼方程)**：推导当前价值与未来价值之间关系的基石。

### 推荐学习资料
- **书籍**：Sutton & Barto 的 *《Reinforcement Learning: An Introduction》*（RL 领域的圣经，重点看前 6 章：动态规划、蒙特卡洛、时间差分）。
- **视频**：David Silver (DeepMind) 的 [UCL 强化学习课程](https://www.youtube.com/playlist?list=PLqYmG7hTraZDM-OYHWgPebj2MfCFzFObQ)。

---

## 阶段二：经典表格型 RL 与 Value-Based 方法

在这个阶段，状态和动作空间是有限且离散的（可以用表格存储）。

### 必须掌握的知识点
1. **动态规划 (Dynamic Programming)**
   - Policy Evaluation（策略评估）
   - Policy Iteration & Value Iteration（策略迭代与价值迭代）
2. **无模型预测与控制 (Model-Free)**
   - 蒙特卡洛方法 (Monte Carlo, MC)
   - 时间差分法 (Temporal Difference, TD)
3. **两大核心经典算法**
   - **Q-Learning** (Off-policy, 异策略)：学习全局最优策略，不依赖当前策略。
   - **SARSA** (On-policy, 同策略)：边走边学，保守且安全。

### 实践任务
- 使用 Python 和 NumPy，手写一个 Q-Learning 算法解决 OpenAI Gym 中的 `FrozenLake`（冰湖迷宫）或 `CliffWalking`（悬崖寻路）问题。

---

## 阶段三：深度强化学习 (Deep RL) —— 价值函数逼近

当状态空间变得无限或连续（如机器人的关节角度、摄像头的图像），无法用表格存储时，引入**神经网络**来逼近价值函数或策略。

### 必须掌握的知识点
1. **DQN (Deep Q-Network)**
   - **算法核心**：用神经网络替代 Q 表。
   - **两大关键技术**（解决神经网络训练不稳定的问题）：
     1. **Experience Replay (经验回放池)**：打乱数据相关性。
     2. **Target Network (目标网络)**：固定目标，缓解自举（Bootstrapping）带来的震荡。
2. **DQN 的改进变体**
   - Double DQN (DDQN)：解决 Q 值过估计问题。
   - Dueling DQN：将网络拆分为状态价值（Value）和动作优势（Advantage）。
   - Prioritized Experience Replay (PER)：优先回放高误差（TD-Error）的样本。

### 实践任务
- 使用 PyTorch/TensorFlow 实现 DQN。
- 解决 OpenAI Gym 中的 `CartPole`（倒立摆）或 Atari 游戏（如 `Pong`）。

---

## 阶段四：深度强化学习 —— 策略梯度 (Policy Gradient)

价值逼近法（如 DQN）很难处理连续动作空间（如机械臂的力矩控制），此时需要直接对策略 $\pi(a|s)$ 进行参数化和优化。

### 必须掌握的知识点
1. **Policy Gradient 定理**
   - REINFORCE 算法核心思想：通过采样轨迹（Trajectory），增加高回报动作的概率，降低低回报动作的概率。
2. **Actor-Critic (演员-评论家) 架构**
   - **Actor（演员）**：负责输出动作（策略网络 $\pi$）。
   - **Critic（评论家）**：负责评价动作的好坏（价值网络 $V$ 或 $Q$）。
   - 结合了 Policy Gradient 和 TD 学习的优势。
3. **前沿核心算法 (PPO & SAC)**
   - **A2C / A3C** (Advantage Actor-Critic)。
   - **DDPG** (Deep Deterministic Policy Gradient)：处理连续动作空间的“DQN”。
   - **PPO** (Proximal Policy Optimization)：OpenAI 的默认基线算法，通过限制策略更新步长（Clip）保证训练极度稳定。
   - **SAC** (Soft Actor-Critic)：引入最大熵强化学习，鼓励智能体探索，在机器人连续控制中表现极其优异。

### 实践任务
- 重点！必须亲手实现一遍 **PPO** 或 **SAC** 算法。
- 解决连续控制环境，如 `Pendulum-v1` 或 `BipedalWalker`。

---

## 阶段五：机器人与高级话题 (Robot Learning)

进入真实世界或高保真物理仿真环境。

### 必须掌握的知识点
1. **Sim-to-Real (仿真到现实的迁移)**
   - Domain Randomization (域随机化)：在仿真中随机化质量、摩擦力、光照等，让策略在真实世界中也能泛化。
2. **物理引擎与环境构建**
   - 熟悉常用的机器人 RL 仿真器：Isaac Gym (NVIDIA), MuJoCo, PyBullet。
   - 学会用 URDF/MJCF 导入机器人模型，并自己编写 Reward 函数。
3. **Offline RL (离线强化学习)**
   - 如 CQL, AWAC。利用过去收集的数据集直接训练策略，无需在真实环境中试错（真实机器人试错成本极高）。
4. **Imitation Learning (模仿学习)**
   - Behavior Cloning (行为克隆)：把专家（人类）的数据当成监督学习的标签。
   - Inverse RL (逆强化学习)：从专家数据中反推 Reward 函数。

### 推荐学习资料
- **课程**：UC Berkeley CS285 (Deep Reinforcement Learning)。
- **实战框架**：熟悉 [Stable-Baselines3](https://github.com/DLR-RM/stable-baselines3) 或 [CleanRL](https://github.com/vwxyzjn/cleanrl)（强烈推荐 CleanRL，单文件实现，极其适合学习源码）。

---

## 总结与建议

1. **重推导，更重代码**：RL 存在严重的“复现难”问题。哪怕数学公式看懂了，代码里的一行 `done` 标志位处理错误，都会导致整个网络无法收敛。
2. **先调包，再手写**：建议先用 Stable-Baselines3 跑通环境，看看正确的 Reward 曲线长什么样，再自己手写算法去对齐基线。
3. **Reward 设计是核心（Reward Shaping）**：在工程实践中，60% 的时间是在调环境和写 Reward 函数，剩下 40% 才是调算法超参数。