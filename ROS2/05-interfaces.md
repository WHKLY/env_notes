# 接口、话题与端口

## 端口

n 为实例号。

| 用途 | 规则 | 实例 0 |
|---|---:|---:|
| Gazebo JSON FDM | 9002 + 10n | UDP 9002 |
| Micro XRCE-DDS | 2019 + 10n | UDP 2019 |
| MAVLink master | 5760 + 10n | TCP 5760 |
| SITL 端口 | 5501 + 10n | 5501 |
| MAVLink 输出 | 14550 + 10n | UDP 14550 |
| MAVProxy 辅助输出 | 固定 | UDP 14551 |
| SYSID | n + 1 | 1 |

多实例时需成套改变实例号、JSON、DDS、MAVLink 和 SYSID。

## Gazebo 桥

共同核心数据：/clock、joint_states、odometry、gz/tf、gz/tf_static。

Iris 还包含 camera image、camera_info、air_pressure、imu、magnetometer、navsat、gpsfix、battery。Alti 当前主要为时钟、关节、里程计、TF、IMU。

## ArduPilot DDS 话题

本机观察到：

- /ap/airspeed
- /ap/battery
- /ap/clock
- /ap/cmd_gps_pose
- /ap/cmd_vel
- /ap/geopose/filtered
- /ap/goal_lla
- /ap/gps_global_origin/filtered
- /ap/imu/experimental/data
- /ap/joy
- /ap/navsat
- /ap/pose/filtered
- /ap/rc
- /ap/status
- /ap/tf
- /ap/tf_static
- /ap/time
- /ap/twist/filtered

~~~bash
ros2 topic list -t
ros2 topic info -v /ap/status
ros2 topic hz /ap/pose/filtered
~~~

## DDS 服务

当前本机话题和服务都位于 /ap 命名空间：

| 服务 | 类型 |
|---|---|
| /ap/arm_motors | ardupilot_msgs/srv/ArmMotors |
| /ap/mode_switch | ardupilot_msgs/srv/ModeSwitch |
| /ap/prearm_check | std_srvs/srv/Trigger |
| /ap/experimental/takeoff | ardupilot_msgs/srv/Takeoff |
| /ap/set_parameters | rcl_interfaces/srv/SetParameters |
| /ap/get_parameters | rcl_interfaces/srv/GetParameters |

~~~bash
ros2 service list -t
ros2 interface show ardupilot_msgs/srv/ArmMotors
ros2 interface show ardupilot_msgs/srv/ModeSwitch
ros2 interface show ardupilot_msgs/srv/Takeoff
~~~

SITL 默认在 TCP 5760 提供 MAVLink master，并向 UDP 14550 输出。集成 MAVProxy 再转发到 127.0.0.1:14551。
