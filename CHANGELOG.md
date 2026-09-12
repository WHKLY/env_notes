# 变更记录

## 2026-09-13

- 将 /home/xanter/Documents/env_notes 初始化为独立的本地 Git 仓库。
- 建立 main 分支，并使用仓库本地作者信息 xanter <xanter@localhost>。
- 纳入现有 ROS 2、Gazebo Harmonic、ArduPilot SITL、DDS、bridge 和 RViz 环境文档。
- 新增 ROS 2 仿真踩坑记录 tips.md，记录本次 Skywalker X8 联合仿真中确认的问题与处理方法。
- 当前仓库未配置远程地址。
- 新增 ROS1 环境文档，记录 Noetic/Gazebo Classic/MAVROS/Docker 的实测版本与架构。
- 记录 ROS1 Iris 联合仿真基线成功、失败过程、最终 rosbag 证据和一键停止归档逻辑。
- 新增 ROS1 Zephyr 普通固定翼任务定义；人工解锁后执行 AUTO 起飞、绕场与降落，当前仅完成静态检查。
- 修复 ROS1 Zephyr 启动前检查对代理环境 ripgrep 的隐式依赖，用户脚本改用 grep。
- 修复 ROS1 Zephyr 视觉网格与桨叶碰撞体导致的地面强振动和 `Accels inconsistent`：改用简化机身碰撞体、三点低摩擦滑橇并移除桨叶碰撞体。
- 完成 Zephyr 普通固定翼动态全任务验证：人工解锁、AUTO 起飞、四边航线、自动降落和自动解除武装均成功；归档 run 为 `run_20260913T015818+0800`。
