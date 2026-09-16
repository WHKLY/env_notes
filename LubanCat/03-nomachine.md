# NoMachine 无头桌面

## 使用入口

主机安装 NoMachine Enterprise Client 10.1.7-1，板子安装 NoMachine 9.9.6-2。通常从桌面快捷方式 `LubanCat-5 V2 (USB)` 打开，也可执行：

~~~bash
/usr/NX/bin/nxplayer --session /home/xanter/Documents/NoMachine/LubanCat-5-V2-USB.nxs
~~~

连接参数为：

| 项目 | 值 |
|---|---|
| 主机 | `10.42.0.2` |
| 端口 / 协议 | TCP 4000 / NX |
| 桌面 | 默认 X session |
| 用户 | `cat` |
| 认证 | 板子系统用户密码 |

若客户端同时发现 `LUBANCAT-5-V2-USB` 和其他名为 `LUBANCAT` 的入口，优先使用固定地址 `10.42.0.2` 的 USB 会话。

## 为什么使用虚拟桌面

拔掉 HDMI 后，板子的物理 Xorg 曾保留一个失效的 720x400 framebuffer。NoMachine 连接到该物理桌面时虽然能显示画面，但鼠标点击坐标不正常。板子现在默认启动到：

~~~text
multi-user.target
~~~

`display-manager` 保持 inactive，让 NoMachine 在连接时创建独立的虚拟 X11 桌面。`nxnode` 在没有活动会话时显示 disabled 属正常现象；`nxserver`、`nxd` 应 enabled，`nxd` 应监听 4000 端口。

## GNOME 软件渲染修复

RK3588 镜像中的专有 Mali EGL 栈依赖物理 DRM 显示器。在 NoMachine 虚拟 X 上，GNOME Shell 会因 `eglInitialize()` 失败而退出，客户端随后提示 `<no available desktops on this server>`。

`/usr/NX/etc/node.cfg` 当前保留默认 Ubuntu 会话，并只给 NoMachine 虚拟桌面加载环境钩子：

~~~text
DefaultDesktopCommand "dbus-launch --exit-with-session gnome-session --session=ubuntu"
VirtualDesktopVariables "/usr/local/bin/nomachine-virtual-env.sh"
~~~

原配置备份为 `/usr/NX/etc/node.cfg.before-mesa-headless`。钩子 `/usr/local/bin/nomachine-virtual-env.sh` 内容为：

~~~sh
#!/bin/sh

# Use Mesa/llvmpipe only inside NoMachine virtual desktops. The RK3588
# proprietary Mali EGL stack requires a physical DRM display and crashes
# GNOME Shell when the board is headless.
echo "LD_LIBRARY_PATH=/lib/aarch64-linux-gnu${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
echo "__GLX_VENDOR_LIBRARY_NAME=mesa"
echo "LIBGL_ALWAYS_SOFTWARE=1"
echo "GALLIUM_DRIVER=llvmpipe"
echo "XDG_SESSION_TYPE=x11"

exit 0
~~~

板子已有 Mesa `swrast_dri.so`、`kms_swrast_dri.so` 和 llvmpipe。这个设置只影响 NoMachine 虚拟桌面，不覆盖板子其他会话的 GPU 环境。

## 状态核对

~~~bash
adb shell '/usr/NX/bin/nxserver --status'
adb shell 'ss -lnt "sport = :4000"'
adb shell 'systemctl get-default; systemctl is-active display-manager || true'
adb shell 'grep -nE "^(DefaultDesktopCommand|VirtualDesktopVariables)" /usr/NX/etc/node.cfg'
~~~

预期默认目标为 `multi-user.target`、显示管理器为 inactive，且 `nxd` 监听 TCP 4000。用户已在拔掉外接显示器后实测：登录、桌面显示、鼠标点击和普通操作均正常。

## 恢复物理本地桌面

若以后需要让板子接显示器后自动出现本地图形登录，可通过 ADB 执行：

~~~bash
adb shell 'systemctl set-default graphical.target; systemctl start display-manager'
~~~

这会恢复物理桌面启动方式。再次改回稳定的无头虚拟桌面：

~~~bash
adb shell 'systemctl set-default multi-user.target; systemctl stop display-manager'
~~~

两种模式都可以修改；切换时应先断开现有 NoMachine 会话，避免客户端继续附着到已停止的桌面。

## 升级备注

板端 NoMachine 从 9.3.7-1 更新到 9.9.6-2 时曾停在 dpkg 配置文件提示，最终保留了当前 `server.cfg.sample` 并完成配置。以后升级后应检查：

- `/usr/NX/etc/node.cfg` 中的 `VirtualDesktopVariables` 是否仍存在；
- `/usr/local/bin/nomachine-virtual-env.sh` 是否可执行；
- 默认启动目标和 `display-manager` 状态是否未被安装脚本改变；
- 新建虚拟桌面能否正常进入和点击。
