# 已知问题与排查

## Docker 命令无法连接

docker.service 默认未启动。通过 GNOME Terminal 执行 sudo systemctl start docker。当前用户不在 docker 组，这是预期权限边界。

## ros1_ws 无法写入

历史 build/devel 是容器 root 创建的，并固化 /root/ros1_ws。不要直接继续增量构建。新的容器以 UID/GID 1000 工作，实验使用私有 overlay；处理历史目录时先保留副本。

## Gazebo 版本混淆

裸 shell 中找不到 gz 不代表 Harmonic 被卸载；Harmonic 由 Jazzy vendor 路径提供。ROS1 使用 gazebo/gzserver，即 Gazebo Classic 11；ROS2 使用 gz sim，即 Gazebo Harmonic。

## legacy 插件缺失

确认 GAZEBO_PLUGIN_PATH 指向派生镜像中的插件 build 目录，并运行 ldd 检查 libArduPilotPlugin.so 是否有 not found。

## SITL 一直等待 Gazebo

核对：

- Iris world 是否加载 libArduPilotPlugin.so；
- UDP 9002/9003 是否被残留进程占用；
- sim_vehicle.py 是否使用 gazebo-iris frame；
- 容器是否使用 host network；
- 插件和当前 ArduPilot 是否仍协议兼容。

## MAVROS disconnected

确认 SITL/MAVProxy 正在向 UDP 14550 输出，MAVROS 使用 fcu_url:=udp://:14550@。先看 /mavros/state，再看端口占用和 MAVProxy output。

## Gazebo GUI 无法显示

确认 DISPLAY、XAUTHORITY、/tmp/.X11-unix 和 /dev/dri 映射。不要用 xhost + 放宽整个本地显示服务器访问。

## GUI 出现但没有联合仿真

Gazebo 可视化只是物理层。还必须确认 FDM 数据、MAVROS connected、/clock 与传感器消息、SITL 状态以及 rosbag 写入。


## world 停在 Getting models from

空的 GAZEBO_MODEL_DATABASE_URI 会被 gazebo_ros 的 gzserver 包装器恢复成在线地址。使用非空本地 file URI，并把 sun 等公共模型改成内联或本地资源。

## Iris 报缺少 gimbal_small_2d

legacy gazebo11 分支没有附带该模型。基础 Iris 联调使用实验本地派生模型，移除对应 include 和无效关节。

## 容器内 Intel DRI 失败

Focal Mesa 不识别本机 Raptor Lake iGPU。基础链使用 LIBGL_ALWAYS_SOFTWARE=1；高负载视觉任务另行验证 NVIDIA runtime。

## sim_vehicle.py 打开 Xterm

设置 SITL_RITW_TERMINAL，明确指定 GNOME Terminal。run_in_terminal_window.sh 的默认探测顺序会优先命中 Xterm。

## local_position 暂时没有消息

等待 EKF 初始化和 origin set。MAVROS connected 只证明 MAVLink 心跳建立，不能证明位置估计已就绪。


## Zephyr 地面抖动并报告 Accels inconsistent

不要强制解锁。检查机腹或桨叶是否使用复杂 mesh 直接参与跑道接触，并查看 VIBRATION 与 IMU clipping。当前实验使用简化 box 机身碰撞体、三点低摩擦球形滑橇，并移除非必要桨叶碰撞体；对应代码和修复前后证据见 [Zephyr 构建示例](../examples/ROS1/02-zephyr-circuit-build.md)。

## 用户 shell 报 rg: command not found

用户入口脚本不能假定存在 Codex 工具环境中的 ripgrep。启动前检查使用系统 `grep`；若新脚本确实需要 `rg`，应先把它声明为依赖并在普通交互 shell 验证。
