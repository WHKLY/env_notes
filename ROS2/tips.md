# ROS 2 仿真踩坑记录

> 最后更新：2026-09-13（Asia/Shanghai）
>
> 适用环境：本机 ROS 2 Jazzy、Gazebo Harmonic、ArduPilot SITL、Micro XRCE-DDS、ros_gz_bridge 和 RViz 联合仿真链。

本文记录本机已经实际遇到的问题。开始新的仿真实验前先浏览一遍；出现故障时，再按 [06-troubleshooting.md](06-troubleshooting.md) 执行诊断命令。

## 1. 每次运行必须使用独立工作目录

**现象：** 家目录出现 mav.tlog、mav.tlog.raw、mav.parm、terrain/、eeprom.bin 或 SITL 飞行日志。

**原因：** MAVProxy 和 SITL 会把运行文件写入进程的当前工作目录。从家目录启动时，这些文件自然落到家目录；这不是 /usr/bin 被污染。

**规则：**

- 每次运行新建 runs/run_YYYYMMDDTHHMMSS+ZZZZ/，并把 GNOME Terminal 的工作目录设为该目录。
- SITL 使用 use_instance_dir:=True。
- 不复用已经生成结果的 run 目录，也不覆盖旧日志。
- 停止后运行实验自己的归档脚本，将 MAVLink、SITL、rosbag 和指标保留在本次 run 内。

## 2. 加载 ROS 环境时谨慎使用 set -u

**现象：** 脚本在 source ROS 或 overlay 的 setup.bash 时，报告 AMENT_TRACE_SETUP_FILES、COLCON_TRACE 等变量未定义并退出。

**原因：** ROS/colcon 的环境脚本会读取一些允许未定义的变量，而 Bash nounset 模式把这种读取当成错误。

**处理：**

~~~bash
set +u
source /opt/ros/jazzy/setup.bash
source /home/xanter/ros2_ws/install/setup.bash
set -u
~~~

set -euo pipefail 可以继续用于自己的脚本，但不要让 -u 包住 ROS setup 文件。

## 3. package.xml 元数据错误会让包类型识别异常

**现象：** colcon 构建似乎完成，但 ros2 launch 报 Package not found；colcon list 可能把 ament_python 包识别成普通 Python 包。

**原因：** 本次实验最初使用了无效的维护者邮箱 xanter@localhost，导致清单校验和包识别异常。

**规则：**

- package.xml 使用合法邮箱格式，例如 xanter@example.com。
- 构建前确认 colcon list 显示正确包名和类型。
- 构建后加载实验 overlay，再用 ros2 pkg prefix 验证 ROS 能找到它。

## 4. Jazzy 的 sdformat_urdf 不接受 Skywalker 根级 pose

**现象：** Gazebo 相关进程已经启动，但 robot_state_publisher 退出，随后 ros_gz_sim create 一直等不到 robot_description。

**原因：** Skywalker X8 原始 SDF 在 model 根级包含 pose。本机 Jazzy 使用的 sdformat_urdf 无法把这种结构转换成 URDF。

**规则：**

- 只对供 robot_state_publisher 和 spawn 使用的临时 SDF 移除 model-level pose。
- 出生位置和方向通过 launch 的 x/y/z/R/P/Y 参数设置。
- 不直接改公共模型原件；转换文件属于实验的派生输入。

## 5. package:// 和 model:// URI 不能假定所有工具都能解析

**现象：** Gazebo 能找到一部分资源，但 sdformat_urdf、模型生成或 RViz 在解析 include/mesh 时失败。

**原因：** Gazebo、ament、sdformat 和 RViz 的资源解析上下文不同。某个 URI 能被 Gazebo 解析，不代表转换工具也能解析。

**处理：**

- 本机公共 robot.launch.py 已在调用 sdformat_urdf 前把 package:// 解析为 file://。
- Skywalker 实验会把 model://skywalker_x8 转为实验包可解析的 URI。
- 更新 ardupilot_gz 后若问题复现，检查本机补丁是否还在：

~~~bash
git -C /home/xanter/Projects/ardupilot_gz diff --   ardupilot_gz_bringup/launch/robots/robot.launch.py
~~~

## 6. 模型网格可能同时引用多个纹理文件名

**现象：** Gazebo 中模型基本正常，但 RViz 报纹理缺失，或同一模型在两个程序中的外观不同。

**原因：** Skywalker 的 Collada 网格实际同时引用 x8_tga.jpg 和 x8.tga。只安装其中一个文件不能满足所有引用。

**规则：**

- 实验包同时自包含并安装 x8_tga.jpg 与 x8.tga。
- 排查纹理时直接检查 DAE 内的文件引用，注意大小写和扩展名。
- 纹理缺失通常不阻止动力学或 DDS，但仍应修复，以免掩盖资源路径问题。

## 7. bridge 配置不会凭空生成 Gazebo 话题

**现象：** bridge YAML 已配置 /odometry，ROS 2 侧却一直没有里程计消息。

**原因：** ros_gz_bridge 只转发已经存在的 Gazebo Transport 话题。Skywalker 原始模型没有加载 OdometryPublisher，因此没有上游消息可桥接。

**规则：**

- 实验 launch 会向临时 SDF 注入 gz-sim-odometry-publisher-system。
- 先确认 Gazebo 侧话题存在，再检查 bridge 配置，最后检查 ROS 2 侧话题。
- 话题出现在 ros2 topic list 中不等于正在发布数据；用 echo --once 或频率检查确认。

## 8. 本机 ArduPilot DDS 接口没有 /v1 层

**现象：** DDS session 已建立，但等待 /ap/v1/status 的脚本或 rosbag 长时间不开始记录。

**原因：** 早期配置使用了错误接口名。本机实测话题是 /ap/status、/ap/pose/filtered 等，服务是 /ap/prearm_check、/ap/arm_motors 等。

**规则：**

- 以运行时 ros2 topic list 和 ros2 service list 为准。
- 本机当前实验和文档统一使用 /ap/...，不要添加 /v1。
- rosbag 等待话题时应输出清晰日志或设置超时，避免把“没有数据”误认为 bag 崩溃。

## 9. DDS 初始化成功需要同时满足 SITL 与 Agent 条件

**现象：** Gazebo 和 SITL 已运行，但 ROS 2 看不到 /ap/...；或者 Agent 在运行，却没有 client session。

**原因：** 只启动 Micro XRCE-DDS Agent 不够，ArduPilot SITL 还必须启用 DDS 并连到一致的 UDP 端口。本机当前使用 2019。

**规则：**

- 正常日志应同时出现 XRCE client/session 建立和 DDS: Initialization passed。
- 核对 UDP 2019 是否被旧实例占用。
- 启动新实验前确认没有旧的 SITL、Agent、bridge 或 Gazebo 进程残留。

## 10. 人工解锁和自动任务之间要有明确闸门

“手动解锁、随后自动飞行”包含两个责任：操作者授权解锁，控制器检测 armed 后接管任务。

**规则：**

- 控制器先上传任务，再写 MISSION_READY，然后等待 armed。
- arm.sh 只在看到 MISSION_READY 后提示输入 ARM，再调用 prearm 和 arm 服务。
- 控制器自身不得解锁；检测到真实 armed 状态后才能切换 AUTO。
- 解锁前确认载具类型、任务、模式、仿真实例和周围无旧进程。

## 11. “任务完成”和“精准落点”是两个判据

本次 run_20260912T235453+0800 成功走完起飞、绕场、LAND 和自动解除武装，最高相对高度约 60.47 m。飞机最终停在 HOME 以南约 155.8 m；相对设在 HOME 前方 40 m 的接地点，纵向约提前 195.8 m，横向误差约 0.4 m。

**规则：**

- 功能成功判据是任务到 LAND、正常离地、最终自动解除武装。
- 落点精度单独记录，不能因任务闭环成功而省略。
- 后续优化检查进近方向、下滑线起点与高度、空速、风和 ArduPlane 着陆参数，并用重复实验评估。

## 12. 手动结束当前不会自动归档

**现象：** 任务结束后关闭主 launch 和 rosbag，run 目录仍没有 DONE/FAILED，运行文件还在 runtime/。

**原因：** 当前 ros2sim_260912_setuptest 的停止和归档是分开的。Ctrl+C 只终止进程，不会自动调用 finalize.sh。

**当前流程：**

1. 在主 GNOME Terminal 中按 Ctrl+C。
2. 在 rosbag GNOME Terminal 中按 Ctrl+C，让 MCAP 正常收尾。
3. 确认 Gazebo、ArduPlane、Agent、bridge、RViz 和 rosbag 已退出。
4. 在实验根目录运行 ./scripts/finalize.sh。

finalize.sh 会拒绝在相关进程仍运行时归档；存在 MISSION_DONE 时写 DONE，否则写 FAILED，并补充 manifest 的结束时间和状态。

当前脚本只整理 mav.tlog* 和精确文件名 mav.parm。某些运行还会生成 mav_0_1.parm，它暂时可能留在 runtime/；后续应扩展为 mav*.parm，并增加统一的 stop.sh。

## 13. 关闭顺序影响 rosbag 和日志完整性

**现象：** 强制结束整个终端或直接杀死进程后，MCAP 没有正常关闭、日志尾部缺失，或后台进程残留。

**规则：**

- 先停止主 launch 与 rosbag，等待正常退出，再归档。
- 不把关闭 GNOME Terminal 窗口当成唯一停止手段。
- 归档前检查相关进程和端口；存在残留时，不开始下一次同端口实验。

## 14. 实验配置只保留一个事实来源

**现象：** 根目录和 ROS 包各有一份 YAML/参数文件，修改其中一份后运行行为没有变化。

**原因：** 重复文件会逐渐不一致，而 launch 实际读取的可能是另一份。

**规则：**

- 实验根目录 config/ 保存受维护的原始配置。
- ROS 包中需要相同文件时使用符号链接或安装规则，避免手工复制。
- manifest 记录任务、参数和控制脚本的校验值，使结果对应到确切输入。

## 15. 一次成功启动至少核对四条链路

仅看到 Gazebo 画面不能证明联合仿真成功。至少确认：

1. **物理链：** 模型创建成功，ArduPilotPlugin 持续收到 JSON FDM。
2. **飞控链：** ArduPlane SITL 正常运行，模式、解锁与任务状态合理。
3. **ROS 2 链：** XRCE-DDS 初始化通过，/ap/status 等有实际消息。
4. **记录链：** rosbag 正在写 MCAP，任务控制器写遥测与最终结果。

任务结束后还要确认最终解除武装、结果 JSON、bag 正常关闭，以及 run 目录的 DONE/FAILED 标记。

## 相关文档

- [04-runbook.md](04-runbook.md)：日常启动、操作与停止命令。
- [05-interfaces.md](05-interfaces.md)：当前 ROS 2 话题、服务和端口。
- [06-troubleshooting.md](06-troubleshooting.md)：按症状执行的排查步骤。
- /home/xanter/ros2simulation/AGENTS.md：实验目录组织和记录规范。
- /home/xanter/ros2simulation/ros2sim_260912_setuptest/README.md：本次普通固定翼实验定义与验证结果。
