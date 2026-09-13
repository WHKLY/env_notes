# 本机环境笔记

本目录保存当前计算机的开发与仿真环境事实、操作手册和已验证问题记录，供本人及后续 AI 会话快速恢复上下文。

## 目录

- [ROS1/README.md](ROS1/README.md)：ROS Noetic、Gazebo Classic、MAVROS 和 ArduPilot SITL 联合仿真链。
- [ROS2/README.md](ROS2/README.md)：ROS 2、Gazebo、ArduPilot SITL 及相关联合仿真链。
- [examples/README.md](examples/README.md)：三个已验证实验的分层复现教程，包含代码改动、操作步骤、判据与历史证据。

仿真实验定义位于 `/home/xanter/ros1simulation` 和 `/home/xanter/ros2simulation`；本仓库只保存环境事实和教程，不复制大型 run 数据。

## 版本管理

本目录自 2026-09-13 起作为独立的本地 Git 仓库维护，默认分支为 main。

- 环境版本、路径、补丁或操作流程发生变化时，同步更新对应文档。
- 新增或修改文档后先检查内容，再提交到本仓库。
- 仓库当前不配置远程地址；需要备份或同步时再单独决定。
- 不提交密码、令牌、私钥、运行日志、构建输出或临时编辑器文件。

历史记录见 [CHANGELOG.md](CHANGELOG.md)。
