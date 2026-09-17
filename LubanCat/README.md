# LubanCat-5 V2 直连环境

快照日期：2026-09-17。

本目录记录 Ubuntu 主机通过 LubanCat-5 V2 唯一的 USB Type-C 口同时使用 ADB、RNDIS 网络和 NoMachine 图形桌面的最终配置。板子使用独立外部电源；主机经 USB 为板子提供网络，不依赖 USB 供电。

## 当前链路

| 功能 | 入口 | 状态 |
|---|---|---|
| ADB shell | `adb shell` | 冷启动后自动恢复 |
| USB 网络 | 主机 `10.42.0.1`，板子 `10.42.0.2` | 固定地址，主机共享上游网络 |
| 图形桌面 | NoMachine，`10.42.0.2:4000` | 无外接显示器时创建虚拟 GNOME 桌面 |
| 手机 USB 共享 | 板子 `usb1`，DHCP | 保留，路由优先级低于电脑 USB |
| 板端环境文档 | `/home/cat/Documents/env_notes` | 独立 Private GitHub 仓库，`main` 分支 |

板子接入后的常用操作：

~~~bash
adb devices -l
ping -c 2 10.42.0.2
/usr/NX/bin/nxplayer --session /home/xanter/Documents/NoMachine/LubanCat-5-V2-USB.nxs
~~~

NoMachine 登录使用板子系统用户 `cat` 和该用户当前密码。密码不保存在本仓库。

## 文档

- [01-environment.md](01-environment.md)：主机和板子的实测环境事实。
- [02-usb-network.md](02-usb-network.md)：USB Gadget、固定地址、网络共享和手机备用链路。
- [03-nomachine.md](03-nomachine.md)：客户端入口、板端无头桌面实现及故障恢复。
- [04-future-roadmap.md](04-future-roadmap.md)：Ubuntu 20 eMMC 基线、Ubuntu 24 SD 卡试验和未来 Ubuntu 24 eMMC 迁移路线。
- [另一台 Ubuntu 电脑复用教程](../examples/LubanCat/README.md)：从新电脑开始恢复 ADB、USB 共享网络和 NoMachine，逐段标明命令运行位置。

## 安全与边界

- `/home/xanter/Documents/NoMachine/LubanCat-5-V2-USB.nxs` 由 NoMachine 客户端维护，可能包含认证字段、主机指纹和会话截图，不复制到公开仓库。
- 不在文档、脚本或 Git 历史中保存板子密码。
- 本机 `WHKLY/env_notes` 是公开仓库，板端 `WHKLY/lubancat-env-notes` 是独立私有仓库；不要把板端 Deploy Key 或私有数据复制到本机仓库。
- 板端环境仓库只保存 Markdown、变更记录和 `.gitignore`，不保存模型、图片、SDK、日志或构建产物。
- 板子当前把 `multi-user.target` 设为默认启动目标，这是稳定无头 NoMachine 桌面的组成部分；恢复本地图形登录前先阅读 [03-nomachine.md](03-nomachine.md)。
