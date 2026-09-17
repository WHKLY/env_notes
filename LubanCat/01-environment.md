# 环境事实

快照日期：2026-09-17，以下状态已在断电重启后重新核对。

## 主机

| 项目 | 当前值 |
|---|---|
| 系统 | Ubuntu 24.04.5 LTS |
| 内核 / 架构 | 7.0.0-31-generic / x86_64 |
| 无线接口 | `wlp4s0`，作为当前上游网络 |
| ADB | 1.0.41，Debian 34.0.4，`/usr/bin/adb` |
| NoMachine 客户端 | `nomachine-enterprise-client` 10.1.7-1 |
| NoMachine 程序 | `/usr/NX/bin/nxplayer` |
| USB 网络接口 | `enx02422a000001`，`10.42.0.1/24` |
| USB 网络配置 | NetworkManager `LubanCat-USB` |

本机连接入口：

- NoMachine 会话：`/home/xanter/Documents/NoMachine/LubanCat-5-V2-USB.nxs`；
- 桌面快捷方式：`/home/xanter/Desktop/LubanCat-NoMachine.desktop`；
- 目标服务器：`10.42.0.2:4000`，NX 协议，系统用户认证。

当前 USB 复合设备的 VID:PID 为 `2207:0019`。仅启用 ADB 时曾显示为 `2207:0006`。ADB 当前序列号为 `0be02ce36007f008`；USB 描述中的 `Nexus_4`、`occam` 和 `mako` 来自 Gadget 模板，不代表真实硬件型号。

## 板子

| 项目 | 当前值 |
|---|---|
| 型号 | Embedfire LubanCat-5 V2 |
| Device Tree | `embedfire,rk3588-lubancat-5-v2`、`rockchip,rk3588` |
| SoC / 架构 | RK3588 / aarch64 |
| 系统 | Ubuntu 20.04.6 LTS |
| 内核 | 5.10.160，构建号 `#15`（2026-01-05） |
| 内存 | 8,106,688 KiB，约 8 GB |
| 根分区 | `/dev/mmcblk0p3`，58 GB，核对时使用 18% |
| 启动分区 | `/dev/mmcblk0p2`，124 MB，核对时使用 61% |
| 常用用户 | `cat`，UID 1000 |
| NoMachine 服务端 | `nomachine` 9.9.6-2 |
| USB 网络接口 | `usb0`，`10.42.0.2/24` |
| ZeroTier | `ztxoogfbja`，`10.67.218.100/24` |
| Git | 2.25.1，`/usr/bin/git` |
| 板端环境仓库 | `/home/cat/Documents/env_notes`，`main` 分支 |
| 板端 GitHub 远端 | `WHKLY/lubancat-env-notes`，Private |

ADB 以 root 身份进入板子。日常图形登录应继续使用普通用户 `cat`。

## 环境文档版本管理

本机与板子的环境文档使用两个独立仓库：

| 范围 | 本地路径 | GitHub 仓库 | 可见性 |
|---|---|---|---|
| 本机综合环境 | `/home/xanter/Documents/env_notes` | `WHKLY/env_notes` | Public |
| LubanCat 板端环境 | `/home/cat/Documents/env_notes` | `WHKLY/lubancat-env-notes` | Private |

板端仓库由 `cat` 用户维护，使用仅限该私有仓库的 Deploy Key。首次提交 `0cecfe369d8ae89296a99bbca4afd897a8b74814` 已推送并与远端 `main` 对齐。密钥文件不属于环境文档，不在本机仓库和板端仓库之间复制。

## 重启验证结果

2026-09-17 的完整断电重启中，USB/RNDIS 约 12 秒后出现，ADB 约 14 秒后可用。随后确认：

- 主机接口名称、MAC 和 `10.42.0.1/24` 未变化；
- 板子 `usb0` 为 `10.42.0.2/24`，默认路由指向主机；
- 板子能够解析外部域名；
- 默认目标仍为 `multi-user.target`，`display-manager` 未启动；
- NoMachine `nxd` 在 TCP 4000 的 IPv4 和 IPv6 地址监听；
- 用户已实际验证无外接显示器时可以进入桌面并正常点击操作。
