# 工具与配置
## GIT相关

```
# git 合并最近几次提交：
git reset --soft HEAD~3
git commit -m "feat: xxx"

# 推送到远程（如果需要强制推送）
git push --force-with-lease

# 将单个commit与它的父commit比较生成
patchgit diff <commit-hash>^! > mypatch.patch
```

## Foxglove工具
```
ros2 launch foxglove_bridge foxglove_bridge_launch.xml port:=8765
ros2 launch rosbridge_server rosbridge_websocket_launch.xml

# plotjuggler工具
ros2 run plotjuggler plotjuggler

# 网页控制端：
ros2 launch ros_web_controller ros_web_controller.launch.py
```

## Terminal自定义布局

配置terminal，在布局，自定义命令中输入类似如下
```
bash -i -c "sourcen; #sleep 5; # ros2 launch nav2_bringup rviz_launch.py;echo 'ros2 launch nav2_bringup rviz_launch.py'; bash "
```
## kst2.py脚本
```
python3 kst2.py ~/local.txt --ylabel "Time: ms" --xlabel "Count" -P "Local_costmap" -T "updateBoundstime" -y 3
```
## rqt_plot 可视化
```
ros2 run rqt_plot rqt_plot
```
## OpenVINNO安装
```
# 官方教程地址
https://www.intel.com/content/www/us/en/developer/tools/openvino-toolkit/download.html?PACKAGE=OPENVINO_BASE&VERSION=v_2024_6_0&OP_SYSTEM=LINUX&DISTRIBUTION=APT

# 安装wget
sudo apt install wget
# 下载并添加GPG密钥
wget https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB
sudo apt-key add GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB

# 添加APT仓库 Ubuntu 22.04 执行：
# Ubuntu 22
echo "deb https://apt.repos.intel.com/openvino ubuntu22 main" | sudo tee /etc/apt/sources.list.d/intel-openvino.list
sudo apt update
# 搜索可用的OpenVINO包，确认仓库配置成功
apt-cache search openvino
# 安装OpenVINO 2025.6开发包
sudo apt install openvino-2025.4.1
```
## ROS2查看action使用service和topic
```
ros2 service list --include-hidden-services | grep _action
ros2 topic list --include-hidden-topics  | grep _action
```
## ROS2 bag 播包
```
ros2 bag play <bag_file_path> --topics /topic1 /topic2 --clock 40
ros2 bag play 0311bag/obstacle_perception_bag_0.db3 --topics /camera_chin/depth/image_raw/compressed /camera_chin/depth/camera_info --clock 50
```
## bash快捷键，可参考
```
alias ..='cd ..'
alias ...='cd ../..'
alias cw='cd /home/unt/code/nav2_ws'
alias cs='cd /home/unt/code/nav2_ws/src'
alias ip='ifconfig |awk -F"[ ]+|[:]" "NR==2 {print $4}"'
alias sourcen='source /home/unt/code/nav2_ws/install/setup.bash'
alias sourceu='source /home/unt/code/unt_humanoid/install/setup.bash; source /home/unt/code/unt_humanoid/scripts/local_pc/env.sh'
alias rn='sourcen; clear; ros2 launch nav2_bringup navigation_launch.py'

alias orinx='sshpass -p "1" ssh agi@10.193.253.199'

cb() { cw; if [ $# -eq 0 ]; then colcon build --symlink-install ; else colcon build --symlink-install --packages-select "$1" --allow-overriding "$1" ; fi; }

cbc() { cw; if [ $# -eq 0 ]; then colcon build --symlink-install --cmake-clean-cache ; else colcon build --symlink-install --packages-select "$1" --cmake-clean-cache; fi; }

# 从ip为192.168.0.202的电脑快速复制文件
lcp() { scp -r user@192.168.0.202:"$1" ./;}
```
# 代码与部署
## 导航代码部署
```
# 1. 获取导航源代码
git clone -b v0.1.0-rc1 git@jihulab.com:unt-robotics/navigation/nav2.git
# 3. 上传代码到orin板，在orin板上新建nav2_ws，并初始化空间
sudo rosdep init && rosdep update
# 3. 用类似filezilla，将代码推送到orin板nav2_ws目录，将nav2改名为src
# 4. 安装依赖，自动安装依赖，孔板时这里会自动安装四个依赖包，libg2o,行为树,libceres-dev,libmagick++-dev
rosdep install -y -r -q --from-paths src --ignore-src --rosdistro humble
#5. 全部编译
colcon build --symlink-install
```

## 代码编译
```
# 指定编译某个包
colcon build --symlink-install --packages-select nav2_bringup

# 编译某个包及依赖：
colcon build --symlink-install --packages-up-to nav2_bringup

# colcon会自动检测编译有改动的包
colcon build --symlink-install --packages-ignore-regex=.* --packages-select $(colcon list --names-only --packages-select-if-changed)

# 清除缓存编译
colcon build --symlink-install --packages-select nav2_bringup --cmake-clean-cache 

# 指定编译线程数
colcon build --symlink-install --packages-select nav2_bringup --parallel-workers 10

# bash 快捷键

cb() { cw; if [ $# -eq 0 ]; then colcon build --symlink-install ; else colcon build --symlink-install --packages-select "$1" --allow-overriding "$1" ; fi; }

cbc() { cw; if [ $# -eq 0 ]; then colcon build --symlink-install --cmake-clean-cache ; else colcon build --symlink-install --packages-select "$1" --cmake-clean-cache; fi; }
```

## 代码编译cmake选项  --cmake-args
```
# 添加cmake选项：--cmake-args
colcon build --symlink-install --cmake-args -DCMAKE_BUILD_TYPE=Release

# 添加compile_commands.json输出
colcon build --symlink-install --cmake-args -DCMAKE_EXPORT_COMPILE_COMMANDS=ON 

#添加调试输出--event-handlers console_direct+
colcon build --symlink-install --cmake-args --event-handlers console_direct+

colcon build --merge-install 
                --parallel-workers 12\
                --cmake-args -DCMAKE_BUILD_TYPE=Release \
                -DCMAKE_CXX_COMPILER_LAUNCHER=ccache \
                -DCMAKE_C_COMPILER_LAUNCHER=ccache \
                -DCMAKE_VERBOSE_MAKEFILE=ON \
                -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
                -DCMAKE_CXX_STANDARD=17 \
                -DCMAKE_CXX_STANDARD_REQUIRED=ON
```                
## 启动navigation2
```
# 生命周期激活：
ros2 lifecycle set /map_server configure
```
全局规划类型
```
Declared types are  nav2_navfn_planner/NavfnPlanner nav2_smac_planner/SmacPlanner2D nav2_smac_planner/SmacPlannerHybrid nav2_smac_planner/SmacPlannerLattice nav2_theta_star_planner/ThetaStarPlanner
```
全局规划类型参数
```
valid options are MOORE, VON_NEUMANN, DUBIN, REEDS_SHEPP, STATE_LATTICE.
```
添加tf发布
```
ros2 run tf2_ros static_transform_publisher 0 0 0 0 0 0 map odom
```
动态切换模型：
```
ros2 param set /controller_server TEB.footprint_model.type "polygon"
ros2 param set /local_costmap/local_costmap footprint_model.type "polygon"
ros2 param set /global_costmap/global_costmap footprint_model.type "polygon"

ros2 topic pub --once /planner_selector std_msgs/msg/String "{data: 'Astar/HybridAstar'}"
ros2 param set /controller_server TEB.min_turning_radius 1.5
```

查看
```
ros2 param get /controller_server TEB.min_turning_radius
```


启动导航
```
ros2 launch nav2_bringup navigation_launch.py

# bash 快捷键
alias rn='sourcen; clear; ros2 launch nav2_bringup navigation_launch.py'
```
## 发布一个静态TF
```
# ros2 run tf2_ros static_transform_publisher x y z roll pitch yaw frame_id child_frame_id
# 它描述的是子坐标系在父坐标系下的位置
ros2 run tf2_ros static_transform_publisher -1.5 0.0 0.0 0 0 0 map map_ground
ros2 run tf2_ros static_transform_publisher 0.0 0.0 0.0 0 0 0 map base_link
ros2 run tf2_ros static_transform_publisher 0.1 0.0 1.06 0 -1.16 0 base_link camera_abdomen_link
```
# 相机-奥比中光335
git地址：https://gitee.com/orbbecdeveloper/OrbbecSDK_ROS2  
启动相机  
```
ros2 launch orbbec_camera gemini_330_series.launch.py  
```
OrbbecSDK V2 ROS2 封装文档  
https://orbbec.github.io/OrbbecSDK_ROS2/zh/index.html

# Woosh 底盘
```
# 启动官方ROS程序  
ros2 run woosh_robot_agent agent --ros-args -r __ns:=/woosh_robot -p ip:="169.254.128.2"
# 取消任务：
ros2 topic pub -1 /goto_cancel std_msgs/msg/Bool "{data: True}"
# 下发pose：
ros2 topic pub -1 /goto_pose woosh_common_msgs/msg/Pose2D "{x: 4.39, y: 1.08, theta: 0.0}"
# 下发预设id：
ros2 topic pub -1 /goto_id std_msgs/msg/String "{data: '2'}"
```
# 网络调试
```
# 查看当前路由情况：
sudo iptables -L -t nat
#清空当前规则：
sudo iptables -t nat -F
```
# 飞书
```
curl -X POST -H "Content-Type: application/json" \-d '{    "msg_type": "text",    "content": {        "text": "有趣，你们在干什么？"    }}' https://open.feishu.cn/open-apis/bot/v2/hook/xxx


curl -X POST -H "Content-Type: application/json" \-d '{    "msg_type": "text",    "content": {        "text": "[看]"    }}' https://open.feishu.cn/open-apis/bot/v2/hook/xxx

```