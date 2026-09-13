# ROS2 Skywalker X8：运行与归档

## 1. 启动

~~~bash
cd /home/xanter/ros2simulation/ros2sim_260912_setuptest
./scripts/start.sh
~~~

脚本创建 `runs/run_时间戳`，并让 `.last_run` 指向它。manifest 记录 ArduPilot/ardupilot_gz 提交、dirty 状态、机型、任务和端口；任务 JSON 与参数文件被复制到 run，防止以后修改模板后无法解释旧结果。

随后打开四个 GNOME Terminal：

1. Gazebo Harmonic、Skywalker、ArduPlane、DDS Agent、bridge 和 RViz；
2. pymavlink 任务控制器；
3. ROS2 bag 记录器；
4. 人工解锁。

主终端工作目录是 `run/runtime`，因此 MAVProxy 和 SITL 文件落在 run 内。rosbag 先等待 `/ap/status`，再按 `config/record_topics.txt` 写 MCAP。

## 2. 解锁前验证四条链

等待主日志出现：

- 模型创建成功和 JSON received；
- ArduPilot Ready to FLY；
- XRCE session 建立与 `DDS: Initialization passed`；
- `/clock`、`/imu`、`/odometry` 和 `/ap/status` 有实际消息；
- 控制器写 `MISSION_READY`。

可在新终端执行：

~~~bash
source /opt/ros/jazzy/setup.bash
source /home/xanter/ros2_ws/install/setup.bash
ros2 topic echo --once /clock
ros2 topic echo --once /ap/status
ros2 topic echo --once /ap/pose/filtered
ros2 service list -t | grep '^/ap/'
~~~

也可只读运行文件：

~~~bash
run_dir=$(readlink -f .last_run)
tail -n 60 "$run_dir/launch.log"
tail -n 30 "$run_dir/controller.log"
~~~

## 3. 人工解锁和 AUTO 飞行

在 MANUAL ARM 终端输入：

~~~text
ARM
~~~

`arm.sh` 通过 ROS2 DDS 服务做 prearm 和 arm。控制器观察到 MAVLink armed 后才切换 AUTO。预期任务序号为 TAKEOFF 1、巡航/进近 2–5、LAND 6。成功要求相对高度曾超过 20 m，并在 LAND 后自动解除武装。

监控结果：

~~~bash
tail -f "$run_dir/controller.log"
tail -f "$run_dir/metrics/telemetry.csv"
~~~

`metrics/result.json` 的 `success=true` 与 `reason=landed_and_auto_disarmed` 表示功能闭环成功。落点精度是另一项指标，不能由这个布尔值代替。

## 4. 当前停止与归档流程

这个 ROS2 示例当前仍把停止和归档分开：

1. 在主 launch GNOME Terminal 按 Ctrl+C，等待 Gazebo、SITL、Agent、bridge 和 RViz 退出；
2. 在 ROS Bag GNOME Terminal 按 Ctrl+C，让 MCAP 写完 metadata；
3. 检查没有相关进程；
4. 在实验根执行 `./scripts/finalize.sh`。

~~~bash
ps -eo pid,comm,args | \
  grep -E 'gz sim|arduplane|micro_ros_agent|parameter_bridge|rviz2|ros2 bag record' | \
  grep -v grep
./scripts/finalize.sh
~~~

`finalize.sh` 在存在相关进程时拒绝归档；有 `MISSION_DONE` 时写 `DONE`，否则写 `FAILED`。它把 `mav.tlog*`、精确文件名 `mav.parm` 和实例目录 `runtime/0` 移入对应目录。

当前已知限制：`mav_0_1.parm` 不匹配 `mav.parm`，成功 run 中仍留在 `runtime/`；当前没有统一 `stop.sh`。这两点是对现状的准确记录，不影响已经闭合的 MCAP 或任务结果。

## 5. 已验证结果

成功 run 为 `runs/run_20260912T235453+0800`：

- `MISSION_DONE` 与 `DONE` 存在；
- `DDS: Initialization passed`；
- 任务结果为 `landed_and_auto_disarmed`；
- 任务用时 201.07 秒，最高相对高度 60.47 m，最终 mission seq 6；
- MCAP 为 495004823 字节，记录 590.8 秒、1920573 条消息；
- 最终 armed=0。

本次功能任务成功，但飞机最终相对规划接地点纵向提前约 195.8 m、横向误差约 0.4 m。后续研究着陆精度时，应把进近几何、空速、风与 ArduPlane LAND 参数作为实验变量，并用 ros2batch 做重复试验。

早期 FAILED run 揭示了接口和模型兼容问题，例如错误等待 `/ap/v1/*`、Skywalker 根级 pose、缺少 OdometryPublisher 与资源 URI。它们应作为回归测试来源，而不是从目录中删除。
