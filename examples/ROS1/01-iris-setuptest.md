# ROS1 Iris 最小联合验证

## 1. 认识目标

实验目录是 `/home/xanter/ros1simulation/ros1sim_260913_iris_setuptest`。它不解锁，也不飞行，只回答一个问题：容器内 Gazebo Classic/ROS1/MAVROS 能否与主机 ArduCopter SITL 稳定连接并形成可记录的数据流。

先做这个实验能把后续固定翼问题与公共基础设施问题分开。成功链路为：

~~~text
Gazebo Classic -> legacy ArduPilotPlugin <-> UDP 9002/9003 <-> ArduCopter SITL
ArduCopter/MAVProxy -> UDP 14550 -> MAVROS -> /mavros/*
Gazebo -> gazebo_ros -> /clock 与 /gazebo/model_states
~~~

## 2. 实验代码怎样构成

### 2.1 本地派生模型

上游 `iris_with_ardupilot/model.sdf` 引用不存在的 `gimbal_small_2d`。实验副本位于 `models/iris_with_ardupilot_local/model.sdf`，删除了这一段：

~~~xml
<include><uri>model://gimbal_small_2d</uri>...</include>
<joint name="iris_gimbal_mount" type="revolute">...</joint>
~~~

其余 Iris 动力学、旋翼气动与 `libArduPilotPlugin.so` 保持不变。这样修复的是缺失资源，不改变基础飞行动力学。

### 2.2 本地自包含 world

`worlds/iris_arducopter_runway_local.world` 把远程 `model://sun` 替换为内联 directional light，并把载具 URI 改为 `model://iris_with_ardupilot_local`。目的分别是消除在线模型依赖、选择刚才的派生模型。

### 2.3 ROS launch

`launch/iris_setuptest.launch`：

- 设置 `/use_sim_time=true`；
- 通过 `gazebo_ros/empty_world.launch` 加载本地 world；
- 通过 `mavros/apm.launch` 监听 `udp://:14550@`；
- system/component 目标为 1。

### 2.4 容器监测和记录

`scripts/container_launch.sh` 在容器内启动 roslaunch 和 rosbag，然后分阶段写标记：

- ROS master 可用：`ROS_MASTER_READY`；
- `/clock` 有消息：`GAZEBO_CLOCK_READY`；
- `/mavros/state` 中 `connected: True`：`MAVROS_CONNECTED`；
- model_states、IMU 与 local pose 都有实际消息：`DATA_TOPICS_READY`。

`config/record_topics.txt` 明确记录 `/clock`、model_states、MAVROS state、IMU、local pose 和全球位置。固定话题清单比全话题记录更容易控制容量。

### 2.5 主机启动

`scripts/start.sh` 为每次运行创建 `runs/run_时间戳/{gazebo,rosbag,sitl,metrics}`，复制版本事实到 `manifest.yaml`，再打开两个 GNOME Terminal：

1. 容器中的 Gazebo、ROS 和 MAVROS；
2. 主机 `sim_vehicle.py -v ArduCopter -f gazebo-iris`。

脚本显式设置 `SITL_RITW_TERMINAL` 为 GNOME Terminal，并把 `--use-dir` 指向本次 `run/sitl`，因此 MAVProxy 文件不会落到家目录。

## 3. 从基础环境开始复现

先确认 infra 已按[基础设施文档](00-noetic-gazebo11-infra.md)构建并通过 doctor，然后启动 Docker：

~~~bash
gnome-terminal --wait --title="ROS1 Iris：启动 Docker" -- bash -lc \
  'sudo systemctl start docker'

cd /home/xanter/ros1simulation/ros1sim_260913_iris_setuptest
./scripts/check.sh
./scripts/start.sh
~~~

`check.sh` 检查 Bash、launch XML、Compose 和 vendor 路径。启动后不要解锁。观察 Gazebo 中 Iris 静止，终端中应出现 `ArduPilot Ready`、MAVROS connected 和 ArduPilot controller online。

主机可直接看最新运行证据：

~~~bash
run_dir=$(cat .last_run)
find "$run_dir" -maxdepth 1 -type f -printf '%f\n' | sort
cat "$run_dir/metrics/mavros_state.yaml"
tail -n 40 "$run_dir/sitl/sim_vehicle.log"
~~~

## 4. 停止和判定

~~~bash
./scripts/stop.sh
~~~

`stop.sh` 在 GNOME Terminal 中停止容器和 SITL；容器 trap 先给 rosbag SIGINT，再停止 roslaunch。`finalize.sh` 只有在三类就绪标记齐全且相关进程已经退出时才写 `DONE`，否则写 `FAILED`。

随后将 Docker 恢复 inactive：

~~~bash
gnome-terminal --wait --title="ROS1 Iris：停止 Docker" -- bash -lc \
  'sudo systemctl stop docker.service docker.socket'
~~~

## 5. 已验证证据

成功 run 为 `runs/run_20260913T011015+0800`：

- `DONE`、`ROS_STOPPED` 与全部就绪标记存在；
- ArduCopter 日志出现 `ArduPilot Ready`；
- MAVROS 收到 heartbeat；
- rosbag v2 时长 154 秒，55.3 MB，310405 条消息；
- `stop.status=0`；
- 没有解锁或起飞。

早期 FAILED run 被保留，用于证明成功判据不是只看 GUI。复现时不要删除历史证据，也不要把旧 run 当作新的工作目录。
