# 架构与数据链

~~~mermaid
flowchart LR
  GZ[Gazebo Harmonic<br/>物理、模型、传感器]
  P[ArduPilotPlugin<br/>JSON FDM]
  S[SITL<br/>arducopter / arduplane]
  X[Micro XRCE-DDS Agent]
  R[ROS 2 Jazzy]
  B[ros_gz_bridge]
  D[robot_state_publisher]
  V[RViz]
  M[MAVProxy / pymavlink]
  GZ <--> P
  P <-->|UDP 9002 + 10n| S
  S <-->|DDS UDP 2019 + 10n| X
  X <--> R
  GZ <-->|Gazebo Transport| B
  B <--> R
  D --> R
  R --> V
  S <-->|MAVLink| M
~~~

n 为实例号，默认 n=0。

## 组件职责

- Gazebo Harmonic 负责动力学、碰撞、世界和传感器。
- ArduPilotPlugin 在 Gazebo 状态、JSON FDM 和执行器输出之间转换。
- SITL 运行飞控代码；Iris 用 arducopter，Alti Transition 用 arduplane。
- XRCE-DDS Agent 将 ArduPilot 的 DDS 客户端接入 ROS 2 图。
- ros_gz_bridge 桥接 /clock、IMU、里程计、关节和其他传感器。
- robot_state_publisher 发布 robot_description、tf、tf_static。
- ros_gz_sim create 等待 robot_description 并在 Gazebo 创建模型。
- RViz 展示 ROS 数据，MAVProxy/pymavlink 提供 MAVLink 控制。

## 正常次序

1. launch 启动 Gazebo server 和 GUI。
2. robot_state_publisher 解析 SDF。
3. ros_gz_sim create 创建实体。
4. 模型内 ArduPilotPlugin 监听 JSON。
5. SITL 与插件连接。
6. XRCE Agent 建立会话，飞控输出 DDS: Initialization passed。
7. bridge 发布 ROS 数据，RViz 开始显示。

若模型创建失败，JSON、DDS 和 MAVLink 现象可能一起异常，应先处理 SDF 和资源路径。

## 模型

Iris 入口是 iris_runway.launch.py，使用 arducopter；参数包含 copter.parm、gazebo-iris-gimbal.parm、dds_udp.parm、dds_use_ns.parm。它桥接相机、气压计、IMU、磁力计、NavSat、GPSFix、电池等数据。

Alti Transition 入口是 alti_transition_runway.launch.py，使用 arduplane 和 alti_transition_quad；参数包含 quadplane.parm、alti_transition_quad.param、dds_udp.parm、dds_use_ns.parm。它是当前已经完整验证的一键 ROS 2 固定翼 QuadPlane 入口。zephyr、skywalker_x8 等常规固定翼模型有资源，但尚无同等级的一键 bringup 验证。

Gazebo 通过 /clock 提供仿真时间，自建节点应使用 use_sim_time=true。use_gz_tf:=true 会桥接 Gazebo TF；增加发布者前需检查是否与 robot_state_publisher 重复。
