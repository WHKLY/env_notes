# USB 复合设备与主机共享网络

## 拓扑

~~~text
Internet
   |
Ubuntu 主机 wlp4s0
   |
NetworkManager shared/NAT
   |
enx02422a000001  10.42.0.1/24
   |  USB Type-C：ADB + RNDIS
usb0              10.42.0.2/24
   |
LubanCat-5 V2
~~~

电脑 USB 链路是首选默认路由。手机 USB 共享配置保留在 `usb1`，接入时作为较低优先级的备用网络，不需要为了电脑共享网络而停用手机 USB。

## 板子 USB Gadget

板子使用官方 `/usr/bin/usbdevice` 管理 ConfigFS Gadget。持久配置 `/etc/init.d/.usb_config` 为：

~~~text
usb_adb_en
usb_rndis_en
~~~

修改前的纯 ADB 配置保存在 `/etc/init.d/.usb_config.before-adb-rndis`。

RNDIS 默认会随机生成 MAC，导致主机接口名和 NetworkManager 绑定在重启后变化。为固定身份，`/etc/usbdevice.d/rndis.sh` 在创建 RNDIS 功能时写入：

~~~sh
#!/bin/sh

rndis_prepare()
{
    # Keep the USB network interface identity stable across gadget restarts.
    echo 02:42:2a:00:00:01 > host_addr
    echo 02:42:2a:00:00:02 > dev_addr
}
~~~

因此主机侧 MAC 为 `02:42:2a:00:00:01`，Linux 接口名稳定为 `enx02422a000001`；板子 `usb0` 的 MAC 为 `02:42:2a:00:00:02`。Gadget 热重启时已有 RNDIS ConfigFS 功能可能阻止 MAC 立即变化，修改该钩子后应做一次完整断电冷启动。

## 主机 NetworkManager

连接名 `LubanCat-USB` 的关键值：

| 属性 | 值 |
|---|---|
| 接口 | `enx02422a000001` |
| 自动连接 | `yes` |
| IPv4 方法 | `shared` |
| IPv4 地址 | `10.42.0.1/24` |
| IPv6 | `disabled` |

`shared` 模式由 NetworkManager 提供 DHCP/DNS 和 NAT，使板子经主机当前的上游连接访问网络。查看或调整配置：

~~~bash
nmcli connection show LubanCat-USB
nmcli connection modify LubanCat-USB ipv4.addresses 10.42.0.1/24
nmcli connection up LubanCat-USB
~~~

固定地址以后仍可更改，但必须同时修改主机、板子和 NoMachine 目标地址，且不要与上游局域网、VPN 或 ZeroTier 网段冲突。

## 板子 NetworkManager

电脑 USB 配置 `LubanCat-PC`：

| 属性 | 值 |
|---|---|
| 接口 | `usb0` |
| 自动连接 | `yes` |
| IPv4 方法 / 地址 | `manual` / `10.42.0.2/24` |
| 网关 / DNS | `10.42.0.1` / `10.42.0.1` |
| 路由 metric / DNS priority | `50` / `50` |
| IPv6 | `disabled` |

手机备用配置 `Phone-USB`：

| 属性 | 值 |
|---|---|
| 接口 | `usb1` |
| 自动连接 / 优先级 | `yes` / `50` |
| IPv4 / IPv6 | DHCP / auto |
| 路由 metric / DNS priority | `600` / `600` |

板子当前主路由应为：

~~~text
default via 10.42.0.1 dev usb0 proto static metric 50
~~~

若要改成另一网段，以 `10.43.0.0/24` 为例，应先保留当前 ADB 会话，再依次修改板子地址、主机共享地址和 NoMachine 会话目标：

~~~bash
# 板子内执行
nmcli connection modify LubanCat-PC \
  ipv4.addresses 10.43.0.2/24 ipv4.gateway 10.43.0.1 ipv4.dns 10.43.0.1

# 主机执行
nmcli connection modify LubanCat-USB ipv4.addresses 10.43.0.1/24
~~~

切换连接会暂时中断网络，因此最后执行 `nmcli connection up` 或直接冷启动两端链路。之后把 NoMachine 的服务器地址改为 `10.43.0.2`。

## 核对命令

~~~bash
# 主机
adb devices -l
ip -brief address show enx02422a000001
nmcli connection show LubanCat-USB
ping -c 2 10.42.0.2

# 板子
adb shell 'ip -brief address show usb0; ip route'
adb shell 'getent ahostsv4 download.nomachine.com | head -1'
~~~

如果 ADB 存在但 RNDIS 不出现，先检查 `/etc/init.d/.usb_config` 和 `/etc/usbdevice.d/rndis.sh`，再冷启动板子。不要删除 `Phone-USB`；它是仍需使用的备用入口。
