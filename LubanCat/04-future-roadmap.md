# LubanCat Future Platform Roadmap

状态：**阶段 0 进行中；Ubuntu 24 迁移尚未开始**

本文规划 LubanCat-5 V2 从当前 Ubuntu 20.04 eMMC 基线，过渡到 Ubuntu 24 SD 卡验证，并在验证通过后决定是否将 Ubuntu 24 写入 eMMC。Ubuntu 20 板端 Git 仓库和首次 push 已完成，环境文档正在进行一致性复核；尚未建立基线标签，也未开始 Ubuntu 24 SD 或 eMMC 操作。

## 总体路线

~~~text
Ubuntu 20.04 eMMC 建立 Git 基线
              |
              v
Ubuntu 24 SD 卡独立试验与兼容性验证
              |
              v
       通过迁移门槛？
          |         |
         否         是
          |         |
保留 Ubuntu 20   备份并刷写 Ubuntu 24 eMMC
                        |
                        v
                重建、复测并建立新基线
~~~

核心原则：GitHub 远端仓库跨系统保留；板子本地仓库、SSH 私钥、软件包和系统配置都属于具体 rootfs，换 SD 卡或刷写 eMMC 后需要重新建立。

## 仓库与身份策略

- 当前 Ubuntu 20.04 eMMC 完成本轮文档复核并建立基线标签前，不调整现有板端文档目录结构。
- 使用同一个私有远端仓库存放各系统的环境记录，但不要提交系统镜像、模型、SDK、数据集、日志或密钥。
- 每个系统实例生成自己的 Deploy Key，绝不复制私钥：
  - `LubanCat Ubuntu20 eMMC`
  - `LubanCat Ubuntu24 SD`
  - `LubanCat Ubuntu24 eMMC`
- Git 操作统一由普通用户 `cat` 完成，不用 root 维护仓库。
- 不建议用 `safe.directory '*'` 绕过所有权问题；新系统应在自己的 `/home/cat` 中重新 clone。

建议在 Ubuntu 20 基线审计完成后，用单独一次提交将仓库逐步整理为：

~~~text
README.md
CHANGELOG.md
systems/
  ubuntu20-emmc/
  ubuntu24-sd/
  ubuntu24-emmc/
guides/
~~~

这是目标结构，不应在当前 Ubuntu 20 基线标签建立前提前重排。

## 阶段 0：冻结 Ubuntu 20.04 eMMC 基线

板端已经安装 Git 2.25.1，以 `cat` 用户初始化 `/home/cat/Documents/env_notes`，并通过专用 Deploy Key 将首次提交推送到私有仓库 `WHKLY/lubancat-env-notes`。文档一致性复核已经完成并经用户确认；完成本次提交和推送后，下一步是建立基线标签。

完成条件：

- 板端文档与实测环境一致；
- 工作区无意外文件，敏感信息未进入历史；
- 本地 `main` 与远端 `origin/main` 指向相同提交；
- GitHub 仓库为 Private，Deploy Key 名称和权限符合预期；
- 冷启动后的 ADB、电脑 USB 供网、手机 USB 备用网络和 NoMachine 均已复核。

审计通过后建立并推送带注释标签：

~~~bash
git tag -a ubuntu20-emmc-baseline -m "Ubuntu 20.04 eMMC baseline"
git push origin ubuntu20-emmc-baseline
~~~

以上命令在**板子的 `cat` 用户终端**执行。标签必须基于已审计的提交，而不是尚未检查的工作区。

## 阶段 1：准备 Ubuntu 24 SD 卡试验

在**电脑上**完成镜像下载、发布方校验和核对以及 SD 卡写入。写卡前再次核对目标设备，避免覆盖电脑磁盘或板子 eMMC。

建议保留以下证据：

- 镜像文件名、下载来源、版本和发布时间；
- 官方 SHA256 与本机计算结果；
- 写卡工具及其版本；
- 当前 Ubuntu 20 Git 提交 ID 和基线标签；
- 必要的板端非 Git 数据备份清单。

首次从 SD 启动后：

1. 确认根文件系统确实位于 SD，而非 eMMC；
2. 创建或确认普通用户 `cat`，并记录 UID/GID；
3. 安装 Git，生成全新的 `LubanCat Ubuntu24 SD` Deploy Key；
4. 将同一私有仓库全新 clone 到 `/home/cat/Documents/env_notes`；
5. 从 `main` 创建临时分支：

~~~bash
git switch -c test/ubuntu24-sd
~~~

该命令在**Ubuntu 24 SD 系统的板子终端**执行。测试分支只用于逐步记录试验，最终结论整理后再合并回 `main`。

## 阶段 2：Ubuntu 24 SD 验证矩阵

### 系统与启动

- 系统版本、内核、Bootloader、设备树和实际根文件系统；
- eMMC、SD、USB 设备枚举及挂载行为；
- 用户 UID/GID、时区、主机名、休眠和重启行为；
- `/etc`、`/usr` 等目录权限是否正常，旧系统发现的异常是否仍存在。

### USB 与网络

- 唯一 Type-C 口能否继续提供 ADB + RNDIS 复合功能；
- Gadget 两端 MAC、`10.42.0.0/24` 地址和接口识别是否稳定；
- 电脑 NetworkManager `shared` 供网、DNS、默认路由和重启恢复；
- 手机 USB 共享仍可使用，并保持为备用链路；
- SSH、FTP、ADB TCP、ZeroTier、UFW 和监听端口是否符合预期。

### 图形桌面

- NoMachine 服务、认证和无外接显示器虚拟桌面；
- GNOME 会话、Mesa/Mali EGL、OpenGL/OpenCL 和硬件加速状态；
- 键鼠操作、分辨率、剪贴板、重连和冷启动恢复。

### NPU、GPU 与媒体栈

- RKNN 驱动、Runtime、Toolkit/Lite 的版本兼容性；
- RGA、Mali、OpenCL、OpenCV 和 GStreamer；
- 已有 YOLOv10、YOLOv8 Pose、ResNet、LPRNet 等自建内容的推理正确性；
- 性能、内存、温度、功耗和长时间稳定性。

模型、测试图片和构建产物仍放在 Git 仓库外。文档只记录其路径、来源、校验和、版本、命令和结果摘要。

## 阶段 3：eMMC 迁移门槛

只有同时满足以下条件，才考虑将 Ubuntu 24 写入 eMMC：

- 阶段 2 的关键功能均验证通过，或已明确接受差异；
- SD 测试文档已经 commit、push，并合并到 `main`；
- 已建立并推送 `ubuntu24-sd-baseline` 标签；
- 当前 Ubuntu 20 基线和重要非 Git 数据至少各有一份板外备份；
- 已记录可用的 Ubuntu 20 恢复镜像、校验和及刷写方法；
- 已明确发生断电、刷写失败或 Ubuntu 24 不可用时的回退方案。

如果 ADB、USB 供网、NoMachine、NPU/媒体栈或启动稳定性中的任何关键项未通过，继续使用 Ubuntu 20 eMMC，把 Ubuntu 24 限制在 SD 测试环境。

## 阶段 4：刷写 Ubuntu 24 eMMC 前

在**板子上**确认两个系统的文档都已提交；在**电脑或 GitHub 页面上**确认提交、分支和标签确实存在于远端。不要只依赖“push 命令没有报错”。

至少记录：

- `ubuntu20-emmc-baseline` 和 `ubuntu24-sd-baseline` 对应的完整提交 ID；
- Ubuntu 24 eMMC 镜像文件名、来源和 SHA256；
- 板外备份的位置与恢复步骤；
- 当前 Deploy Key 列表，但不记录任何私钥内容。

刷写会覆盖 eMMC 上未备份的数据。GitHub 仓库只能恢复已提交文档，不能恢复操作系统、软件包、模型、构建目录或用户数据。

## 阶段 5：Ubuntu 24 eMMC 建立新基线

刷写完成后把它当作新系统，不直接复用 SD 或旧 eMMC 的私钥：

1. 确认系统从 eMMC 启动；
2. 创建或确认 `cat` 用户；
3. 安装 Git，并生成 `LubanCat Ubuntu24 eMMC` Deploy Key；
4. 全新 clone 环境仓库；
5. 重新配置并验证 USB Gadget、网络、NoMachine 和计算栈；
6. 将结果整理并合并到 `main`；
7. 审计通过后建立 `ubuntu24-emmc-baseline` 标签。

新系统稳定且远端同步确认无误后，才在 GitHub 删除已经不再使用的旧 Deploy Key。删除公钥会立即阻止对应旧系统访问仓库，但不会删除仓库历史。

## Git 本身受到的影响

- 更换 Ubuntu 或启动介质不会改变 GitHub 远端的提交、分支和标签。
- 本地 `.git`、Git 配置、SSH 私钥和 `known_hosts` 位于 rootfs；换系统后默认不会自动出现。
- 不同 Ubuntu 上的 Git 版本通常可读写同一仓库；采用普通 Markdown 和标准 Git 功能即可。
- 同一份文档可能记录多个系统状态，提交信息应明确写出 `ubuntu20-emmc`、`ubuntu24-sd` 或 `ubuntu24-emmc`。
- 大文件不应纳入这个环境仓库，因此当前没有引入 Git LFS 的必要。

## 建议的分支与标签

| 用途 | 名称 | 生命周期 |
|---|---|---|
| 可信总记录 | `main` | 长期 |
| Ubuntu 24 SD 试验 | `test/ubuntu24-sd` | 试验期间，合并后可删除 |
| Ubuntu 20 eMMC 基线 | `ubuntu20-emmc-baseline` | 长期标签 |
| Ubuntu 24 SD 基线 | `ubuntu24-sd-baseline` | 长期标签 |
| Ubuntu 24 eMMC 基线 | `ubuntu24-emmc-baseline` | 长期标签 |

不建议为每个系统永久维护互不合并的分支；那会让通用说明和安全修订逐渐分叉。`main` 应保留所有系统的可追溯记录，短期试验分支只承担尚未验证的工作。

## 后续审计节点

在以下节点让 Codex 做只读检查：

1. Ubuntu 20 eMMC 首次 push 后及基线标签建立前（当前节点）；
2. Ubuntu 24 SD 首次完整验证后；
3. 决定刷写 eMMC 前；
4. Ubuntu 24 eMMC 冷启动复测后。

审计范围包括本机文档、板端文档、工作区状态、分支与标签、远端地址、提交对应关系和 GitHub 仓库元数据；不会读取或展示私钥、密码、Token。
