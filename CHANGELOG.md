# 变更记录

## 2026-09-17

- 新增 LubanCat-5 V2 环境文档，记录主机与板子的系统版本、设备身份和关键路径。
- 将板子唯一的 Type-C 口配置为 ADB + RNDIS 复合设备，并固定两端 MAC 与 `10.42.0.0/24` 地址。
- 主机通过 NetworkManager `shared` 模式为板子提供路由和 DNS；保留板子的手机 USB 共享网络配置作为低优先级备用链路。
- 记录 NoMachine 客户端、板端服务和无物理显示器虚拟桌面配置；板端虚拟桌面单独使用 Mesa llvmpipe，避免 Mali EGL 在无头环境中使 GNOME Shell 退出。
- 完成断电重启验证：USB 网络、ADB、默认路由、无头启动配置和 NoMachine 监听均能自动恢复。
- 新增另一台 Ubuntu 电脑复用教程，覆盖 ADB、固定 RNDIS 接口、NetworkManager 网络共享、NoMachine 客户端、环境文档同步和故障排查，并明确区分电脑端与板端命令。

## 2026-09-13

- 将 /home/xanter/Documents/env_notes 初始化为独立的本地 Git 仓库。
- 建立 main 分支，并使用仓库本地作者信息 xanter <xanter@localhost>。
- 纳入现有 ROS 2、Gazebo Harmonic、ArduPilot SITL、DDS、bridge 和 RViz 环境文档。
- 新增 ROS 2 仿真踩坑记录 tips.md，记录本次 Skywalker X8 联合仿真中确认的问题与处理方法。
- 当前仓库未配置远程地址。
- 新增 ROS1 环境文档，记录 Noetic/Gazebo Classic/MAVROS/Docker 的实测版本与架构。
- 记录 ROS1 Iris 联合仿真基线成功、失败过程、最终 rosbag 证据和一键停止归档逻辑。
- 新增 ROS1 Zephyr 普通固定翼任务定义；该提交阶段只完成静态检查，后续提交完成动态全任务验证。
- 修复 ROS1 Zephyr 启动前检查对代理环境 ripgrep 的隐式依赖，用户脚本改用 grep。
- 修复 ROS1 Zephyr 视觉网格与桨叶碰撞体导致的地面强振动和 `Accels inconsistent`：改用简化机身碰撞体、三点低摩擦滑橇并移除桨叶碰撞体。
- 完成 Zephyr 普通固定翼动态全任务验证：人工解锁、AUTO 起飞、四边航线、自动降落和自动解除武装均成功；归档 run 为 `run_20260913T015818+0800`。
- 全局复核 ROS1/ROS2 文档和当前本机状态，补充 `/home/xanter/ros1simulation/infra/noetic_gazebo11` 的结构、固定输入、构建、doctor、权限与服务生命周期。
- 新增 `examples/ROS1` 与 `examples/ROS2`，将 Iris 基线、Zephyr 固定翼和 Skywalker X8 固定翼三个已验证实验整理为分层复现教程。
- 修正 ROS2 旧状态：Skywalker X8 已完成全任务；标准实验运行结果位于 `ros2simulation/*/runs`，`sim_runs` 仅保留为公共 bringup 临时调试目录。
- 创建 GitHub 仓库 `WHKLY/env_notes`，初始设为私有，配置 SSH remote `origin` 并推送 `main`。
- 将 GitHub 仓库 `WHKLY/env_notes` 的可见性改为 Public，便于云端只读获取和协作。
