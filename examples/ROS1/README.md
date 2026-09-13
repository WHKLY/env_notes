# ROS1 示例学习路径

ROS1 示例依赖 `/home/xanter/ros1simulation/infra/noetic_gazebo11` 提供的固定容器镜像。容器运行 Noetic、Gazebo Classic 11、gazebo_ros 和 MAVROS；Ubuntu 24.04 主机运行 ArduPilot SITL。双方通过 host network 上的 UDP 通信。

先阅读[基础设施](00-noetic-gazebo11-infra.md)，再运行[Iris 基线](01-iris-setuptest.md)。Iris 成功后，继续阅读 [Zephyr 的构建](02-zephyr-circuit-build.md)与[运行](03-zephyr-circuit-run.md)。这种顺序先证明公共链路，再把故障范围收缩到固定翼模型、任务或控制器。

## 两个实验的区别

| 项目 | Iris 基线 | Zephyr 闭环 |
|---|---|---|
| 飞控 | ArduCopter | ArduPlane |
| frame | `gazebo-iris` | `gazebo-zephyr` |
| 操作 | 不解锁，只观察 | 人工输入 `ARM`，随后 AUTO |
| 控制器 | 无 | pymavlink 任务控制器 |
| 成功判据 | `/clock`、MAVROS、数据话题、bag | 基线判据加起飞、LAND、自动解除武装 |
| 作用 | 隔离并验证公共基础设施 | 验证普通固定翼全链路 |

ROS1 主机 shell 不需要 source Noetic。ROS 命令由容器内脚本执行。主机只需要 Docker、GNOME Terminal、`/home/xanter/Projects/ardupilot` 和 `/home/xanter/venv-ardupilot`。
