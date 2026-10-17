# Docker 入门核心知识梳理

> **一句话定义**：Docker 是一个把「应用 + 它需要的一切运行环境」打包成一个标准单元（镜像），并能在任何装了 Docker 的机器上以隔离的进程（容器）运行起来的工具。
>
> **适合谁读**：完全没接触过 Docker 的开发者 / 运维初学者。不需要 Linux 内核基础，但读到第 3 章时请耐心——那是"理解"和"背命令"的分水岭。
>
> **怎么读**：第 1~2 章建立直觉（1 小时）→ 第 3 章理解原理（1~2 小时，可先跳读，实操后再回来）→ 第 4~6 章动手（2 小时）→ 第 7~11 章当手册反复查。

---

## 目录

1. [Docker 到底是什么](#1-docker-到底是什么)
2. [核心概念：先建立心智模型](#2-核心概念先建立心智模型)
3. [原理：Docker 是怎么做到的](#3-原理docker-是怎么做到的)
4. [安装与环境准备](#4-安装与环境准备)
5. [上手实操：从 0 到跑起来](#5-上手实操从-0-到跑起来)
6. [命令速查表](#6-命令速查表)
7. [Dockerfile 精讲](#7-dockerfile-精讲)
8. [数据持久化：卷与挂载](#8-数据持久化卷与挂载)
9. [网络](#9-网络)
10. [Docker Compose：多容器编排](#10-docker-compose多容器编排)
11. [最佳实践清单](#11-最佳实践清单)
12. [常见报错与坑](#12-常见报错与坑)
13. [术语中英对照表](#13-术语中英对照表)
14. [7 天学习路线](#14-7-天学习路线)
15. [自测 12 问（附答案）](#15-自测-12-问附答案)
16. [延伸阅读](#16-延伸阅读)

---

## 1. Docker 到底是什么

### 1.1 一个类比你就能懂

想象你要把一套乐高作品寄给朋友。

- **没有集装箱的世界**：你把零件拆散、混在箱子里，附一张说明"需要 XX 型号的桌子、YY 牌子的胶水"。朋友收到后要么缺件，要么他的桌子上摆不下，要么胶水型号不对。这就是**传统部署**——依赖操作系统版本、库版本、环境变量、目录结构，全靠文档和运气。
- **集装箱的世界**：你把作品连同它需要的桌面一起装进标准集装箱。集装箱的尺寸是全球统一的，任何货轮、卡车、吊机都能搬运它，**不需要知道里面装的是什么**。这就是 **Docker**——它定义了一个"标准箱子"（镜像），任何支持这个标准的地方都能运行它。

所以 Docker 常被叫做 **"软件交付的集装箱标准"**。

### 1.2 官方定义拆解

Docker 官方对它的定位是 **容器化平台（containerization platform）**。拆成三个动作：

| 动作 | 对应命令 | 干了什么 |
|---|---|---|
| **打包（Build）** | `docker build` | 按 Dockerfile 把应用和依赖"烤"成一个只读镜像 |
| **分发（Ship）** | `docker push` / `docker pull` | 把镜像推到仓库，像 git 推代码一样，任何机器都能拉下来 |
| **运行（Run）** | `docker run` | 把镜像启动成一个隔离的进程（容器） |

一句话：**Build once, run anywhere**（构建一次，到处运行）。

### 1.3 它到底解决了什么痛点

| 痛点 | 没有 Docker 时的样子 | 有 Docker 之后 |
|---|---|---|
| **"在我机器上能跑"** | 开发 Win11 + Python 3.12，服务器 Ubuntu + Python 3.8，`SyntaxError` | 开发和生产都用同一个镜像，环境完全一致 |
| **依赖冲突** | 项目 A 要 Node 16，项目 B 要 Node 20，装来装去互相打架 | 每个容器自带自己的 Node，互不干扰 |
| **部署靠文档** | "先装 nginx，再改这个配置，再 `systemctl restart`……" | `docker run` 一条命令，配置写在镜像里 |
| **环境恢复慢** | 新同事配环境配两天 | `docker compose up` 五分钟 |
| **资源浪费** | 一个服务一台虚拟机，8 核 16G 只用 5% | 一台机器跑几十个容器，共享内核开销极小 |
| **回滚困难** | 出问题了不知道改了什么，回滚全靠备份 | 镜像有 tag，`docker run image:v1.2` 就是回滚 |

### 1.4 最关键的一张表：Docker ≠ 虚拟机

初学者最大的误解是把容器当成"轻量虚拟机"。**不是。** 这是全书最重要的一张表：

| 对比维度 | 虚拟机（VM） | 容器（Container） |
|---|---|---|
| **隔离的是什么** | 整套硬件（虚拟 CPU、内存、磁盘、网卡） | 进程的"视野"（看到哪些进程、文件、网络） |
| **有没有独立内核** | **有**，每个 VM 跑一个完整的 Guest OS | **没有**，共享宿主机内核 |
| **启动时间** | 几十秒到几分钟（要引导操作系统） | 毫秒到秒级（就是启动一个进程） |
| **体积** | GB 级（含完整 OS） | MB 级（只含应用 + 依赖） |
| **一台机器能跑多少** | 几个到十几个 | 几十到几百个 |
| **隔离强度** | 强（硬件级，逃逸极难） | 较弱（内核共享，逃逸面更大） |
| **能否跑不同内核** | 能（Windows 里跑 Linux、跑 BSD） | 不能（Linux 宿主机只能跑 Linux 内核的容器） |
| **典型代表** | VMware、VirtualBox、Hyper-V、KVM | Docker、Podman、containerd |

> **一句话记住**：虚拟机虚拟的是**硬件**，容器隔离的是**进程**。
>
> 容器里 `top` 看到的第一个进程（PID 1）就是你的应用本身，而不是 `systemd`。很多"为什么容器里没有 `systemctl`"的困惑都源于这一点。

### 1.5 Docker 不擅长什么（别用错地方）

- ❌ **不是虚拟机**：不能在里面装 Windows、不能换内核、不能当 VPS 卖。
- ❌ **不适合强隔离场景**：多租户不可信代码、需要内核级安全的场景，用 VM 或 gVisor/Kata 这类安全容器。
- ⚠️ **数据库可以跑，但要懂行**：数据必须落盘到卷（第 8 章），生产环境要有人管备份、IO、主从。
- ⚠️ **单机 ≠ 集群**：Docker 只管一台机器上的容器。要跨机器调度、自动扩缩容、故障自愈，那是 **Kubernetes** 的活（Docker 是它的地基之一）。
- ⚠️ **GUI 桌面应用**：技术上能跑（Linux 容器 + X11 转发），但体验差，不是它的主场。

---

## 2. 核心概念：先建立心智模型

### 2.1 六件套总览

先记住这六个词，后面所有命令都是它们的排列组合。

| 概念 | 一句话解释 | 类比 |
|---|---|---|
| **镜像 Image** | 只读的模板，包含文件系统 + 启动命令 | 类（Class）/ 光盘 ISO / 菜谱 |
| **容器 Container** | 镜像运行起来的实例，是一个被隔离的进程 | 对象（Object）/ 运行中的程序 / 做好的菜 |
| **仓库 Registry** | 存放和分发镜像的服务 | GitHub / 应用商店 |
| **Dockerfile** | 描述"如何构建镜像"的文本文件 | 源码 / 构建脚本 |
| **数据卷 Volume** | 宿主机上的一块持久化存储，独立于容器生命周期 | 外接硬盘 / U 盘 |
| **网络 Network** | 容器之间、容器与外界通信的虚拟网络 | 局域网 / 交换机 |

### 2.2 镜像（Image）

- **本质**：一堆**只读的层（layer）**叠起来 + 一份元数据（配置）。
- **分层**：每一条构建指令产生一层。比如 `FROM ubuntu` 一层、`RUN apt install nginx` 一层、`COPY app.py /app` 一层。
- **为什么分层**：层可以被多个镜像**共享**。你拉 10 个都基于 `ubuntu:24.04` 的镜像，硬盘上只存一份 ubuntu 层。这是 Docker 省空间的核心机制。
- **只读**：镜像一旦构建完成就不可修改。想改？重新 build 一个新的。
- **命名规则**：

```
[registry地址/][命名空间/]仓库名[:标签]
        ↓              ↓        ↓      ↓
   docker.io     library    nginx   1.27-alpine
```

  - `nginx` 是 `docker.io/library/nginx:latest` 的简写
  - `:latest` 是默认标签，**生产环境永远不要用它**（它是个会变的指针，不是"最新版"的保证）
  - 更精确的定位是 **digest**：`nginx@sha256:abc123...`，内容哈希，永不改变。生产可用来做完全可复现的部署。

### 2.3 容器（Container）

这是全篇最需要扭转认知的一点：

> **容器 = 镜像的只读层 + 一个薄薄的可写层 + 一组被内核隔离起来的进程。**

所以：

- 容器**不是**一个迷你操作系统，它是**一个进程**（可能带几个子进程）。
- 容器"启动快"是因为它不引导系统，只是 `fork + exec` 一个进程。
- 容器"用完就删"没关系，因为状态应该放在卷里，而不是容器里。
- 容器**删掉后，可写层里的数据就没了**——这是新手最常见的"我的数据去哪了"。
- 容器里的 `root` 默认就是宿主机上带 capabilities 的 root（除非你用 `--user` 和 userns-remap），所以**别在容器里乱来**。

### 2.4 仓库（Registry）

- **Registry** 是服务端（如 Docker Hub、Harbor、云厂商的容器镜像服务）。
- **Repository** 是里面某个镜像的所有版本集合（如 `nginx` 仓库下的 `1.25`、`1.27` 标签）。
- 常用公共仓库：`docker.io`（Docker Hub）、`ghcr.io`（GitHub）、`quay.io`、`registry.k8s.io`。
- 公司内部一般搭私有 Registry，存放业务镜像。

### 2.5 一张图串起来

```
   你写的代码                 Dockerfile
       │                         │
       └──────────┬──────────────┘
                  ▼
          docker build  ──►  【镜像 Image】（只读、分层、可打 tag）
                  │                    │
                  │                    ├── docker push ──► 【仓库 Registry】
                  │                    │                        │
                  │                    │                  docker pull
                  ▼                    ▼                        ▼
          docker run / create ──► 【容器 Container】（镜像 + 可写层 + 隔离进程）
                                       │
                        ┌──────────────┼──────────────┐
                        ▼              ▼              ▼
                   【数据卷】      【网络】      【日志/资源限制】
```

---

## 3. 原理：Docker 是怎么做到的

> 这一章回答："凭什么一个进程能被'关'起来，看不到别的进程，还只能用 512MB 内存？"
>
> 答案：Docker **自己并没有发明隔离技术**。它只是把 Linux 内核里三样早就存在的东西（Namespaces、Cgroups、UnionFS）**组合起来，做成了好用的产品**。**Docker 的价值在产品化，不在底层技术发明**——这句话能帮你去掉一半神秘感。

### 3.1 总体架构

```
┌──────────────────────────────────────────────────────────┐
│  客户端 Client                                            │
│  docker CLI  ──  docker compose  ──  docker buildx        │
└───────────────────────┬──────────────────────────────────┘
                        │  REST API（默认 /var/run/docker.sock）
                        ▼
┌──────────────────────────────────────────────────────────┐
│  Docker 守护进程 dockerd                                  │
│  负责：镜像管理、容器生命周期、网络、卷、API               │
└───────────────────────┬──────────────────────────────────┘
                        │  gRPC
                        ▼
┌──────────────────────────────────────────────────────────┐
│  containerd（业界通用的容器运行时，CNCF 项目）            │
│  负责：镜像拉取/解包、容器创建、任务管理                   │
└───────────────────────┬──────────────────────────────────┘
                        │  OCI Runtime Spec
                        ▼
┌──────────────────────────────────────────────────────────┐
│  runc（底层运行时）                                       │
│  真正干活的人：调用 clone/unshare、pivot_root、            │
│  设置 cgroups、execve 应用进程，然后自己退出              │
└───────────────────────┬──────────────────────────────────┘
                        ▼
              【 你的应用进程，就是容器 】
```

要点：

- **CLI 和守护进程是分离的**。`docker` 命令只是发 HTTP 请求，所以你能用 `DOCKER_HOST=tcp://远程:2375` 操作远程 Docker（也所以永远别把 2375 裸奔在公网）。
- **Docker 引擎已经模块化**：`dockerd → containerd → runc`。这也是 Kubernetes 后来能扔掉 dockerd、直接用 containerd 的原因（CRI 接口）。
- **OCI 标准**：镜像格式（image-spec）和运行时（runtime-spec）都标准化了。所以 Docker 构建的镜像能在 Podman、containerd、K8s 里跑。**镜像不是 Docker 专属的。**

### 3.2 三大基石之一：Namespaces —— "看不见"的隔离

Linux 的 namespace 能把一组进程的**全局资源视图**切开，让它们各自以为自己是独生子。

| Namespace | 隔离了什么 | 容器里的直观效果 |
|---|---|---|
| **PID** | 进程号 | 容器里 `ps aux` 只看到自己的进程，且自己是 PID 1 |
| **NET** | 网络设备、IP、端口、路由表 | 每个容器有自己的 `eth0` 和独立端口，`80` 端口互不冲突 |
| **MNT** | 挂载点 / 文件系统树 | 容器看到的是镜像里的 `/`，看不到宿主机目录（除非挂载） |
| **UTS** | 主机名和域名 | 容器有自己的 `hostname` |
| **IPC** | 信号量、共享内存、消息队列 | 容器间无法用共享内存直接通信 |
| **USER** | 用户和用户组 ID 映射 | 容器里的 root 可以映射成宿主机的普通用户，降低逃逸风险 |
| **CGROUP** | cgroup 根目录视图 | 容器只能看到自己的资源限制 |
| **TIME**（较新） | 系统时钟（`CLOCK_MONOTONIC` 等） | 可用于时间旅行测试 |

**亲手验证**（Linux）：

```bash
# 看某个进程属于哪些 namespace
ls -l /proc/<pid>/ns/
# 在一个新的隔离环境里跑 shell（不装 Docker 也能体验）
sudo unshare --pid --net --mount --uts --fork --mount-proc /bin/bash
```

`unshare` 这个系统调用就是 Docker 隔离的起点。

### 3.3 三大基石之二：Cgroups —— "用不了那么多"的限制

Namespace 解决"看得见 / 看不见"，**不解决"能用多少"**。如果容器里跑个死循环吃满内存，整台机器都会被拖垮。这就是 Cgroups（control groups）的职责。

| 限制维度 | Docker 参数 | 说明 |
|---|---|---|
| CPU 配额 | `--cpus=1.5` | 最多用 1.5 个核（时间片配额） |
| CPU 权重 | `--cpu-shares=512` | 竞争时的相对优先级，不是硬限制 |
| 内存 | `--memory=512m` | 超了会被 OOM Killer 杀掉（容器退出码 **137**） |
| 内存+交换 | `--memory-swap=1g` | 总内存 + swap 上限 |
| 块设备 IO | `--device-read-bps` / `--blkio-weight` | 限制磁盘读写速率 |
| 进程数 | `--pids-limit=100` | 防 fork 炸弹 |

```bash
docker run -d --name limited --memory=256m --cpus=0.5 nginx
docker stats limited          # 实时看 CPU / 内存 / 网络 / IO
```

**两种退出码请务必记住**：

- `137` = `128 + 9`（SIGKILL）：通常是被 OOM 杀了，或者 `docker stop` 超时后被强杀。
- `143` = `128 + 15`（SIGTERM）：被正常 `stop` 终止。这也解释了为什么"容器优雅停机"要求你的应用能处理 SIGTERM。

### 3.4 三大基石之三：UnionFS / OverlayFS —— 分层文件系统

镜像的"分层"不是逻辑概念，是**真实的文件系统技术**。OverlayFS 把多个只读目录（lowerdir）和一个可写目录（upperdir）**叠加**成一个合并视图（merged）。

```
        容器看到的 /（merged，合并视图，看起来是一整个文件系统）
                              ▲
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
   ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
   │ 可写层       │      │ 镜像层 3     │      │ 镜像层 1     │
   │ (upperdir)  │      │ COPY app.py │      │ FROM ubuntu │
   │ 读+写        │      │ (只读)       │      │ (只读)       │
   └─────────────┘      └─────────────┘      └─────────────┘
        ▲                     ▲                     ▲
        └────────── 多个容器共享下面所有只读层 ──────┘
                    （每起一个容器，只新增一个空的可写层）
```

**写时复制（Copy-on-Write）** 的完整流程，读三遍：

1. **读一个文件**：从最上层往下找，谁先有就用谁 → 所以永远不会去修改底层镜像。
2. **改一个已存在的文件**：先把该文件从只读层**复制**到可写层，再改副本。原镜像纹丝不动。
3. **新建文件**：直接写在可写层。
4. **删一个文件**：在可写层打一个"白障（whiteout）"标记，遮住下层，实际不删。

这解释了三件事：

- ✅ **为什么镜像能被几十个容器共享**：只读层一份，各容器只加一个薄可写层。
- ✅ **为什么容器删除即"失忆"**：可写层随容器一起被删。→ 数据要放**卷**（第 8 章）。
- ⚠️ **为什么容器里大量写文件很慢**：每改一个文件都要先复制一份。数据库、日志、编译产物请挂载出去。

> 默认存储驱动是 **overlay2**。老资料里的 `aufs`、`devicemapper` 已是历史。`docker info | grep -i storage` 可以看当前驱动。

### 3.5 完整链路：`docker run nginx` 到底发生了什么

1. **CLI 解析**命令，转成 HTTP 请求发给 `dockerd`（走 `unix:///var/run/docker.sock`）。
2. **检查本地**有没有 `nginx:latest` 这个镜像；没有就去 Registry 拉。
3. **拉取镜像**：按层下载（layer 有 digest，已存在的层直接跳过），解包到 `/var/lib/docker/overlay2/`。
4. **创建容器**：在镜像所有只读层之上，新建一个空的可写层；生成一份 `config.v2.json`（记录环境变量、挂载、端口等）。
5. **准备隔离环境**（containerd → runc 接手）：
   - 用 `clone()` 创建新进程，同时带上新的 PID / NET / MNT / UTS / IPC namespace；
   - 用 `pivot_root` 把根目录切换到叠加好的合并目录（容器里的 `/`）；
   - 设置 cgroups 的 CPU / 内存限制；
   - 配置 veth 网卡对、分配 IP、写 iptables 规则（做端口映射 `-p`）。
6. **execve 启动**：执行镜像的 `ENTRYPOINT` / `CMD`，进程成为容器里的 PID 1。**runc 自己退出，容器就是那个进程。**
7. **前台进程退出 = 容器退出**（状态变成 `Exited`）。这是"容器跑起来就退出"的根因，见第 12 章。

### 3.6 镜像构建原理：分层缓存

Dockerfile 每条指令 = 一层。构建时 Docker 会检查该指令 + 上层内容是否与缓存一致，一致就直接复用缓存层。

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .        # ← 只要依赖清单没变，这一层就命中缓存
RUN pip install -r requirements.txt   # ← 所以这步也不用重跑（关键！）
COPY . .                       # ← 代码天天改，放在最后
CMD ["python", "app.py"]
```

**反过来写就是灾难**：

```dockerfile
COPY . .                       # 改一行代码 → 这一层变了
RUN pip install -r requirements.txt   # → 缓存失效 → 每次 build 重装所有依赖，几分钟
```

这就是"**把变化频率低的放前面，变化频率高的放后面**"这条铁律的由来。

### 3.7 为什么 Windows / macOS 上要先装个 Linux 虚拟机

回顾 1.4 节：**容器共享宿主机内核**。所以 Linux 容器必须跑在 Linux 内核上。

- **Linux 服务器**：原生就是 Linux 内核 → Docker 直接跑，无额外开销。
- **Windows / macOS**：系统内核不是 Linux → Docker Desktop 会在后台起一台**轻量 Linux 虚拟机**（Windows 用 WSL2，macOS 用 Apple Virtualization / HyperKit），容器实际跑在那台 VM 里。

这解释了一堆"Windows 上才有的怪现象"：

| 现象 | 原因 |
|---|---|
| 挂载宿主机目录（bind mount）读写很慢 | 跨 VM 边界的文件同步（WSL2 里 `/mnt/c/...` 尤其慢） |
| 建议把代码放在 WSL 的文件系统里 | `\\wsl$\Ubuntu\home\...` 内部路径性能好得多 |
| 文件权限出现奇怪的 uid/gid | Windows 与 Linux 权限模型不同 |
| Docker Desktop 很吃内存 | 它开着 VM；可在设置里限制 CPU/内存上限 |

---

## 4. 安装与环境准备

### 4.1 选哪种

| 场景 | 推荐方案 |
|---|---|
| Windows 10/11（含家庭版） | **WSL2 + Docker Desktop** |
| macOS（Intel / Apple Silicon） | Docker Desktop，或轻量替代 Colima |
| Linux 服务器（生产） | 直接装 Docker Engine（无 Desktop） |
| 不想用 Docker 官方商业条款 | Podman（命令几乎兼容 `podman` ≈ `docker`） |

### 4.2 Windows 11 完整步骤（针对你当前这台机器）

你这台机器目前 **Docker 未安装、WSL 也未安装**，按下面顺序来：

**① 确认 BIOS 开启了虚拟化**

任务管理器 → 性能 → CPU → 看"虚拟化"是否为"已启用"。若是"已禁用"，进 BIOS 打开 `Intel VT-x` / `AMD-V` / `SVM`。

**② 安装 WSL2（管理员 PowerShell）**

```powershell
wsl --install            # 默认装 Ubuntu，并启用所需 Windows 功能
# 如果上面报错，手动来：
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
# 然后重启电脑
wsl --set-default-version 2
wsl --update
```

重启后设置 Linux 用户名和密码。验证：

```powershell
wsl -l -v      # 应显示 VERSION = 2
```

**③ 安装 Docker Desktop**

下载 <https://www.docker.com/products/docker-desktop/> → 双击安装 → 安装时勾选 **"Use WSL 2 instead of Hyper-V"** → 重启 → 首次启动同意协议。

在 **Settings → General** 里确认勾选了 *Use the WSL 2 based engine*；在 **Settings → Resources → WSL Integration** 里勾选你的发行版（这样 WSL 里也能直接用 `docker`）。

**④ 验证安装**

```powershell
docker --version
docker run hello-world
docker compose version
```

看到 `Hello from Docker!` 就成功了。

> **个人使用免费**：Docker Desktop 对个人、教育、小型企业（<250 员工且年收入 <1000 万美元）免费；更大规模商用需要订阅。服务器上装 Docker Engine（Apache-2.0）则完全免费。

### 4.3 Linux（Ubuntu/Debian）步骤

```bash
# 官方一键脚本（测试/个人环境方便；生产建议用 apt 仓库方式）
curl -fsSL https://get.docker.com | sudo sh -s -- --mirror Aliyun

# 免 sudo（重新登录生效）
sudo usermod -aG docker $USER

# 开机自启
sudo systemctl enable --now docker

# 验证
docker run --rm hello-world
```

### 4.4 国内拉镜像慢：配置镜像加速

编辑 `/etc/docker/daemon.json`（Docker Desktop 则在 Settings → Docker Engine 里改同一份 JSON）：

```json
{
  "registry-mirrors": ["https://<你的加速地址>"],
  "log-driver": "json-file",
  "log-opts": { "max-size": "100m", "max-file": "3" }
}
```

改完重启：`sudo systemctl restart docker`。加速地址请以**云厂商当前文档**为准（这类地址变动频繁，本文不写死）。顺手加的 `log-opts` 很重要——否则容器日志会把磁盘写满（第 12 章）。

### 4.5 装好后的健康检查清单

```bash
docker version            # 客户端 + 服务端都通了吗
docker info               # 存储驱动、cgroup 版本、镜像加速是否生效
docker run --rm hello-world
docker run --rm -it alpine sh -c 'echo ok'   # 能跑 Linux 容器
```

---

## 5. 上手实操：从 0 到跑起来

> 全部命令在 PowerShell 里可直接跑。注意 Windows 路径挂载要用 `${PWD}`（PowerShell）/ `%cd%`（CMD）/ `$(pwd)`（Git Bash）。

### 5.1 第一个容器：看懂"用完即弃"

```bash
docker run --rm hello-world        # --rm：退出后自动删除容器
docker run --rm alpine echo "hi"   # 在 alpine 镜像里执行 echo
```

`docker run` 的完整语义其实是三步：**pull（如果没有）→ create → start**。

### 5.2 跑一个 Web 服务（端口映射第一次登场）

```bash
docker run -d --name my-nginx -p 8080:80 nginx
```

- `-d` 后台运行（detach）
- `--name my-nginx` 给容器起名，否则是随机名 `lucid_einstein`
- `-p 8080:80` **宿主机端口:容器端口**（方向别记反）
- `nginx` 镜像默认前台跑 nginx，所以容器不会退出

打开 <http://localhost:8080> 应该看到 "Welcome to nginx!"。

```bash
docker ps                    # 只列运行中的
docker ps -a                 # 包括已退出的
docker logs -f my-nginx      # 跟踪日志（-f 类似 tail -f）
docker stop my-nginx         # 停止（发 SIGTERM，10 秒后 SIGKILL）
docker rm my-nginx           # 删除容器（必须先停，或 -f 强删）
```

### 5.3 进容器里看看（会颠覆你的直觉）

```bash
docker run -it --rm ubuntu:24.04 bash
# 进入后执行：
ps aux          # 只有 bash 和 ps，没有 systemd、没有 cron、没有内核线程
hostname        # 随机容器 ID
ip addr         # 有自己独立的 eth0 和 IP
cat /etc/hostname
exit
```

再验证一次"容器就是宿主机上的进程"（Linux 宿主机）：

```bash
docker run -d --name t nginx
ps aux | grep nginx     # 直接在宿主机 PS 里能看到 nginx 进程
```

`-it` = `-i`（保持 stdin 打开）+ `-t`（分配伪终端）。**没有 `-it` 你就没法交互**；反过来，想让容器跑在后台就别给 `-t`。

### 5.4 写自己的第一个镜像

新建一个目录 `hello-docker`，放入三个文件：

**`app.py`**

```python
from flask import Flask
import os, socket

app = Flask(__name__)

@app.get("/")
def index():
    return {
        "message": "Hello from Docker!",
        "hostname": socket.gethostname(),   # 容器 ID，用于验证负载均衡
        "version": os.getenv("APP_VERSION", "dev"),
    }

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)      # 必须 0.0.0.0，不能 127.0.0.1
```

**`requirements.txt`**

```
flask==3.0.3
```

**`Dockerfile`**

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 8000
CMD ["python", "app.py"]
```

构建并运行：

```bash
docker build -t hello-docker:1.0 .
docker images                      # 看刚构建的镜像有多大
docker run -d --name hello -p 8000:8000 -e APP_VERSION=1.0 hello-docker:1.0
docker logs hello
curl http://localhost:8000
docker exec -it hello bash         # 进运行中的容器（不是新容器！）
```

两个新手必踩的坑，这里都埋了：

- **`host="0.0.0.0"`**：如果写 `127.0.0.1`，服务只监听容器内的回环地址，`-p` 转发不进来，外部访问必然失败。
- **`PYTHONUNBUFFERED=1`**：否则 Python 输出被缓冲，`docker logs` 里半天看不到东西，容易误判"程序没起来"。

### 5.5 数据卷：让数据活过容器

用 Nginx 挂一个自己的首页：

```bash
mkdir html
'<h1>Hello Volume</h1>' > html/index.html

docker run -d --name web -p 8081:80 \
  -v "${PWD}/html:/usr/share/nginx/html:ro" \
  nginx
curl http://localhost:8081          # 输出 Hello Volume
```

- `-v 宿主机路径:容器路径:ro` → **绑定挂载（bind mount）**，直接映射宿主机目录，改文件立刻生效（开发时最常用）。
- `-v 卷名:容器路径` → **命名卷（named volume）**，由 Docker 管理，适合数据库。

```bash
docker volume create pgdata
docker run -d --name pg -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data postgres:16
docker rm -f pg
docker run -d --name pg2 -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data postgres:16   # 数据还在！
```

### 5.6 Compose：一次起多个容器

**`compose.yaml`**（放在项目根目录）

```yaml
services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      APP_VERSION: "1.0"
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: app
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  pgdata:
```

```bash
docker compose up -d --build     # 构建 + 后台启动全部服务
docker compose ps
docker compose logs -f web
docker compose down              # 停止并删除容器/网络（保留卷）
docker compose down -v           # 连卷一起删（数据没了，慎用）
```

### 5.7 收尾：清理

```bash
docker rm -f $(docker ps -aq)             # 删除所有容器（Linux/macOS/Git Bash）
docker container prune -f                 # 删除所有已停止容器
docker image prune -a                     # 删除未被使用的镜像（慎用）
docker system df                          # 先看看空间被谁占了
docker system prune -a --volumes          # 核弹级清理，会删数据卷！
```

PowerShell 版本（不支持 `$(...)` 直接嵌套）：

```powershell
docker ps -aq | ForEach-Object { docker rm -f $_ }
```

---

## 6. 命令速查表

> 记忆法：`docker <对象> <动作>`，如 `docker image ls`、`docker container rm`。老式短命令（`docker ps`、`docker rmi`）是它的省略形式。

### 6.1 镜像

| 命令 | 作用 |
|---|---|
| `docker pull nginx:1.27` | 拉取镜像 |
| `docker images` / `docker image ls` | 列出本地镜像 |
| `docker build -t 名字:标签 .` | 构建镜像（`.` 是构建上下文） |
| `docker tag 源 目标:标签` | 打标签（用于推送到仓库） |
| `docker push 仓库/名字:标签` | 推送 |
| `docker rmi 镜像` | 删除镜像（有容器在用会失败） |
| `docker history 镜像` | 看镜像的分层历史（排查体积、泄露密钥） |
| `docker inspect 镜像/容器` | 看完整 JSON 详情（排错神器） |
| `docker save -o a.tar 镜像` / `docker load -i a.tar` | 导出/导入镜像文件（离线传输） |

### 6.2 容器生命周期

| 命令 | 作用 |
|---|---|
| `docker run [选项] 镜像 [命令]` | 创建 + 启动（最常用） |
| `docker create` / `docker start` / `docker stop` / `docker restart` | 分步操作 |
| `docker rm 容器` | 删除（`-f` 强删运行中的） |
| `docker pause` / `unpause` | 冻结/解冻所有进程（用 cgroup freezer） |
| `docker rename 旧 新` | 改名 |
| `docker commit 容器 镜像` | 把容器当前状态存成镜像（**不推荐**，用 Dockerfile） |

**`docker run` 高频参数速查**

| 参数 | 含义 |
|---|---|
| `-d` | 后台运行 |
| `-it` | 交互式 + 伪终端（进容器/跑 shell 必备） |
| `--rm` | 退出即删除容器 |
| `--name X` | 指定名字 |
| `-p 8080:80` | 端口映射（宿主:容器），可加 `/tcp` `/udp` |
| `-v 源:目标[:ro]` | 挂载卷或目录 |
| `-e KEY=VALUE` | 环境变量（密码类优先用 `--env-file`） |
| `--network X` | 加入指定网络 |
| `-w /app` | 工作目录 |
| `--restart unless-stopped` | 自动重启策略（`no`/`on-failure`/`always`/`unless-stopped`） |
| `--memory=512m --cpus=1` | 资源限制 |
| `--user 1000:1000` | 以指定用户运行（安全实践） |
| `--read-only` | 根文件系统只读（配合 tmpfs 更安全） |
| `--health-cmd` | 健康检查 |
| `--network host` | 直接用宿主机网络（Linux 有效；Desktop 上需开启） |

### 6.3 查看与调试

| 命令 | 作用 |
|---|---|
| `docker ps -a` | 所有容器及状态（`Up` / `Exited (1)` / `Restarting`） |
| `docker logs -f --tail 100 容器` | 看日志 |
| `docker exec -it 容器 bash` | 进入运行中的容器（没有 bash 就试 `sh`） |
| `docker top 容器` | 看容器内的进程（宿主机视角） |
| `docker stats` | 实时资源占用 |
| `docker inspect 容器` | 全部配置与状态 |
| `docker cp 容器:/path ./本地` | 容器与宿主互拷文件 |
| `docker diff 容器` | 容器可写层改了什么 |
| `docker port 容器` | 查看端口映射 |
| `docker events` | 监听 Docker 事件流（方便调试） |

### 6.4 网络 / 卷 / 系统

| 命令 | 作用 |
|---|---|
| `docker network ls` / `create` / `rm` / `inspect` | 网络管理 |
| `docker network connect 网络 容器` | 给运行中容器加网络 |
| `docker volume ls` / `create` / `rm` / `inspect` | 卷管理 |
| `docker system df` | 磁盘占用总览 |
| `docker system prune` | 清理未使用资源 |
| `docker version` / `docker info` | 版本与环境信息 |

### 6.5 Compose

| 命令 | 作用 |
|---|---|
| `docker compose up -d --build` | 构建并后台启动 |
| `docker compose ps` / `logs -f 服务` | 状态 / 日志 |
| `docker compose exec 服务 bash` | 进容器 |
| `docker compose restart 服务` | 重启某个服务 |
| `docker compose down [-v]` | 停止并清理（`-v` 删卷） |
| `docker compose config` | 校验并渲染最终配置（排错 YAML 常用） |

---

## 7. Dockerfile 精讲

### 7.1 常用指令

| 指令 | 作用 | 备注 |
|---|---|---|
| `FROM` | 基础镜像 | 必须第一条；`FROM scratch` 表示从空白开始（静态编译程序用） |
| `RUN` | 构建时执行命令，产生新层 | 多个命令用 `&&` 合并，末尾 `&& rm -rf /var/lib/apt/lists/*` 瘦身 |
| `COPY` | 复制文件进镜像 | 首选；支持 `--chown` |
| `ADD` | 类似 COPY，还能解压 tar、拉 URL | 语义模糊，**建议只用 COPY** |
| `WORKDIR` | 设定工作目录（不存在会创建） | 别用 `RUN cd`，那是无效的 |
| `ENV` | 构建期 + 运行期都可见的环境变量 | 不要用它存密钥（会留在镜像里） |
| `ARG` | 只在构建期有效的变量 | `--build-arg` 传入 |
| `EXPOSE` | 声明意图端口 | **纯文档作用**，不真的开放端口；映射靠 `-p` |
| `CMD` | 默认启动命令 | 可被 `docker run 镜像 别的命令` 覆盖 |
| `ENTRYPOINT` | 入口命令 | 不易被覆盖，常与 CMD 配合 |
| `USER` | 指定运行用户 | 安全实践：别用 root |
| `VOLUME` | 声明匿名卷挂载点 | 用于强制数据外置 |
| `HEALTHCHECK` | 健康检查命令 | Compose/K8s 会据此判断就绪 |
| `LABEL` | 元数据（作者、版本、源码地址） | 便于运维追溯 |
| `SHELL` | 改默认 shell（默认 `/bin/sh -c`） | 需要 bash 特性时设置 |
| `.dockerignore` | 构建时排除文件 | 不含在指令里，但极其重要 |

### 7.2 CMD vs ENTRYPOINT（最容易被问的面试题）

| 写法 | `docker run 镜像` | `docker run 镜像 参数` |
|---|---|---|
| 只有 `CMD ["a","b"]` | 执行 `a b` | 覆盖成 `参数` |
| 只有 `ENTRYPOINT ["a"]` | 执行 `a` | 执行 `a 参数` |
| `ENTRYPOINT ["a"]` + `CMD ["b"]` | 执行 `a b` | 执行 `a 参数` |

**推荐模式**：`ENTRYPOINT` 放可执行程序，`CMD` 放默认参数。

```dockerfile
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8000"]
```

**两种写法必须分清**：

```dockerfile
CMD ["python", "app.py"]        # exec 形式（推荐）：进程就是 python，能收到 SIGTERM，能优雅停机
CMD python app.py               # shell 形式：实际是 /bin/sh -c "python app.py"，
                                #        信号发给 sh 而不是 python，容易停机失败、退出码被吞
```

### 7.3 多阶段构建（让镜像小一个数量级）

**问题**：编译型语言需要编译器（gcc、JDK、node_modules），但运行时只需要产物。

**改前**（单阶段，1.2GB）：

```dockerfile
FROM golang:1.22
WORKDIR /app
COPY . .
RUN go build -o server .
CMD ["./server"]
```

**改后**（多阶段，20MB）：

```dockerfile
# 阶段一：编译环境（这个镜像的产物不会进最终镜像）
FROM golang:1.22 AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /out/server .

# 阶段二：运行环境（只要二进制的运行时）
FROM gcr.io/distroless/static-debian12
COPY --from=builder /out/server /server
USER nonroot:nonroot
ENTRYPOINT ["/server"]
```

关键在于 `COPY --from=builder`：只搬需要的文件，编译工具链全部丢弃。

> 同样的思路适用于前端：`node:20 AS builder` → `npm ci && npm run build` → `nginx:alpine` 只放 `dist/`。

### 7.4 五条最实用的瘦身与安全优化

1. **选对基础镜像**：`ubuntu`(78MB) → `debian:slim`(75MB) → `alpine`(7MB) → `distroless`(几 MB，无 shell)。
   - ⚠️ Alpine 用 musl libc，某些预编译二进制（如老版本 glibc 依赖的 Python 包）会报 `not found` 或兼容问题。不确定就先别用 alpine。
2. **合并 RUN 并清理缓存**：

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*
```

3. **用 `.dockerignore`**：别把 `node_modules`、`.git`、`.venv`、`*.log`、密钥文件塞进构建上下文（既慢又危险）。

```
.git
node_modules
.venv
__pycache__
*.log
.env
dist
```

4. **非 root 运行**：

```dockerfile
RUN useradd -r -u 10001 appuser
USER appuser
```

5. **固定版本**：`FROM node:20.11.1-alpine3.19` 而不是 `node:latest`；`pip install` 用 `==` 锁版本。否则今天能构建，明天就不能。

### 7.5 一个生产级 Dockerfile 模板（Python）

```dockerfile
# ---------- 构建阶段 ----------
FROM python:3.12-slim AS builder
ENV PIP_NO_CACHE_DIR=1 PIP_DISABLE_PIP_VERSION_CHECK=1
WORKDIR /build
COPY requirements.txt .
RUN pip install --prefix=/install -r requirements.txt

# ---------- 运行阶段 ----------
FROM python:3.12-slim
ENV PYTHONUNBUFFERED=1 PYTHONDONTWRITEBYTECODE=1 TZ=Asia/Shanghai
WORKDIR /app

RUN groupadd -r app && useradd -r -g app -u 10001 app

COPY --from=builder /install /usr/local
COPY --chown=app:app . .

USER app
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD python -c "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://127.0.0.1:8000/').status==200 else 1)"

ENTRYPOINT ["python", "app.py"]
```

---

## 8. 数据持久化：卷与挂载

### 8.1 三种方式对比

| 方式 | 命令 | 存放位置 | 适用场景 | 备注 |
|---|---|---|---|---|
| **命名卷 Volume** | `-v mydata:/data` | Docker 管理（Linux：`/var/lib/docker/volumes/`） | **数据库、生产数据** | 性能最好，跨平台一致，可备份可迁移 |
| **绑定挂载 Bind mount** | `-v /host/path:/data` | 宿主机任意路径 | **开发时代码热更新、配置文件** | Windows/macOS 上跨 VM，性能较差 |
| **tmpfs 内存盘** | `--tmpfs /tmp` | 内存 | 临时缓存、敏感数据 | 容器停止即消失 |

### 8.2 命令速查

```bash
docker volume create appdata
docker volume ls
docker volume inspect appdata        # 看挂载点真实路径
docker run -v appdata:/data alpine sh -c 'echo hi > /data/a.txt'
docker run -v appdata:/data alpine cat /data/a.txt     # 数据还在
docker volume rm appdata             # 有容器用则失败
```

### 8.3 备份与恢复（生产必备）

```bash
# 备份：起一个临时容器，把卷内容打包到宿主机当前目录
docker run --rm -v appdata:/src -v "${PWD}:/backup" alpine \
  tar czf /backup/appdata.tar.gz -C /src .

# 恢复
docker run --rm -v appdata:/dst -v "${PWD}:/backup" alpine \
  sh -c 'tar xzf /backup/appdata.tar.gz -C /dst'
```

### 8.4 三条铁律

1. **容器无状态**：能跑丢的东西（代码、配置）都来自镜像；不能丢的（数据、上传文件、日志）全部放卷。
2. **`docker rm` 不删卷，但 `docker compose down -v`、`docker volume prune` 会删**。生产环境执行前确认三遍。
3. **日志也会撑爆磁盘**：配置 `log-opts` 限制大小（见 4.4），或把日志丢给 stdout 由外部系统收集（12-Factor 原则）。

---

## 9. 网络

### 9.1 四种常用网络模式

| 模式 | `--network` | 特点 | 用途 |
|---|---|---|---|
| **bridge**（默认） | `bridge` | 每个容器独立 IP，通过 NAT 出网，需 `-p` 才能被外部访问 | 绝大多数场景 |
| **自定义 bridge** | `my-net` | 同上，但**自带 DNS：容器名可直接当主机名用** | **推荐**，多容器互访 |
| **host** | `host` | 容器直接用宿主机网络栈，无 NAT，`-p` 无效 | 追求极致性能；Linux 有效 |
| **none** | `none` | 只有 lo，无网络 | 完全隔离的批处理任务 |

### 9.2 自定义网络 + 容器名解析（最实用）

```bash
docker network create app-net

docker run -d --name db  --network app-net -e POSTGRES_PASSWORD=secret postgres:16
docker run -d --name web --network app-net -p 8000:8000 myapp:1.0
# web 里连数据库，直接用主机名 db：
#   postgresql://postgres:secret@db:5432/postgres
```

**为什么"用容器名当域名"能工作**：Docker 在内置 DNS（127.0.0.11）里为同一网络内的容器注册了名字。**注意仅限自定义网络**——默认 `bridge` 网络不支持名字解析（老版本还需要 `--link`，已废弃，别用）。

### 9.3 端口映射的三个细节

```bash
-p 8080:80              # 宿主机 8080 → 容器 80
-p 127.0.0.1:8080:80    # 只监听本地回环（更安全）
-p 8080:80/udp          # UDP
-P                      # 随机映射镜像 EXPOSE 的所有端口
```

- **`-p` 是宿主机侧的端口，容器之间互访不需要 `-p`**（走内部网络 + 容器端口）。新手常犯：给 db 也映射了 5432，其实没必要，还增加暴露面。
- **`EXPOSE` 不开放端口**，只是文档 + `-P` 的参考。
- **`expose`/`ports` 冲突**：宿主机端口被占用会报 `port is already allocated`（第 12 章）。

### 9.4 排障口诀

```bash
docker exec -it web sh -c 'ping db || getent hosts db'    # 名字解析通不通
docker exec -it web sh -c 'wget -qO- http://db:5432'      # 端口通不通（不通很正常，PG 不是 HTTP）
docker network inspect app-net                            # 看谁在网里、IP 多少
docker exec -it web ip route                              # 看默认网关
```

---

## 10. Docker Compose：多容器编排

### 10.1 它解决什么

一个真实应用往往是"web + db + cache + worker"。用 `docker run` 一个个起，要记参数、顺序、网络。Compose 用一个 YAML 描述**整组服务**，一条命令起停。

- 文件默认名：`compose.yaml`（新）或 `docker-compose.yml`（旧）。**`compose.yaml` 是当前推荐**。
- 命令从 `docker-compose`（v1，Python 写的，已停止维护）变成 **`docker compose`**（v2，Go 插件）。别再用 v1。

### 10.2 核心结构（记住四块）

```yaml
services:      # 每个“服务”= 一个容器（或一组副本）
  web:
    build: .                       # 从当前目录 Dockerfile 构建
    image: myapp:1.0               # 也可同时指定构建出的镜像名
    ports: ["8000:8000"]
    environment:
      - DEBUG=false
    env_file: [.env]               # 从文件读环境变量
    volumes:
      - .:/app                     # 开发：代码热更新
      - /app/node_modules          # 匿名卷“遮住”宿主机目录，保护容器内依赖
    depends_on:
      - db
    restart: unless-stopped

networks:      # 自定义网络（默认会自动建一个默认网络）
  app-net:

volumes:       # 命名卷
  pgdata:
```

### 10.3 三个高频能力

| 能力 | 写法 | 用途 |
|---|---|---|
| **健康检查依赖** | `depends_on: {db: {condition: service_healthy}}` + db 的 `healthcheck` | 等 db 真就绪再启动 web（`depends_on` 默认只等"启动"，不等"就绪"） |
| **profiles** | `profiles: ["debug"]` + `docker compose --profile debug up` | 可选服务（如调试工具、pgadmin）按需启动 |
| **多文件覆盖** | `docker compose -f compose.yaml -f compose.prod.yaml up` | 基础配置 + 环境差异分离 |

### 10.4 命令与排错

```bash
docker compose config          # 校验 YAML，输出合并后的最终配置（第一个该用的命令）
docker compose up -d --build
docker compose ps
docker compose logs -f --tail=50 web
docker compose exec web bash
docker compose down -v
```

---

## 11. 最佳实践清单

**镜像构建**

- [ ] 基础镜像固定版本（不用 `:latest`），优先 slim/alpine/distroless
- [ ] 用多阶段构建，最终镜像不含编译工具链
- [ ] `COPY requirements.txt` 在前，`COPY . .` 在后（缓存命中）
- [ ] 有 `.dockerignore`
- [ ] 合并 `RUN`，清理包管理器缓存
- [ ] 镜像里**不放密钥**（`ENV`/`COPY .env` 都会留在层历史里，`docker history` 一览无余）

**运行时**

- [ ] 用非 root 用户（`USER`），必要时 `--read-only` + `--tmpfs`
- [ ] 设置资源限制（`--memory`、`--cpus`），防止单个容器拖垮宿主机
- [ ] 配置日志轮转（`log-opts`），否则磁盘必爆
- [ ] `ENTRYPOINT` 用 exec 形式，正确响应 SIGTERM
- [ ] 加 `HEALTHCHECK`
- [ ] 数据放命名卷，容器无状态
- [ ] 只暴露必要端口，`-p 127.0.0.1:...` 优于 `-p ...`（不直接暴露公网）
- [ ] 设置 `--restart unless-stopped`（守护型服务）

**协作与运维**

- [ ] `.env` 不进 git，用 `env_file` 注入
- [ ] 容器名、网络名、卷名带项目前缀（Compose 自动加）
- [ ] 定期 `docker system df` 巡检，避免磁盘 100%
- [ ] 镜像推到私有仓库并打 git commit 号 tag，便于追溯
- [ ] 生产编排交给 Kubernetes / Swarm / Nomad，别手写一堆 `docker run`

**安全（初学者最容易忽略）**

- [ ] 镜像扫描漏洞（`docker scout cves 镜像` / Trivy）
- [ ] 别 `docker run --privileged`、别挂 `/var/run/docker.sock` 进容器（等于给 root）
- [ ] 只从可信仓库拉镜像
- [ ] 不要把 Docker daemon 的 2375 端口暴露到公网

---

## 12. 常见报错与坑

| 症状 | 原因 | 解决 |
|---|---|---|
| 容器**启动后立刻退出** | 前台进程执行完就退（如 `docker run ubuntu`，ubuntu 默认命令是 bash 但无 TTY 就退出） | 给个长驻命令：`docker run -d ubuntu sleep infinity`；或 `docker logs <id>` 看原因 |
| `Exited (0)` | 程序正常执行完 | 需要常驻就换命令；批处理任务正常结束是合理的 |
| `Exited (1)` | 应用自身报错 | `docker logs`；`docker run -it 镜像 sh` 手动进去试 |
| `Exited (137)` | **被 OOM 杀**，或 stop 超时被 SIGKILL | 调大 `--memory`；优化内存；查应用内存泄漏 |
| `Exited (143)` | 正常收到 SIGTERM 退出 | 正常现象（优雅停机成功） |
| `bind: address already in use` / `port is already allocated` | 宿主机端口被占 | `netstat -ano \| findstr :8080`（Win）/ `ss -lntp`（Linux）；换端口或杀掉占用者 |
| `permission denied` 挂载目录 | 容器内用户 UID 无权写宿主机目录 | `--user $(id -u):$(id -g)`；`chown` 目录；或改用命名卷 |
| `exec: "bash": not found` | Alpine/distroless 没有 bash | 用 `sh`；或换 `debian:slim` 基础镜像 |
| `no space left on device` | 镜像/容器/日志占满磁盘 | `docker system df` → `docker system prune -a`；配日志轮转 |
| 拉镜像卡住 / `TLS handshake timeout` | 网络/加速器配置问题 | 配 registry-mirrors；`docker login`；检查代理 |
| 改了代码但**镜像没变** | 缓存或忘了 `--build` | `docker build --no-cache`；compose 用 `up -d --build` |
| 容器里**时区是 UTC** | 基础镜像默认 UTC | `ENV TZ=Asia/Shanghai` + 装 `tzdata`；或挂载 `/etc/localtime:ro` |
| 中文/emoji 乱码或方块 | 镜像缺字体/语言包 | 生产别在容器里渲染中文 PDF/图片；装 `fonts-noto-cjk` |
| `standard_init_linux.go: ... exec format error` | 架构不匹配（如 Apple Silicon 拉 amd64 镜像） | `docker buildx build --platform linux/amd64`；拉多架构镜像 |
| Python 包在 alpine 上装不上 | musl 与 glibc 不兼容，需现场编译 | 用 `python:3.12-slim`（Debian）而不是 alpine |
| 改了 `daemon.json` 不生效 | 没重启守护进程 / JSON 语法错 | `sudo systemctl restart docker`；`dockerd --validate` |
| Docker Desktop 磁盘越来越大 | 镜像层、构建缓存、卷堆积 | `docker system df`、`docker builder prune`、Desktop 里清理 |
| `Cannot connect to the Docker daemon` | 守护进程没启动 / 用户不在 docker 组 / Desktop 未启动 | 启动 Docker Desktop；`sudo systemctl start docker`；`usermod -aG docker` |
| 容器里 `ping`/`curl` 不存在 | 精简镜像没装这些工具 | `docker exec` 里装，或用 `nicolaka/netshoot` 镜像调试 |

---

## 13. 术语中英对照表

| 中文 | 英文 | 一句话 |
|---|---|---|
| 镜像 | Image | 只读模板，分层存储 |
| 容器 | Container | 镜像的运行实例 = 隔离的进程 |
| 仓库（服务） | Registry | 存镜像的服务 |
| 仓库（项目） | Repository | 某镜像的所有版本 |
| 标签 | Tag | 版本别名，如 `1.27-alpine` |
| 摘要 | Digest | 内容哈希，绝对唯一 |
| 层 | Layer | 镜像的一层只读文件系统 |
| 可写层 | Writable layer / Container layer | 容器独有的那层 |
| 写时复制 | Copy-on-Write (CoW) | 改文件先复制再改 |
| 数据卷 | Volume | 持久化存储 |
| 绑定挂载 | Bind mount | 直接映射宿主机目录 |
| 命名空间 | Namespace | 隔离"看得见什么" |
| 控制组 | Cgroup | 限制"能用多少" |
| 联合文件系统 | UnionFS / OverlayFS | 多层叠加成一个视图 |
| 容器运行时 | Container runtime | 真正创建容器的东西（runc） |
| 编排 | Orchestration | 多机器多容器的调度管理（K8s） |
| 构建上下文 | Build context | `docker build .` 里的那个 `.` |
| 健康检查 | Healthcheck | 判断容器是否"可用"而非"在跑" |
| 无状态 | Stateless | 容器本身不保存数据 |

---

## 14. 7 天学习路线

| 天 | 目标 | 动手任务 | 验收标准 |
|---|---|---|---|
| **D1** | 装好 + 建立直觉 | 装 WSL2 + Docker Desktop；跑 hello-world、nginx、alpine | 能用 `-p` 访问容器里的网页 |
| **D2** | 掌握容器生命周期 | `run/ps/logs/exec/stop/rm` 全过一遍；故意让容器退出并 `docker logs` 找原因 | 能解释 137 和 143 的区别 |
| **D3** | 会写 Dockerfile | 把 5.4 的 Flask 应用构建成镜像；改代码重新 build，观察缓存命中 | 能说清为什么 `COPY requirements.txt` 要放在 `COPY . .` 前面 |
| **D4** | 数据与网络 | 用卷跑 Postgres；建自定义网络，让 web 用容器名连 db | 删掉 db 容器重建后数据仍在 |
| **D5** | Compose | 用 compose.yaml 一键起 web+db；加 healthcheck + depends_on | `docker compose config` 能跑通，服务按顺序起来 |
| **D6** | 优化 + 安全 | 把镜像从 1GB 降到 100MB 以内（多阶段/slim/alpine）；改成非 root | `docker history` 里看不到密钥；镜像明显变小 |
| **D7** | 串联 + 排错 | 故意制造 5 个故障（端口冲突、OOM、无 bash、缓存、时区），逐个排查 | 每种都能在 3 分钟内定位 |

**学完之后往哪走**：Kubernetes（生产编排）→ CI/CD 集成（GitHub Actions 构建推送镜像）→ 可观测性（Prometheus + Grafana）→ 服务网格。

---

## 15. 自测 12 问（附答案）

1. **容器和虚拟机最本质的区别？**
   容器共享宿主机内核，隔离的是进程视图；虚拟机有独立内核，虚拟的是硬件。

2. **镜像是什么？为什么多个镜像能共享空间？**
   只读的分层文件系统 + 元数据。相同的基础层按 digest 去重共享。

3. **`docker run` 背后的三个动作？**
   pull（若无本地镜像）→ create（建可写层、配 namespace/cgroup）→ start（execve 启动进程）。

4. **容器里的 PID 1 是什么？为什么 `systemctl` 用不了？**
   是你的应用进程。没有 systemd/init 体系，容器不需要也不需要"初始化系统"。

5. **`docker stop` 和 `docker kill` 区别？**
   stop 先发 SIGTERM，等 10 秒（`-t` 可改）后 SIGKILL；kill 直接发信号（默认 SIGKILL）。

6. **为什么容器重启后数据没了？怎么解决？**
   可写层随容器删除而消失。数据放命名卷或绑定挂载。

7. **`-p 8080:80` 谁在前谁在后？容器之间互访要写 `-p` 吗？**
   宿主机:容器。不需要——同网络内用容器名 + 容器端口直接访问。

8. **`EXPOSE` 有什么用？**
   纯声明/文档，不开放端口；映射靠 `-p`。

9. **`CMD` 和 `ENTRYPOINT` 的区别？**
   CMD 是默认命令/参数，可被 `docker run` 后的命令覆盖；ENTRYPOINT 是固定入口，后面的参数成为它的参数。

10. **Alpine 镜像为什么小？什么时候别用？**
    用 musl libc + busybox，基础层仅几 MB。需要 glibc 的预编译程序、依赖 glibc 的 Python 包、或要 bash 时不要用。

11. **`docker compose down` 和 `down -v` 的区别？**
    前者删容器和网络、保留卷；后者连卷一起删，**数据会被清空**。

12. **怎么让镜像构建更快？**
    利用层缓存（变化少的放前面）、`.dockerignore` 精简上下文、多阶段构建、必要时 `--cache-from` / BuildKit 缓存挂载。

---

## 16. 延伸阅读

- 官方文档（最权威，英文）：<https://docs.docker.com/>
- Dockerfile 参考：<https://docs.docker.com/reference/dockerfile/>
- Compose 文件规范：<https://docs.docker.com/compose/compose-file/>
- 最佳实践：<https://docs.docker.com/build/building/best-practices/>
- Play with Docker（浏览器里练，不用装）：<https://labs.play-with-docker.com/>
- OCI 镜像规范：<https://github.com/opencontainers/image-spec>
- Linux namespaces 手册：`man 7 namespaces`、`man 7 cgroups`
- 12-Factor App（容器化应用的指导思想）：<https://12factor.net/zh_cn/>

---

> **最后一句总结**：
> Docker 把"环境"变成了可版本化、可分发、可复现的**代码**。
> 理解了"容器就是被 namespace 隔离、被 cgroup 限流、跑在分层文件系统上的一个进程"，**剩下的全是命令和约定**——查手册就够了。
