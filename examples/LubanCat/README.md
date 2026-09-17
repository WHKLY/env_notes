# 在另一台 Ubuntu 电脑上复用 LubanCat USB 桌面与共享网络

快照日期：2026-09-17。

本文说明如何把已经配置好的同一块 LubanCat-5 V2 接到另一台 Ubuntu 电脑，并恢复以下能力：

- 通过唯一的 USB Type-C 口使用 ADB；
- 新电脑通过 USB RNDIS 直接访问板子；
- 新电脑把自己的 Wi-Fi 或有线网络共享给板子；
- 在新电脑上通过 NoMachine 操作板子的无头 GNOME 桌面；
- 保留板子的手机 USB 共享网络备用配置。

板端 Gadget、固定地址和 NoMachine 无头配置已经持久化，因此更换电脑时通常不需要重新修改板子。

## 命令归属规则

本文严格区分命令运行位置：

- **新电脑执行**：直接在新电脑的 Ubuntu 终端中输入。
- **板子执行**：进入 `adb shell` 后，在 LubanCat shell 中输入。
- **新电脑执行，通过 ADB 在板子内运行**：命令从新电脑发起，但引号内部分由板子执行。

不要在板子 shell 中运行 `sudo apt install adb`、`nmcli connection add LubanCat-USB` 或 NoMachine 客户端安装命令；这些属于新电脑配置。

## 1. 前提与地址规划

本教程假定：

- 使用的是已经完成配置的同一块 LubanCat-5 V2；
- 板子继续由外部电源单独供电；
- 新电脑使用 Ubuntu 和 NetworkManager；
- USB 数据线支持数据传输；
- 新电脑的其他网络没有占用 `10.42.0.0/24`。

固定拓扑：

~~~text
新电脑上游网络（Wi-Fi 或以太网）
            |
    NetworkManager shared/NAT
            |
新电脑 USB 网卡  10.42.0.1/24
            |
       USB Type-C
       ADB + RNDIS
            |
LubanCat usb0      10.42.0.2/24
~~~

板端已保存：

- `/etc/init.d/.usb_config`：同时启用 ADB 与 RNDIS；
- `/etc/usbdevice.d/rndis.sh`：固定电脑侧 MAC 为 `02:42:2a:00:00:01`，板端 MAC 为 `02:42:2a:00:00:02`；
- NetworkManager `LubanCat-PC`：板端地址 `10.42.0.2/24`，网关和 DNS 为 `10.42.0.1`；
- NetworkManager `Phone-USB`：手机 USB DHCP 备用链路，metric 600；
- NoMachine 9.9.6 无头虚拟桌面配置。

## 2. 在新电脑安装 ADB

### 新电脑执行

~~~bash
sudo apt update
sudo apt install adb usbutils
adb version
~~~

给板子接通外部电源，再用 Type-C 数据线连接新电脑。检查 USB 和 ADB：

### 新电脑执行

~~~bash
lsusb | grep 2207
adb devices -l
~~~

预期 USB VID:PID 为 `2207:0019`，ADB 序列号为 `0be02ce36007f008`。ADB 输出里的 `Nexus_4`、`occam` 和 `mako` 是 Gadget 模板描述，不是实际板型；真实板型由 Device Tree 确认为 LubanCat-5 V2。

若 ADB 首次显示 `unauthorized`，按新电脑和板子界面的授权提示处理；当前板端配置通常会直接允许 USB ADB。

## 3. 找到新电脑上的 RNDIS 接口

板端已固定电脑侧 MAC，因此 Ubuntu 通常把接口命名为 `enx02422a000001`。

### 新电脑执行

~~~bash
ip -o link | grep -i '02:42:2a:00:00:01'
ip -brief link show enx02422a000001
~~~

如果第一条命令显示了不同接口名，后续所有新电脑命令都应使用实际名称替换 `enx02422a000001`。

如果 ADB 已出现但找不到该 MAC，先跳到“故障排查”检查板端 Gadget 配置，并做一次完整断电冷启动。

## 4. 在新电脑建立 USB 共享网络

这一步只配置新电脑。不要在板子的 NetworkManager 中创建同名 `LubanCat-USB`。

### 新电脑执行

确认没有同名连接：

~~~bash
nmcli connection show LubanCat-USB
~~~

在全新的电脑上，该命令通常会报告连接不存在，此时创建连接：

如果连接已经存在，先用 `nmcli connection show LubanCat-USB` 核对其接口和地址，然后跳过下面的 `connection add`，直接执行后续 `connection modify`。不要创建多个同名连接。

~~~bash
sudo nmcli connection add \
  type ethernet \
  ifname enx02422a000001 \
  con-name LubanCat-USB
~~~

设置固定地址、自动连接和 IPv4 共享：

~~~bash
sudo nmcli connection modify LubanCat-USB \
  connection.autoconnect yes \
  ipv4.method shared \
  ipv4.addresses 10.42.0.1/24 \
  ipv6.method disabled
~~~

启用连接：

~~~bash
sudo nmcli connection up LubanCat-USB
~~~

检查结果：

~~~bash
ip -brief address show enx02422a000001
nmcli -f NAME,TYPE,DEVICE connection show --active
nmcli connection show LubanCat-USB
~~~

预期新电脑接口获得 `10.42.0.1/24`。NetworkManager 的 `shared` 模式会自动提供 DNS 转发和 NAT，并使用新电脑当前的默认上游网络；不要求上游接口也叫 `wlp4s0`。

如果 NetworkManager 已自动为该 USB 网卡创建了类似 `Wired connection 2` 的连接，执行 `nmcli connection up LubanCat-USB` 时会切换到本教程创建的固定配置。

## 5. 验证电脑与板子直连

### 新电脑执行

~~~bash
ping -c 3 10.42.0.2
~~~

### 新电脑执行，通过 ADB 在板子内运行

以下命令由新电脑终端发起，但单引号里的 `ip`、`ping` 和 `getent` 实际在板子上运行：

~~~bash
adb shell 'ip -brief address show usb0'
adb shell 'ip route'
adb shell 'ping -c 2 -W 2 10.42.0.1'
adb shell 'getent ahostsv4 download.nomachine.com | head -1'
~~~

预期板端状态：

~~~text
usb0  UP  10.42.0.2/24
default via 10.42.0.1 dev usb0 proto static metric 50
~~~

能够解析外部域名说明板子已经通过新电脑获得 DNS 和外网路由。

也可以先进入板子，再逐条运行同样的检查：

### 新电脑执行

~~~bash
adb shell
~~~

### 板子执行

~~~bash
ip -brief address show usb0
ip route
ping -c 2 -W 2 10.42.0.1
getent ahostsv4 download.nomachine.com | head -1
exit
~~~

## 6. 在新电脑安装 NoMachine 客户端

从 NoMachine 官方网站下载适用于新电脑架构的 Enterprise Client `.deb`。普通 x86-64 Ubuntu 电脑选择 amd64 包。当前已验证的客户端是 10.1.7-1，兼容板端 NoMachine 9.9.6。

### 新电脑执行

进入下载目录后安装：

~~~bash
sudo apt install ./nomachine-enterprise-client_*_amd64.deb
/usr/NX/bin/nxplayer
~~~

在 NoMachine 图形界面中新建连接：

| 项目 | 值 |
|---|---|
| 连接名 | `LubanCat-5-V2-USB` |
| 协议 | NX |
| 主机 | `10.42.0.2` |
| 端口 | `4000` |
| 用户 | `cat` |
| 认证 | 板子当前的系统用户密码 |
| 桌面 | 默认 X session；无物理桌面时创建新的虚拟桌面 |

首次连接时核对并接受服务器身份。板子默认启动到 `multi-user.target`，NoMachine 会按需创建虚拟 GNOME X11 桌面，不需要外接 HDMI 显示器。

不要直接复制旧电脑已经使用过的 `.nxs` 文件：它可能包含认证字段、服务器身份、会话截图和窗口状态。新电脑重新创建连接更安全。

## 7. 可选：建立桌面快捷方式

NoMachine 保存连接后，先确认会话文件的真实位置：

### 新电脑执行

~~~bash
find "$HOME/Documents/NoMachine" -maxdepth 1 -type f -name '*.nxs' -print
~~~

创建 `~/Desktop/LubanCat-NoMachine.desktop`。下面的 `<新电脑用户名>` 和 `<实际会话文件名>` 必须替换，Desktop Entry 不会自动展开这些占位符。

~~~ini
[Desktop Entry]
Type=Application
Name=LubanCat-5 V2 (USB)
Comment=Connect to LubanCat graphical desktop over USB
Exec=/usr/NX/bin/nxplayer --session /home/<新电脑用户名>/Documents/NoMachine/<实际会话文件名>.nxs
Icon=/usr/NX/share/icons/128x128/apps/nxplayer.png
Terminal=false
Categories=Network;RemoteAccess;
~~~

### 新电脑执行

~~~bash
chmod +x "$HOME/Desktop/LubanCat-NoMachine.desktop"
~~~

桌面环境若提示快捷方式不受信任，再在文件属性中选择“允许启动”。

## 8. 同步环境文档

主机侧配置说明保存在公开仓库 `WHKLY/env_notes`：

### 新电脑执行

~~~bash
sudo apt install git
mkdir -p "$HOME/Documents"
git clone https://github.com/WHKLY/env_notes.git \
  "$HOME/Documents/env_notes"
~~~

板子自身的详细环境文档位于私有仓库 `WHKLY/lubancat-env-notes`。新电脑已经获得该私有仓库访问权限并配置自己的 GitHub 身份时，优先独立 clone：

### 新电脑执行

~~~bash
git clone git@github.com:WHKLY/lubancat-env-notes.git \
  "$HOME/Documents/lubancat-board-env_notes"
~~~

不要把板子的 Deploy Key 私钥复制到新电脑。新电脑应使用自己的 GitHub 账户凭据；若只需要临时离线快照，可以通过 ADB 复制文档但排除 `.git`：

### 新电脑执行

~~~bash
mkdir -p "$HOME/Documents/lubancat-board-env_notes-snapshot"
adb exec-out 'tar -C /home/cat/Documents/env_notes --exclude=.git -cf - .' |
  tar -C "$HOME/Documents/lubancat-board-env_notes-snapshot" -xf -
~~~

板端仓库只包含系统、驱动、RKNN、项目产物和维护风险等文本记录，不包含模型文件、图片数据或 SSH 密钥。

## 9. 故障排查

### 9.1 ADB 正常，但新电脑没有 RNDIS 网卡

### 新电脑执行，通过 ADB 在板子内运行

~~~bash
adb shell 'cat /etc/init.d/.usb_config'
adb shell 'cat /etc/usbdevice.d/rndis.sh'
adb shell 'cat /sys/kernel/debug/usb/fc000000.usb/mode'
~~~

预期 `.usb_config` 同时包含 `usb_adb_en` 与 `usb_rndis_en`，USB 控制器模式为 `device`。固定 MAC 钩子修改后必须完整断电冷启动，热重启 Gadget 可能继续使用旧 MAC。

### 9.2 能 ping 板子，但 NoMachine 无法连接

### 新电脑执行

~~~bash
ping -c 2 10.42.0.2
~~~

### 新电脑执行，通过 ADB 在板子内运行

~~~bash
adb shell '/usr/NX/bin/nxserver --status'
adb shell 'ss -lnt "sport = :4000"'
~~~

`nxserver` 和 `nxd` 应 enabled，TCP 4000 应监听。没有活动桌面时 `nxnode` 显示 disabled 属正常现象。

### 9.3 出现 `<no available desktops on this server>`

### 新电脑执行，通过 ADB 在板子内运行

~~~bash
adb shell 'systemctl get-default'
adb shell 'systemctl is-active display-manager || true'
adb shell 'grep -nE "^(DefaultDesktopCommand|VirtualDesktopVariables)" /usr/NX/etc/node.cfg'
adb shell 'cat /usr/local/bin/nomachine-virtual-env.sh'
~~~

预期：

- 默认目标为 `multi-user.target`；
- `display-manager` 为 inactive；
- `VirtualDesktopVariables` 指向 `/usr/local/bin/nomachine-virtual-env.sh`；
- 环境脚本启用 Mesa、llvmpipe 和 X11。

### 9.4 `10.42.0.0/24` 与新电脑现有网络冲突

先检查：

### 新电脑执行

~~~bash
ip -4 route
nmcli connection show --active
~~~

如果已有热点、VPN、容器网络或其他 USB 共享占用了该网段，不要只改新电脑一端。必须协调修改：

1. 新电脑 `LubanCat-USB` 的地址；
2. 板端 `LubanCat-PC` 的地址、网关和 DNS；
3. NoMachine 的目标地址。

地址切换会短暂中断网络，但 ADB 通常仍可作为恢复入口。具体更改方法见 [USB 网络环境说明](../../LubanCat/02-usb-network.md)。

## 10. 完成判据

复用成功需要同时满足：

- 新电脑的 `adb devices -l` 能识别板子；
- 新电脑 USB 接口为 `10.42.0.1/24`；
- 新电脑能 ping `10.42.0.2`；
- 板端 `usb0` 为 `10.42.0.2/24`；
- 板端默认路由指向 `10.42.0.1` 且能解析外部域名；
- 新电脑能连接 `10.42.0.2:4000`；
- 拔掉板子外接显示器后，NoMachine 桌面仍可显示、点击和输入；
- 手机 USB 共享配置仍保留，没有为了电脑链路而删除。

板子只有一个 Type-C Gadget 口，因此同一时刻只能通过该 USB 口直连一台电脑。若需要多台电脑同时访问，应让其他电脑通过以太网、ZeroTier 或已建立的 IP 网络连接，而不是同时占用该 Type-C 口。
