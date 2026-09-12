# 更新与维护

## 更新快照

升级后记录系统、ROS/Gazebo/Python/编译器版本，仓库分支与提交，包来自工作区还是 /opt，DDS 生成器标签，启动参数、端口、实际通过项和遗留告警。

~~~bash
source /opt/ros/jazzy/setup.bash
source /home/xanter/ros2_ws/install/setup.bash
uname -a
lsb_release -ds
printenv ROS_DISTRO
gz sim --versions
python3 --version
ros2 pkg prefix ardupilot_gz_bringup
ros2 pkg prefix ros_gz_bridge
git -C /home/xanter/ros2_ws/src/ardupilot status --short --branch
git -C /home/xanter/Projects/ardupilot_gz status --short --branch
~~~

## 保护本机补丁

pull、rebase、checkout 或 reset ardupilot_gz 前保存 robot.launch.py 的 URI 修复：

~~~bash
mkdir -p /home/xanter/Documents/env_notes/ROS2/patches
git -C /home/xanter/Projects/ardupilot_gz diff --   ardupilot_gz_bringup/launch/robots/robot.launch.py   > /home/xanter/Documents/env_notes/ROS2/patches/robot-package-uri.patch
~~~

当前未自动生成补丁副本，避免它随源码演进而过期。

## 清理边界

确认无需保留后可清理：/home/xanter/sim_runs 历史运行、ros2_ws/log、失败包的 build 子目录、mav.tlog、mav.parm、terrain、eeprom.bin、BIN 日志和可重建缓存。

应保留：ros2_ws/src、ros2_ws/install（除非完整重建）、Projects/ardupilot_gz 及本机修改、venv-ardupilot、~/.local/bin/microxrceddsgen 及对应 ~/.local/share、本目录。

## 最小回归

1. source 两层环境，核对关键包和 Gazebo。
2. 构建修改过的包。
3. 启动 Iris，检查实体、JSON、DDS、/clock、/ap/status。
4. 停止并确认无残留。
5. 启动 Alti，重复链路检查。
6. 仿真内执行 prearm、GUIDED、arm、20 m takeoff。
7. 停止，归档 launch.log，更新本文档。

## 系统目录原则

当前链路使用 /opt/ros/jazzy 系统包和家目录工作区。不要手工将自编译程序复制到 /usr/bin。自建工具放在工作区 install、虚拟环境或 ~/.local/bin，通过 source 和 PATH 明确选择来源。
