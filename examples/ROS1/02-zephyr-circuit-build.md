# ROS1 Zephyr 固定翼：从模型到任务闭环

## 1. 在 Iris 基线上增加了什么

实验目录为 `/home/xanter/ros1simulation/ros1sim_260913_zephyr_circuit`。它复用同一 ROS1 infra，但把 Iris/ArduCopter 替换为 Zephyr/ArduPlane，并新增任务控制器、人工解锁程序、相对航线、飞行结果判定和更严格的停止归档。

## 2. 模型转换

### 2.1 来源

模型源自本机 `ardupilot_gazebo` Git 对象提交 `8f3970a3d2bf0a5c4f283b969b39853e36716774`。来源是 Gazebo Sim 模型，不能原样加载到 Gazebo Classic 11。实验副本位于 `models/zephyr_with_ardupilot/`，来源和许可证记录在实验的 `MODEL_SOURCE.md`。

### 2.2 气动与飞控插件

`models/zephyr_with_ardupilot/model.sdf` 的模型级插件做了以下转换：

- 七个 `gz-sim-lift-drag-system` 改为七个名字唯一的 `libLiftDragPlugin.so`，保留各翼面原有的 `a0`、`cla`、`cda`、`cma`、控制关节和升阻方向；
- 删除 Gazebo Sim 的 JointStatePublisher 与 ApplyJointForce 系统插件；
- 飞控插件改为 Classic 的 `libArduPilotPlugin.so`；
- FDM 输入端口设为 9002，IMU 名称为 `imu_sensor`；
- channel 0/1 驱动左右 elevon 的 POSITION，channel 2 驱动推进器 VELOCITY。

这一步只换插件 API，气动参数仍来自固定来源模型。

### 2.3 地面接触修复

第一次动态运行中，细致机腹三角网格和向下延伸的桨叶碰撞体直接接触跑道，导致强烈抖动与 `PreArm: Accels inconsistent`。最终模型在 `model.sdf` 的 wing link 中使用：

~~~xml
<collision name="body_collision">
  <pose>0 -0.10 0 0 0 0</pose>
  <geometry><box><size>1.45 0.65 0.04</size></box></geometry>
</collision>
~~~

并加入 `skid_nose`、`skid_left`、`skid_right` 三个半径 0.04 m 的球形滑橇。滑橇摩擦系数为 0.18、slip 为 0.02、接触 `kp=100000`、`kd=100`。非必要桨叶碰撞体被移除，视觉和气动力仍保留。`worlds/zephyr_runway_classic.world` 把出生高度设为 0.155 m。

修复后静止角速度约 0.0004 rad/s，振动约 0.003 m/s²，IMU clipping 为 0。

## 3. world 与 ROS launch

`worlds/zephyr_runway_classic.world` 自包含地面、1400×45 m 跑道、中心线和 directional light，不请求在线模型。模型以航向 90° 放在跑道上。

`launch/zephyr_circuit.launch` 加载该 world，设置仿真时间，并让 MAVROS 监听 UDP 14550。容器侧 `scripts/container_launch.sh` 还会先查找 `libLiftDragPlugin.so`，再启动 roslaunch、rosbag和就绪探针。

## 4. 任务参数和几何

`config/plane_task.param` 是 ArduPlane 参数叠加层：

~~~text
LAND_DISARMDELAY 5
TKOFF_ALT 50
TKOFF_LVL_ALT 10
WP_RADIUS 35
WP_LOITER_RAD 60
~~~

`missions/circuit.json` 保存相对几何：起飞点前方 300 m/50 m，高度 60 m 的左矩形航线，进近点在 HOME 后方 500 m/45 m，LAND 目标在 HOME 前方 40 m。控制器以实时 HOME 和初始航向换算经纬度，因此任务不绑定某个绝对地图坐标。

`scripts/mission_controller.py` 构造七项任务：HOME、TAKEOFF、三个巡航 WP、进近 WP、LAND。上传后写 `MISSION_READY`，等待 heartbeat 中真实 armed 位，再切换 AUTO。它每 0.5 秒写 `metrics/telemetry.csv`；只有曾高于 20 m 且最终解除武装，才写：

~~~json
{"success": true, "reason": "landed_and_auto_disarmed"}
~~~

控制器代码禁止包含解锁命令。

## 5. 人工解锁边界

`scripts/arm.sh` 等待 `MISSION_READY`，要求操作者准确输入 `ARM`。随后 `scripts/manual_arm.py`：

1. 在 UDP 14552 等待 heartbeat；
2. 拒绝非固定翼 MAV 类型；
3. 发送标准 `MAV_CMD_COMPONENT_ARM_DISARM`，参数 `arm=1, force=0`；
4. 检查 COMMAND_ACK、STATUSTEXT 和 armed heartbeat。

任务控制器使用 14551，人工解锁使用 14552。分开端口与程序使“人工授权”和“自动执行任务”在代码层保持独立。

## 6. 静态检查为何这样设计

~~~bash
cd /home/xanter/ros1simulation/ros1sim_260913_zephyr_circuit
./scripts/check.sh
~~~

检查包含 Bash/Python/JSON/XML、七个 Classic 气动插件、三个控制 channel、简化 body collision、三个滑橇、无桨叶碰撞、launch/world 引用、Compose、SITL 二进制和资源文件。它还断言控制器没有解锁命令、人工解锁程序确实有解锁命令、用户脚本不含 Xterm。

用户入口只使用系统 `grep`，因为普通 shell 不保证安装 ripgrep。静态检查输出 `No simulation was started.`，表示检查本身不会启动飞行。
