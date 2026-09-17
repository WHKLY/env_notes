# ROS1 项目 Docker 构建与 GPU 推理复盘

## 1. 文档目的

本文记录 24ZCold 在本机迁移、容器化和 `catkin_make` 验证中的已知工作流、问题与解决办法，供后续 ROS1 项目复用。

本文只总结环境与构建方法，不改变 24ZCold 的功能代码。后续推理环境按 GPU 优先、后端显式选择的原则设计，不能把 CPU 推理写死在 Dockerfile、依赖文件或启动脚本中。

记录日期：2026-09-18。

## 2. 当前状态快照

- 项目目录：`/home/xanter/Projects/24ZCold`
- 本次项目派生镜像：`xanter/24zcold-ros1:noetic-20260918`，已删除。
- 该派生镜像没有对应容器，因此没有项目容器需要删除。
- 构建文件仍保留在项目中，可审查、修改后重新构建。
- 基础设施镜像 `xanter/ros1-noetic-gazebo11:20260913` 未删除。
- 上游镜像 `osrf/ros:noetic-desktop-full-focal` 未改动。
- 容器 `ros1_noetic` 未改动；它已停止，但仍引用上游镜像，所以 Docker 会把该镜像标记为 `in use`。

删除镜像不等于删除构建方法。只要 Dockerfile、依赖清单、Compose 文件和源代码仍在，就可以重新构建。删除镜像也不会删除 bind mount 指向的宿主机项目文件。

## 3. `osrf/ros:noetic-desktop-full-focal` 的只读检查结果

### 3.1 镜像元数据

| 项目 | 检查结果 |
| --- | --- |
| 标签 | `osrf/ros:noetic-desktop-full-focal` |
| 镜像 ID / 本地摘要 | `sha256:f138c82f326f179e8510eb03f40ee7ccaf16538e5577ab8550af74c4e12616de` |
| 创建时间 | `2025-06-10T00:43:22Z` |
| 系统 / 架构 | `linux/amd64` |
| Docker inspect 报告大小 | `1163119522` 字节，约 1.16 GB |
| ROS 发行版 | `noetic` |
| 基础系统标签 | Ubuntu 20.04 |
| 默认入口 | `/ros_entrypoint.sh` |
| 默认命令 | `bash` |
| 默认用户 | 未显式指定，即 root |

`docker image ls`、`docker image inspect` 和 `docker system df -v` 的大小口径不同，不能直接把它们当作同一数值比较。判断实际磁盘占用时优先查看 `docker system df -v`。

### 3.2 本地可见构建历史

`docker history --no-trunc` 显示的主要层包括：

1. Ubuntu 20.04 基础层；
2. 时区、证书、curl、dirmngr、GnuPG 等基础工具；
3. ROS apt 密钥和 Noetic 软件源；
4. `ros-noetic-ros-core`；
5. `rosdep`、`rosinstall`、`vcstools` 和编译工具；
6. `ros-noetic-ros-base`、robot、desktop、desktop-full 等逐层安装；
7. `/ros_entrypoint.sh`、入口命令和默认 `bash`。

镜像通常不内嵌完整的原始 Dockerfile。`docker history` 能看到层和部分构建命令，但不能保证无损还原原始文件。要实现可复现构建，应同时保留 Dockerfile，并固定基础镜像 digest。

### 3.3 为什么它显示为 `in use`

本机存在以下停止状态容器：

| 项目 | 值 |
| --- | --- |
| 容器名 | `ros1_noetic` |
| 容器 ID | `e03de930def15e501c9e5ac3047269722863ebc2d81063e48444d0373d3acddd` |
| 状态 | `exited`，退出码 0 |
| 镜像 | `osrf/ros:noetic-desktop-full-focal` |
| 网络模式 | `host` |
| bind mount | `/home/xanter/ros1_ws:/root/ros1_ws`，读写 |
| 特权模式 | 关闭 |

停止的容器仍保存镜像引用，因此镜像会显示为 `in use`。只有删除所有引用它的容器后，镜像才不再被这些容器占用。本次按要求没有删除或修改该容器。

只读查看命令：

~~~bash
docker image inspect osrf/ros:noetic-desktop-full-focal
docker history --no-trunc osrf/ros:noetic-desktop-full-focal
docker ps -a --filter ancestor=osrf/ros:noetic-desktop-full-focal
docker inspect ros1_noetic
docker system df -v
~~~

## 4. 已验证的项目迁移与构建工作流

### 4.1 先做静态盘点

在构建前确认：

- catkin 工作空间根目录、`src` 和所有 ROS package；
- launch 文件、world、模型、权重、相机配置和外部脚本；
- 源码中的绝对路径、用户名、设备名、IP、串口和 ROS master 地址；
- Python import、系统库、ROS 包依赖和编译器依赖；
- 是否存在密钥、密码、令牌或私有地址；
- 仿真资源是否完整，缺失项是否会在启动时才暴露。

建议命令：

~~~bash
cd /home/xanter/Projects/24ZCold
find . -maxdepth 3 -type f | sort
rg -n '/home/|/media/|xanter|\.pt|\.pth|\.engine|\.world|/dev/' .
rg -n '^(import|from) ' --glob '*.py' .
~~~

### 4.2 改动前建立 Git 基线

先提交原始迁移版本，再做本机适配。这样路径改动、容器文件和原始业务逻辑可以清晰区分，也便于回退和审查。

提交前必须先检查敏感文件并配置 `.gitignore`。不要因为要建立基线而提交密码、令牌、私钥或大体积运行产物。

### 4.3 固定基础镜像输入

稳定项目应优先使用 digest，而不是只依赖可移动标签：

~~~dockerfile
FROM osrf/ros:noetic-desktop-full-focal@sha256:f138c82f326f179e8510eb03f40ee7ccaf16538e5577ab8550af74c4e12616de
~~~

digest 固定的是基础镜像内容，不等于整个项目环境已经可复现。apt 仓库、pip 依赖、外部下载和未固定版本同样需要管理。

### 4.4 缩小构建上下文

使用 `.dockerignore` 排除 `.git`、`build`、`devel`、日志、缓存、数据集和大模型。只有 Dockerfile 当前步骤真正需要的文件才进入 build context。

这样能减少传输时间、避免缓存被无关文件击穿，也降低意外把敏感文件打进镜像的风险。

### 4.5 将依赖构建进镜像

不要依赖进入一次性容器后手动 `pip install`。容器被删除后，这些安装会一起消失，也无法被其他人稳定复现。

正确做法是：

1. 在 Dockerfile 中安装系统依赖和 Python 依赖；
2. 固定经过验证的版本组合；
3. 重新构建镜像；
4. 用干净容器做 import smoke test。

开发阶段临时安装可以用于定位缺包，但定位完成后必须回写到依赖清单或 Dockerfile。

### 4.6 明确容器内路径契约

catkin 生成的 `build` 和 `devel` 文件可能记录绝对路径。若工作空间从 `/workspace/24zcold_ws` 改到其他容器路径，应在新路径重新执行 `catkin_make`，不能把旧 `build/devel` 当作可移植产物。

运行时 bind mount 的目标路径应和构建时路径保持一致，例如：

~~~bash
--mount type=bind,src=/home/xanter/Projects/24ZCold,dst=/workspace/24zcold_ws
~~~

这是容器实例的运行时挂载，不是镜像内容。新建另一个容器时必须再次声明，除非把配置写进 Compose 并通过 Compose 创建容器。

### 4.7 分层验证

推荐按以下顺序验证，避免把多个问题混在一次启动里：

1. Dockerfile 能完成构建；
2. ROS 环境可 source；
3. Python 依赖可 import；
4. `catkin_make` 成功；
5. `devel/setup.bash` 可 source；
6. launch 文件和资源路径静态检查通过；
7. GPU 能被容器识别；
8. 模型能在 GPU 上做最小推理；
9. 最后才启动完整仿真。

`catkin_make` 只负责编译、生成消息/服务以及生成工作空间环境，不会自动保证 world、权重、外部设备或运行时 Python 资源齐全。

## 5. 本次遇到的问题与处理经验

### 5.1 PyTorch 安装源选择不明确

问题：仅写通用 PyTorch 版本时，pip 可能解析出体积很大的 CUDA 相关 wheel；反过来，使用仅面向 CPU 的安装源又会把 CPU 后端固定进镜像，妨碍后续 GPU 推理。

处理：通用 Python 依赖与 PyTorch 分开安装。PyTorch 的版本和 wheel 索引必须由构建参数显式传入，不提供 CPU 默认值，也不让解析器自行猜测 GPU 变体。

### 5.2 将专用 extra-index 用于全部 pip 依赖

问题：把 PyTorch 专用索引作为整个 requirements 安装过程的 `extra-index-url`，会让 pip 对每个包都查询多个索引，增加延迟，也可能带来依赖混淆风险。

处理：普通依赖从项目规定的主索引安装；PyTorch 单独一步从明确的官方 wheel 索引安装。

### 5.3 Python import 缺失

问题：源码使用了 `geopy`、`pyproj` 等库，但原始依赖声明不完整。

处理：构建前扫描 Python import；镜像完成后执行最小 import smoke test。扫描只能发现显式 import，动态加载和条件分支仍需运行测试覆盖。

### 5.4 `pyOpenSSL` 与 `cryptography` 不兼容

问题：旧 `pyOpenSSL` 与很新的 `cryptography` 组合可能在 import 阶段报错。

本次历史环境验证过的组合是 `pyOpenSSL==23.2.0` 和 `cryptography==41.0.7`。该组合适用于复现本次 Python 3.8 环境，但不是未来项目永久不变的通用答案。新项目应根据 Python、Ubuntu 和上游安全更新重新生成并验证锁文件。

### 5.5 Docker 构建缓存占用空间

问题：删除项目镜像后，BuildKit 缓存可能仍保留，磁盘空间不会全部立即释放。

处理：先用 `docker system df -v` 区分镜像、容器、volume 和 build cache。缓存清理是独立动作，不要把 `docker system prune -a` 当成默认命令；它可能删除其他项目仍有价值的缓存和未使用镜像。

### 5.6 源码中的绝对路径

问题：原项目路径包含旧用户名、移动磁盘路径或固定工作空间位置，在容器中会失效。

处理：仅修改必要的路径入口，优先由环境变量、ROS 参数或启动脚本传入；避免顺手重构业务逻辑。对必须固定的容器内路径形成单一契约，并写进 Compose 和文档。

### 5.7 机密配置混入镜像或仓库

问题：账号、密码、token 或私有连接信息可能出现在脚本中，并被 Git 或 Docker layer 永久记录。

处理：敏感值放入宿主机 `.env` 或只读挂载文件，仓库只保留 `.env.example`；确认 `.env` 已被 `.gitignore` 和 `.dockerignore` 排除。已经写进镜像层的秘密不能靠后续 `RUN rm` 真正清除，必须轮换秘密并重建历史。

### 5.8 仿真资源缺失

问题：缺少 world、模型、权重、相机配置或外部工程时，编译仍可能成功，但完整仿真无法启动。

处理：在启动脚本中做 preflight，逐项报告缺失文件并非零退出。不要静默替换模型、生成空文件或猜测资源路径，因为这会让仿真产生误导性结果。

## 6. 后续项目的 GPU 优先方案

### 6.1 当前宿主机事实

2026-09-18 的只读检查结果：

- GPU：NVIDIA GeForce RTX 5070 Ti Laptop GPU；
- 驱动：595.84；
- CUDA Compute Capability：12.0；
- Docker 当前只列出 `runc` 和 `io.containerd.runc.v2`；
- 未检测到 NVIDIA Container Toolkit 软件包；
- 因此宿主机 GPU/驱动可见，但 Docker GPU 透传尚未配置和验证。

在安装 NVIDIA Container Toolkit 前，`gpus: all` 或 `docker run --gpus all` 不能视为已经可用。安装和配置工具包属于宿主机基础设施变更，应单独实施、记录和验证；不要在 ROS 应用镜像里安装宿主机显卡驱动。

### 6.2 构建参数必须显式选择 GPU wheel

后续 Dockerfile 可采用以下模式。示例故意不提供 CPU 默认值，也不把某个 CUDA wheel 索引永久写死：

~~~dockerfile
ARG PYTORCH_INDEX_URL
ARG TORCH_VERSION
ARG TORCHVISION_VERSION

RUN test -n "$PYTORCH_INDEX_URL" \
 && test -n "$TORCH_VERSION" \
 && test -n "$TORCHVISION_VERSION" \
 && python3 -m pip install --no-cache-dir \
      --index-url "$PYTORCH_INDEX_URL" \
      "torch==$TORCH_VERSION" \
      "torchvision==$TORCHVISION_VERSION"
~~~

构建时由 `.env`、CI 或命令行传入经过兼容性核验的值：

~~~bash
docker build \
  --build-arg PYTORCH_INDEX_URL='https://download.pytorch.org/whl/<cuNNN>' \
  --build-arg TORCH_VERSION='<已验证版本>' \
  --build-arg TORCHVISION_VERSION='<已验证版本>' \
  -t '<项目名>:<版本>' .
~~~

`<cuNNN>` 只是占位符。实施时必须依据当时的 PyTorch 官方兼容矩阵、宿主机驱动和目标 GPU 选择真实值，并记录选择依据。不能从本次旧项目的依赖版本直接推断新项目的 CUDA 组合。

普通 Python 依赖应在另一步安装，避免 PyTorch 专用索引参与全部包的解析：

~~~dockerfile
COPY requirements.txt /tmp/requirements.txt
RUN python3 -m pip install --no-cache-dir -r /tmp/requirements.txt
~~~

### 6.3 Compose 中声明 GPU 运行需求

Compose 示例：

~~~yaml
services:
  ros1:
    build:
      context: .
      args:
        PYTORCH_INDEX_URL: ${PYTORCH_INDEX_URL:?must be set}
        TORCH_VERSION: ${TORCH_VERSION:?must be set}
        TORCHVISION_VERSION: ${TORCHVISION_VERSION:?must be set}
    gpus: all
    environment:
      NVIDIA_VISIBLE_DEVICES: ${NVIDIA_VISIBLE_DEVICES:-all}
      NVIDIA_DRIVER_CAPABILITIES: ${NVIDIA_DRIVER_CAPABILITIES:-compute,utility}
~~~

推理用 CUDA 和 Gazebo/GUI 的图形渲染不是同一个问题。只有确需 GPU 图形渲染时才增加 `graphics` 能力并单独验证 X11/Wayland。`LIBGL_ALWAYS_SOFTWARE=1` 可以作为排障用的软件渲染选项，但不应在 GPU 项目中永久写死。

### 6.4 推理后端显式选择

应用启动时建议要求调用方显式指定后端，例如：

~~~bash
INFERENCE_BACKEND=cuda
~~~

允许的值可定义为 `cuda`、`cpu`，但不要把 `cpu` 作为 Dockerfile 或启动脚本中的隐式固定结果。生产或性能验证模式下，如果指定 `cuda` 而 GPU 不可用，应清楚报错并退出，不要静默回退到 CPU 后继续输出看似正常的结果。

### 6.5 GPU 最小验证

容器运行前先验证透传：

~~~bash
docker run --rm --gpus all '<匹配的 CUDA 基础镜像>' nvidia-smi
~~~

应用镜像内再验证 PyTorch：

~~~bash
python3 - <<'PY'
import torch

print('torch:', torch.__version__)
print('torch CUDA runtime:', torch.version.cuda)
print('CUDA available:', torch.cuda.is_available())
assert torch.cuda.is_available(), '指定了 GPU 推理，但容器内 CUDA 不可用'
print('device:', torch.cuda.get_device_name(0))
x = torch.ones((1024, 1024), device='cuda')
print('sum:', x.sum().item())
PY
~~~

每次生成可交付镜像时，至少记录：

- 基础镜像 tag 和 digest；
- 宿主机 GPU 型号与驱动版本；
- NVIDIA Container Toolkit 版本；
- `torch.__version__` 和 `torch.version.cuda`；
- cuDNN 或 TensorRT 版本（若使用）；
- 模型文件摘要；
- GPU smoke test 结果。

### 6.6 模型与 TensorRT 注意事项

- 原始模型权重应作为可追溯输入保存，并记录摘要；
- TensorRT `.engine` 往往依赖 GPU 架构、TensorRT/CUDA 版本和构建参数，不应默认跨机器复用；
- 在目标 GPU 环境中生成 engine，并保留生成日志和参数；
- 构建镜像时不要无意把大型模型复制进每个 layer；可按交付方式选择独立模型镜像、只读挂载或受控下载缓存。

## 7. 推荐的后续项目文件结构

~~~text
project/
├── Dockerfile
├── compose.yaml
├── .dockerignore
├── .env.example
├── requirements.txt
├── requirements.lock
├── docker/
│   └── ros_entrypoint.sh
├── scripts/
│   ├── build-image.sh
│   ├── check-gpu.sh
│   ├── preflight.sh
│   └── run-sim.sh
└── docs/
    └── build-manifest.md
~~~

原则：版本、路径、GPU 后端和外部资源来源均有唯一入口；脚本做校验并快速失败；业务代码不承担容器环境探测和路径拼接的全部责任。

## 8. 清理镜像与容器的安全顺序

先只读确认精确目标：

~~~bash
docker ps -a --filter ancestor='<镜像名:标签>'
docker image inspect '<镜像名:标签>'
~~~

再删除项目容器和精确镜像：

~~~bash
docker rm '<项目容器名或 ID>'
docker image rm '<镜像名:标签>'
~~~

最后复核：

~~~bash
docker ps -a
docker image ls
docker system df -v
~~~

build cache、volume、基础镜像和停止容器是四类不同对象，应分别确认。不要为了删除一个项目镜像直接执行大范围 prune。

## 9. 新项目执行清单

### 构建前

- [ ] 原始代码已建立 Git 基线，敏感文件已排除；
- [ ] ROS package、Python import、系统依赖和外部资源已盘点；
- [ ] 基础镜像 digest 已记录；
- [ ] 容器内工作空间路径已统一；
- [ ] GPU 型号、驱动和 NVIDIA Container Toolkit 状态已核验；
- [ ] PyTorch、CUDA、cuDNN/TensorRT 兼容组合已按当时官方资料选择；
- [ ] 推理后端通过构建参数或运行参数显式选择，没有写死 CPU；
- [ ] `.dockerignore`、`.gitignore` 和 `.env.example` 已检查。

### 构建后

- [ ] ROS 环境能 source；
- [ ] Python import smoke test 通过；
- [ ] `catkin_make` 通过；
- [ ] GPU 在容器内可见；
- [ ] PyTorch CUDA 张量测试通过；
- [ ] 模型最小推理通过且设备确认为 CUDA；
- [ ] launch、world、模型、权重、相机和外部工程通过 preflight；
- [ ] 完整仿真才开始运行；
- [ ] 构建 manifest 和问题记录已更新。

## 10. 对 24ZCold 当前构建文件的说明

项目现有 Dockerfile 和 Compose 文件用于复现这次已经完成的历史构建与静态验证，其中可能保留当时为旧依赖选择的实现。为了不改变当前项目功能和已验证基线，本次没有改写这些文件。

后续 GPU 项目不应直接复制其中面向 CPU 的 PyTorch 安装方式。应使用本文第 6 节的显式 GPU 构建参数、容器 GPU 透传和 CUDA smoke test，并在真正选定兼容版本后再形成新的项目锁文件。
