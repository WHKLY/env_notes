# 构建与运行手册

## 基础设施位置

~~~text
/home/xanter/ros1simulation/infra/noetic_gazebo11
~~~

## 构建原则

- 基于已经存在的固定摘要镜像构建。
- legacy ArduPilotPlugin 源码随基础设施保存并记录提交号。
- 构建产物留在派生镜像中，不安装到主机 /usr。
- Docker 与 Gazebo 操作通过 GNOME Terminal 显示。

## 验证层级

1. Docker image 能构建。
2. doctor 能确认 Noetic、Gazebo、gazebo_ros、MAVROS 和 libArduPilotPlugin.so。
3. Gazebo Classic 能显示空世界，/clock 有消息。
4. Iris world 能加载 legacy 插件。
5. 主机 SITL 与插件通过 UDP 9002/9003 通信。
6. MAVROS 的 /mavros/state 显示 connected: true。
7. 解锁与起飞测试通过，rosbag 能正常收尾。

## 运行目录

所有实验使用：

~~~text
/home/xanter/ros1simulation/ros1sim_YYMMDD_slug/runs/run_YYYYMMDDTHHMMSS+ZZZZ
~~~

SITL/MAVProxy 的当前目录、Gazebo 日志和 ROS bag 都指向本次 run，防止 mav.tlog 等文件落到家目录。

## 停止顺序

1. 停止 rosbag 并等待 bag 索引写完。
2. 对 roslaunch/Gazebo 发送 Ctrl+C。
3. 对主机 SITL/MAVProxy 发送 Ctrl+C。
4. 检查 roscore、gzserver、gzclient、MAVROS 和 ArduPilot 无残留。
5. 整理运行文件并写 DONE 或 FAILED。


## 已实现命令

~~~bash
cd /home/xanter/ros1simulation/infra/noetic_gazebo11
./scripts/build-visible.sh
./scripts/doctor-visible.sh
./scripts/shell-visible.sh

cd /home/xanter/ros1simulation/ros1sim_260913_iris_setuptest
./scripts/check.sh
./scripts/start.sh
./scripts/stop.sh
~~~

start.sh 打开 GNOME Terminal 并创建独立 run。stop.sh 在可见终端请求 sudo，停止容器与 SITL、收尾 rosbag、检查残留并自动执行 finalize。

## Zephyr 普通固定翼任务

~~~bash
cd /home/xanter/ros1simulation/ros1sim_260913_zephyr_circuit
./scripts/check.sh
./scripts/start.sh
~~~

任务控制器上传航线后停在人工解锁门。只有人工解锁终端输入 `ARM` 后才发送解锁命令；降落并自动解除武装后运行 `./scripts/stop.sh` 完成 rosbag 收尾和归档。

2026-09-13 的 `run_20260913T015818+0800` 已完整验证该流程。首次从基础环境复现时按 [examples/ROS1](../examples/ROS1/README.md) 的 infra、Iris、Zephyr 顺序操作。
