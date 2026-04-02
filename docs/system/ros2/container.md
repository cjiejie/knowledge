# ROS2 容器化部署 (Docker)

在现代机器人系统工程中，使用 Docker 容器化 ROS2 环境已经成为了标准的最佳实践。机器人的开发、测试和实车部署往往面临复杂的依赖地狱（C++ 库版本、Python 环境、特定 Ubuntu 版本的绑定），容器化能够完美解决这些痛点。

## 为什么 ROS2 需要容器化？

1. **环境隔离与一致性**：“在我的电脑上能跑，在车上跑不起来”是机器人开发的常态。Docker 确保了从开发机到实车工控机（如 NVIDIA Jetson）的 OS 环境、系统库、ROS2 版本绝对一致。
2. **多版本共存**：同一台电脑上可以同时运行 ROS1 Noetic、ROS2 Foxy 和 ROS2 Humble，互不干扰。
3. **快速部署与 CI/CD**：通过拉取镜像，几分钟内即可在一台裸机上配置好复杂的自动驾驶或导航环境，非常适合车队批量部署和云端自动化测试。

## ROS2 容器化的核心原理与难点

容器本质上是利用 Linux 内核的 **Namespaces（命名空间）** 和 **Cgroups（控制组）** 实现的轻量级虚拟化。运行 ROS2 容器与运行普通的 Web 服务（如 Nginx）有极大的不同，因为机器人应用需要深度依赖网络、硬件和图形界面。

### 1. DDS 与网络通信 (Network)

ROS2 的底层通信基于 DDS（数据分发服务），DDS 严重依赖 UDP 多播（Multicast）进行节点发现。
- **默认桥接网络 (Bridge)**：Docker 默认使用隔离的虚拟网桥，这会导致容器内的 ROS2 节点无法发现宿主机或其他局域网设备上的节点。
- **解决方案 (`--network host`)**：在启动 ROS2 容器时，**必须**使用宿主机网络模式。这样容器共享宿主机的网络栈，DDS 的多播数据包能够畅通无阻，实现容器内外节点的无缝通信。

### 2. 硬件外设访问 (Devices)

机器人需要读取激光雷达、相机、串口等外设。
- **设备映射**：Linux 下的设备都映射在 `/dev` 目录下。启动容器时可以通过 `--device=/dev/ttyUSB0` 挂载单个串口，或者使用粗暴但高效的 `-v /dev:/dev --privileged` 赋予容器所有硬件的完全访问权限（在实车部署时常用）。
- **GPU 加速 (NVIDIA)**：如果要运行深度学习感知算法（如 YOLO）或 GPU 加速的仿真（如 Isaac Sim），需要安装 `Nvidia-Container-Toolkit`，并在启动时加上 `--gpus all` 参数，这会将宿主机的显卡驱动和 CUDA 环境透传进容器。

### 3. 图形化界面 (GUI & X11)

经常需要在容器内运行 `rviz2`、`rqt` 或 `gazebo` 进行可视化调试。
- 容器默认没有图形显示能力。需要在宿主机开放 X11 权限：`xhost +local:root`。
- 然后将宿主机的 X11 套接字挂载到容器中，并透传环境变量：`-v /tmp/.X11-unix:/tmp/.X11-unix -e DISPLAY=$DISPLAY`。

## 典型的 Dockerfile 示例

以下是一个用于构建基于 ROS2 Humble 环境的轻量化镜像脚本：

```dockerfile
# 使用官方 OSRF 提供的 ROS2 基础镜像
FROM osrf/ros:humble-desktop

# 设置环境变量，避免 apt 安装时弹出交互式对话框
ENV DEBIAN_FRONTEND=noninteractive

# 安装常用的调试工具和额外的 ROS2 包
RUN apt-get update && apt-get install -y \
    nano \
    tmux \
    htop \
    python3-pip \
    ros-humble-navigation2 \
    ros-humble-nav2-bringup \
    && rm -rf /var/lib/apt/lists/*

# 设置工作目录
WORKDIR /root/ros2_ws

# （可选）将宿主机的代码拷贝进去
# COPY ./src /root/ros2_ws/src

# 每次进入容器自动 source ROS2 环境
RUN echo "source /opt/ros/humble/setup.bash" >> /root/.bashrc

# 默认启动命令
CMD ["bash"]
```

## 典型的启动命令 (docker run)

在开发时，我们通常不把代码直接打包进镜像，而是通过**挂载数据卷 (Volumes)** 的方式映射代码，这样在宿主机用 VSCode/Cursor 修改代码，容器内直接编译：

```bash
docker run -it --rm \
  --name ros2_dev_env \
  --network host \
  --ipc host \
  --pid host \
  --privileged \
  -v /dev:/dev \
  -e DISPLAY=$DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v /home/unt/knowledge/src:/root/ros2_ws/src \
  my_ros2_image:latest
```

### 参数解析：
- `--network host / --ipc host / --pid host`：打破容器的网络和进程隔离，让 ROS2 的 DDS 通信（特别是基于共享内存的 FastDDS/CycloneDDS）达到最高性能。
- `--privileged / -v /dev:/dev`：解决所有雷达、相机、底盘串口的权限问题。
- `-e DISPLAY / -v /tmp...`：解决 Rviz2 弹窗显示问题。
- `-v .../src:/root...`：将本地代码挂载到容器的工作空间中进行热开发。