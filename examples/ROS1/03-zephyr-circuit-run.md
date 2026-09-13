# ROS1 Zephyr 固定翼：运行、观察与归档

先完成 [infra](00-noetic-gazebo11-infra.md) 和 [Zephyr 构建说明](02-zephyr-circuit-build.md)中的检查。

## 1. 启动

~~~bash
cd /home/xanter/ros1simulation/ros1sim_260913_zephyr_circuit
./scripts/start.sh
~~~

`start.sh` 先运行静态检查、排除同名残留进程。若 Docker inactive，它会在 GNOME Terminal 中请求 sudo 并记录 `DOCKER_STARTED_BY_RUN`。然后创建：

~~~text
runs/run_时间戳/
├── manifest.yaml
├── mission.json
├── plane_task.param
├── gazebo/
├── sitl/
├── mavlink/
├── rosbag/
└── metrics/
~~~

接着打开四类可见终端：Gazebo/ROS/MAVROS 容器、ArduPlane/MAVProxy、任务控制器、人工解锁。SITL 命令使用 `-v ArduPlane -f gazebo-zephyr --use-dir=本次run/sitl`，并把 MAVLink 转发到 14551 和 14552。

## 2. 解锁前观察

不要立即输入 `ARM`。先确认：

~~~bash
run_dir=$(cat .last_run)
find "$run_dir" -maxdepth 1 -type f -printf '%f\n' | sort
tail -n 40 "$run_dir/controller.log"
tail -n 50 "$run_dir/sitl/sim_vehicle.log"
~~~

应看到 `ROS_MASTER_READY`、`GAZEBO_CLOCK_READY`、`MAVROS_CONNECTED`、`DATA_TOPICS_READY` 和 `MISSION_READY`。Gazebo 中飞机应静止，控制器应停在 `WAITING FOR MANUAL ARM`。如果出现 `Accels inconsistent`、clipping 快速增加或飞机在地面自转，应停止，不要强制解锁。

## 3. 人工解锁与自动任务

在“ROS1 Zephyr：人工解锁”终端输入：

~~~text
ARM
~~~

这是本次运行唯一的解锁入口。成功后人工程序显示 `ARMED confirmed`，控制器观察 armed 后切换 AUTO。预期飞行顺序为：

1. `Mission: 1 Takeoff`，约 49 m 完成起飞；
2. `Mission: 2..5 WP`，沿左矩形绕场；
3. `Mission: 6 Land`，进入下滑进近；
4. 接地并在 `LAND_DISARMDELAY=5` 后自动解除武装；
5. 控制器写 `MISSION_DONE` 和 `metrics/result.json`。

主机可监控：

~~~bash
tail -f "$run_dir/controller.log"
# 另一个终端
tail -f "$run_dir/metrics/telemetry.csv"
~~~

`telemetry.csv` 的列依次为 elapsed、mission_seq、armed、经纬度、相对高度、地速和航向。

## 4. 停止与自动归档

任务完成后仿真保持开启供观察。执行：

~~~bash
./scripts/stop.sh
~~~

它打开 GNOME Terminal，按以下顺序收尾：停止容器；给人工解锁、控制器和 SITL 的进程组发 SIGINT；等待 rosbag 和 Gazebo；必要时发 TERM；运行 `finalize.sh`。只有 `MISSION_DONE` 和四个基础就绪条件全部满足才写 `DONE`，否则写 `FAILED`。

核对：

~~~bash
cat "$run_dir/stop.status"
cat "$run_dir/metrics/result.json"
cat "$run_dir/metrics/rosbag_info.txt"
find "$run_dir" -maxdepth 1 -type f -printf '%f\n' | sort
~~~

若本次 run 启动了 Docker 且没有其他容器，stop worker 会恢复 Docker inactive。若 Docker 在启动实验前已经 active，实验不会擅自关闭它，操作者应在确认无其他用途后停止 service/socket。

## 5. 成功 run 与故障复盘

成功证据为 `runs/run_20260913T015818+0800`：

- 人工解锁后进入 AUTO；
- 起飞、航点 2–5、LAND 全部执行；
- 控制器结果 `landed_and_auto_disarmed`；
- 任务用时 290.19 秒，最高相对高度 60.35 m；
- rosbag v2 时长 389 秒，183.6 MB，787461 条消息；
- `DONE`、`ROS_STOPPED` 存在，`stop.status=0`。

前一 run `run_20260913T015218+0800` 因地面碰撞抖动失败。保留 FAILED run 的意义是让模型修复有可比较证据：修复前 vibration z 约 58 m/s²且 clipping 数千，修复后静止 clipping 为 0 并完成全任务。
