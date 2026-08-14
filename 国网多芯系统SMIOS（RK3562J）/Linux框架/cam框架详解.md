# 摄像头框架通关指南（V4L2/MC 通用核心 + Rockchip 三层源码）

> 目标：先建立 **Linux V4L2 + Media Controller + videobuf2** 通用模型，再对照本仓库 **Rockchip OOT 移植源码** 吃透 probe → 建链 → 出图 + 3A。  
> 代码根（副本）：`kx/移植/cam/`；板级 DT：`kx/移植/rockchip/`、`kx/smios-sdk-v2.0/dts/`。  
> 移植踩坑见 [`cam框架移植.md`](./cam框架移植.md)；上板短手册见 [`cam/README.md`](./cam/README.md)。

---

# 第一部分：摄像头通用核心（与芯片无关）

## 1. 摄像头子系统要回答什么

1. **传感器**怎么出 MIPI/并行像素？  
2. SoC 上 **哪个控制器**收包、拆包、做 ISP？  
3. 用户态怎么拿到一帧（格式、缓冲、流控）？  
4. （可选）**3A**（AE/AWB/AF）参数怎么与硬件握手？

老接口偏「单设备 `/dev/video0`」。  
现代 SoC 摄像头是：

- **V4L2**：捕获/输出设备节点、格式、缓冲、STREAMON/OFF  
- **Media Controller (MC)**：实体（entity）+ pad + link，描述图拓扑  
- **videobuf2 (vb2)**：缓冲队列（mmap / dma-buf / userptr）

用户态（v4l2-ctl、自研 `v4l2_grab`、rkaiq）通过 `/dev/video*` + `/dev/media*` 操作。

本仓库业务目标：**`rkisp_mainpath` 上连续 NV12 + rkaiq 3A 彩图**。

---

## 2. 核心对象模型

逻辑管道（抽象）：

```text
Sensor ──► Subdev(s) ──► Bridge/CIF ──► ISP ──► Capture video node
                                              │
                                    params / stats（3A）
```

| 对象 | 含义 | 用户态大致对应 |
|------|------|----------------|
| **video_device** | `/dev/videoN`：捕获或输出 | open / S_FMT / REQBUFS / DQBUF |
| **v4l2_subdev** | 无独立大节点的「处理块」（sensor、phy、isp 子模块） | `VIDIOC_SUBDEV_*` / MC |
| **media_entity** | MC 里的一个节点（可挂 video 或 subdev） | `media-ctl -p` |
| **media_pad** | entity 上的输入/输出端 | pad 编号 |
| **media_link** | pad 之间的边（可 enable/disable） | `media-ctl -l` / `media_link_setup` |
| **vb2_queue** | 一帧缓冲队列 | REQBUFS / QBUF / DQBUF |

关系：

```text
MC 建好 link → 应用对 capture 节点 STREAMON
  → 驱动启动传感器时钟/流水线
  → 完成帧进 vb2 → DQBUF 到用户态
```

一次完整「出图」至少要：拓扑 link 正确、格式协商一致、有足够 DMA 缓冲、STREAMON、（彩图还需）3A 在喂 params。

---

## 3. 设备节点：`/dev/video*` / `/dev/media*` / `/dev/v4l-subdev*`

### 3.1 Media 设备

驱动注册 `media_device` 后出现 `/dev/mediaN`。  
一个 SoC 上常有多个 media 图（CIF 一路、ISP 一路…）。  
rkaiq 可能扫到 `media2–15` 报 *Bad topology*——**本板可忽略**，真正用的是带 `rkisp_mainpath` 的那张图。

### 3.2 Video 节点

| 类型 | 例子（概念） | 用途 |
|------|--------------|------|
| Capture | `rkisp_mainpath` | 取 NV12 业务帧 |
| Meta / params | `rkisp_isp_params` | 3A 写 ISP 参数 |
| Meta / stats | `rkisp_isp_stats` | 3A 读统计 |
| 其它 | CIF raw、selfpath… | 本业务常不用 |

节点编号随加载顺序变，**不要写死 `/dev/video11`**；用名字/`media-ctl` 查。

### 3.3 ioctl 分发（用户态入口）

应用通过 libc/v4l2 打 ioctl，核心在 videodev：

| 能力 | 典型 ioctl / 语义 |
|------|-------------------|
| 查能力 | `QUERYCAP` |
| 设格式 | `S_FMT` / `G_FMT` |
| 缓冲 | `REQBUFS` → `QUERYBUF` → `QBUF` / `DQBUF` |
| 流控 | `STREAMON` / `STREAMOFF` |
| 事件 | `DQEVENT`（如 STREAM_START/STOP） |
| MC | `MEDIA_IOC_*` 建链/查拓扑 |

---

## 4. videobuf2 与内存

```text
REQBUFS(n)
  → vb2 向驱动 .buf_init / .buf_prepare
  → 分配 DMA 内存（本板：CMA / contig）
STREAMON
  → 硬件写物理地址
DQBUF
  → 用户态读 mmap / 拷文件
STREAMOFF
  → 停硬件、还缓冲（注意与 3A 生命周期对齐）
```

本板要点：

- Image `#236`：`MAX_ORDER=11` → buddy 最大 **4MiB**；一帧 2592×1944 NV12 ≈ **8MiB**  
- 必须走 **真 DMA-CMA**（`media_deps` 导出的 `cma_alloc` → `dma_alloc_from_contiguous`）  
- grab 默认 **-b 2**，避免一次占满 32MB CMA 池

---

## 5. Media Controller：为什么要建链

Rockchip 摄像头不是「一个驱动自动串好」，而是：

```text
多个 platform/i2c 驱动各自 probe 成 entity
  → 用户态（或驱动默认）enable link
  → 数据才能从 sensor pad 流到 mainpath
```

本仓库：`tools/media_link_setup --with-3a` 负责 GC05A2→…→`rkisp_vir0`→mainpath，并挂上 params/stats。

---

## 6. 3A 握手（用户态 ↔ 内核）

rkaiq 不直接「开摄像头」，而是：

```text
等内核发 CIFISP_V4L2_EVENT_STREAM_START
  → sysctl_start：读 stats、算 AE/AWB、写 params
等 STREAM_STOP
  → sysctl_stop
```

应用侧 `STREAMON`/`STREAMOFF` 与事件顺序错了，会出现：灰图、二次抓 Oops、params dma_fd UAF。  
OOT 关键修复：STREAMOFF **先** `rkisp_stop_3a_run` **再** `params_stream_stop`；params 仍 open 时 **keep IQ CMA**。

---

## 7. 与「内核内置 MEDIA」的关系

| | 开发板（SDK 整编） | 本板 SMIOS OOT |
|--|-------------------|----------------|
| `CONFIG_MEDIA_SUPPORT` | 常 =y | **not set** |
| 符号 | 内核已导出 | 缺 → `media_deps.ko` |
| 驱动位置 | SDK 树内 | **仅** `kx/移植/cam/` 副本 |
| IOMMU | 常可用 | DT 关 / 无 domain → `is_mmu=0` |

---

## 8. 通用核心文件（概念路径）

```text
drivers/media/
  mc/                 # media controller
  v4l2-core/          # videodev、ioctl、事件
  common/videobuf2/   # vb2
  platform/...        # SoC CIF/ISP
  i2c/                # sensor
```

芯片相关在 `platform/rockchip/`、`i2c/gc05a2.c`；本仓库对应 **`cam/video_rkisp/`、`cam/gc05a2/`** 等副本。

---

# 第二部分：结合源码的三层（Rockchip 实战）

总览：

```text
① OF / I2C compatible     驱动认硬件、probe 成 subdev / video
② Media graph 建链         entity/pad/link → 通数据
③ V4L2 流 + 3A            STREAMON、vb2、params/stats → NV12 彩图
```

对应启动：`insmod.sh` → `start_3a.sh` → `grab_3a.sh` + 相机使能 DTB。

硬件数据路径（本板）：

```text
GC05A2 @ i2c5 0x37
  → csi2-dphy0 → mipi0-csi2 → rkcif_mipi_lvds
  → rkcif_mipi_lvds_sditf → rkisp_vir0
  → rkisp_mainpath ──► NV12
         ▲
         │ rkisp params / stats ↔ rkaiq_3A_server
```

---

## 第一层：驱动注册 + DT / I2C 匹配

### 1.1 模块与角色

典型 OOT 加载（见 `cam-kos-198/insmod.sh`）：

```text
media_deps          # 补 cma_alloc / frame_vector / iommu stub
mc, videodev, v4l2-*, vb2-*
phy-rockchip-csi2-dphy*, phy-rockchip-mipi-rx
video_rkcif
video_rkisp
gc05a2
```

| 角色 | 典型 compatible / 绑定 | 副本位置 |
|------|------------------------|----------|
| Sensor | I2C `gc05a2` @ `0x37` | `cam/gc05a2/` |
| DPHY | `rockchip,rk3568-csi2-dphy` 等 | SDK phy（编进 cam 包） |
| MIPI CSI-2 | rockchip mipi-csi2 | 同上 |
| CIF | `rockchip,rk3568-cif` 一类 | `video_rkcif` ko |
| ISP HW + vir | `rockchip,rk3562-rkisp` 等 | `cam/video_rkisp/` |

### 1.2 匹配算法

```text
DT status=okay → platform_device / i2c_client
.of_match_table / i2c id 命中 → .probe
  → v4l2_subdev_init / video_register_device / media_device_register
```

板级要点：

```text
kx_rk3562-camera.dtsi     # 补回被多核 dtsi 裁掉的 SoC 相机节点
板级 .dts
  &i2c5 { gc05a2@37 { clocks/reset/pwdn; port → dphy } }
  endpoint 串：sensor → dphy → mipi-csi2 → rkcif → isp
  &rkisp_mmu { status = "disabled" }   # 常如此 → 无 IOMMU
```

`status=disabled` → 不 probe → graph 缺环。

### 1.3 第一层结束状态

- `i2cdetect` 上 **0x37 UU**（驱动占住）  
- `/dev/media*`、`/dev/video*` 出现  
- dmesg：`OOT isp is_mmu=0 contig=1`（无 domain 时）  
- **尚未**保证数据通路（link 可能未 enable）

---

## 第二层：Media graph 建链

### 2.1 驱动侧：entity 已在

CIF/ISP/sensor probe 时注册 entity 与 pad；部分默认 link 由驱动创建，**业务主路径**仍常需用户态显式 enable。

### 2.2 用户态：`media_link_setup --with-3a`

```text
start_3a.sh
  → 拷 IQ 到 /tmp/iqfiles
  → media_link_setup --with-3a
  → 启动 rkaiq_3A_server（私有 ld-linux + library-path）
```

建链语义（概念）：

```text
enable: gc05a2 → csi2-dphy0 → mipi0-csi2 → rkcif → sditf → rkisp_vir0
enable: vir0 → rkisp_mainpath
enable: params / stats 与 ISP 的 meta 边
```

### 2.3 第二层结束状态

- `media-ctl -p`（或工具日志）显示主路径 enabled  
- 3A 进程在等 **STREAM_START**  
- 尚无业务帧文件（需第三层 STREAMON）

---

## 第三层：V4L2 流 + 3A → NV12

### 3.1 抓帧：`grab_3a.sh` / `v4l2_grab`

```text
找到 rkisp_mainpath 节点
S_FMT：NV12，2592×1944
REQBUFS：默认 2
STREAMON
  → 内核事件 STREAM_START → rkaiq sysctl_start
循环 DQBUF → 写 .nv12 文件
STREAMOFF
  → STREAM_STOP → rkaiq stop
  → OOT：先 stop_3a，再 params_stream_stop；必要时 keep IQ CMA
```

### 3.2 ISP 无 IOMMU 时的寄存器策略（OOT）

```text
hw.c          DT 有 iommus 但无 domain → is_mmu=0
isp_params_*  skip BAY3D / LDCH / CAC / 3DLUT 的 DDR 表
isp_stats_*   不使能 3A_DDR_WRITE；stats 走寄存器
```

结果：**无硬件 3DNR/mesh**，但 AE/AWB/CCM 仍可跑，彩图可用。

### 3.3 内存路径（OOT）

```text
vb2-cma-sg → cma_alloc（media_deps）
  → dma_alloc_from_contiguous（真 CMA）
dmesg: media_deps cma_alloc: dma-cma count=1853
```

假路径 `alloc_pages+split_page` 仅第一次可能碰巧成功，第二次必挂。

### 3.4 第三层结束状态 / 验收

```text
/root/cam3a.nv12、再次 grab → cam3a2.nv12 都写出
Ymean 合理；uv0% 不要接近 100（接近 100 ≈ 灰图/无 3A）
dmesg 有 keep IQ…、skip BAY3D…、Waiting for 3A…
用户态：wait stream start/stop event success
```

---

## 对照表：三层 ↔ 本仓库文件

| 层 | 关键代码 / 脚本 |
|----|----------------|
| ① Probe | `gc05a2/gc05a2.c`，`video_rkisp/hw.c`，DT `kx_rk3562-camera.dtsi` + 板 dts，`insmod.sh` |
| ② Link | `tools/media_link_setup`，`start_3a.sh` |
| ③ Stream+3A | `video_rkisp/rkisp.c`（STREAMOFF 顺序），`isp_params_v32.c` / `isp_stats_v32.c`，`capture_v32.c`，`grab_3a.sh`，`camera_engine_rkaiq/.../rkaiq_3A_server.cpp` |
| 基建 | `media-deps/cma_shim.c`，`build_cam_kos.sh`（canary **1192**） |

---

## 排障速查

| 现象 | 先查哪一层 |
|------|------------|
| Unknown symbol / 装不上 ko | ① 依赖顺序、`media_deps`、ABI 198 |
| 假 stack-protector | ① canary 是否强制 1192 |
| 0x37 不是 UU | ① DT 电源时钟；卸 `cam_mclk` |
| Bad topology / CIF open fail | ② 可忽略噪音 |
| STREAMON 无帧 / 灰图 | ② link；③ 3A 是否活、IQ 路径 |
| 第二次 REQBUFS 失败 | ③ 是否真 CMA、`-b 2` |
| 第二次 Oops / dma_fd | ③ STREAMOFF 与 3A、keep IQ |

更细案例：[`OOT-KO-OOPS-CASES.md`](./OOT-KO-OOPS-CASES.md) 15–20。

---

## 附录：路径速查

| 项 | 路径 |
|----|------|
| 框架移植（故事+步骤） | `kx/移植/cam框架移植.md` |
| 本文件 | `kx/移植/cam框架详解.md` |
| 上板短手册 | `kx/移植/cam/README.md` |
| 上板包 | `kx/移植/cam/cam-kos-198/` |
| ISP 副本 | `kx/移植/cam/video_rkisp/` |
| 从零总览 | `kx/移植/OOT-PORTING-GUIDE.md` |
