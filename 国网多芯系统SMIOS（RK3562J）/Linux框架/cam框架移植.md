# 一、背景情况

## 约束：多芯系统因验签文件，不能改内核

多核（SMIOS）板上跑的是已验签内核：

| 项 | 现状 |
|------|------|
| 内核 | `5.10.198 #236`，Image / U-Boot 不能刷 |
| MEDIA | `# CONFIG_MEDIA_SUPPORT is not set`，无 videodev / 无 `/dev/video*` media 栈 |
| 现网 DTB | 多核 dtsi 常裁掉 CSI/CIF/ISP；需板级补齐 |
| 可改项 | DTB 不验签；可加载外部 `.ko`；rootfs 可加用户态 |
| 目标硬件 | RK3562J CORE DDR4 V10，GC05A2 @ i2c5 `0x37`，2 lane MIPI |
| 业务目标 | **连续 NV12 + rkaiq 3A 彩图**（不要内核默认 IQ） |

## 硬约束（与 DRM 不同）

- **不要改** SDK：`kernel-5.10/.../rockchip/isp/`、`external/camera_engine_rkaiq`
- **只改** `kx/移植/cam/` 下副本：`video_rkisp/`、`gc05a2/`、`media-deps/`、`camera_engine_rkaiq/`、脚本
- Image `#236`：`task_struct.stack_canary = **1192**`（本地 tree 常编出 1208 → 假 stack-protector）

## 对照：开发板怎么出图

SDK Buildroot 开发板常见是：

```text
内核 CONFIG_MEDIA_SUPPORT=y + CIF/ISP/sensor 编进 Image 或模块
  + 完整 DTB（dphy / mipi-csi2 / rkcif / rkisp / sensor）
  + /etc/iqfiles + rkaiq_3A_server
  → media graph 建链 → STREAMON → NV12 + 3A
```

## 方案演进（认识过程）

1. 现核无 MEDIA → 必须整栈 OOT（mc/videodev/vb2/phy/cif/isp/sensor）。
2. 驱动源码用 **SDK 226 系 Rockchip 树副本**，用 **198 ABI 头 + canary 1192** 编出 vermagic=`5.10.198`。
3. 无可用 IOMMU domain：关 IQ DDR 模块，走寄存器 3A + 真 DMA-CMA。
4. 与 rkaiq 握手对齐 STREAM_START/STOP；二次抓拍靠真 CMA + keep IQ，不用「永远不 free」堵 UAF。

---

# 二、摄像头打通链路的整体流程

```text
[1] 换 DTB（补相机节点）
      i2c5/gc05a2 + csi2-dphy + mipi-csi2 + rkcif + rkisp(+vir0) …
        ↓
[2] ABI=198 编译 OOT media 栈（+ canary 1192）+ rkaiq
      media_deps → mc/videodev/vb2 → dphy → rkcif → rkisp → gc05a2
      Buildroot 编 rkaiq / rkaiq_3A_server → cam-kos-198/tools/rkaiq/
        ↓
[3] 板上 insmod.sh 按序加载
        ↓
[4] sensor 0x37 UU；media 设备出现；isp is_mmu=0
        ↓
[5] start_3a.sh：IQ→/tmp/iqfiles、media_link_setup --with-3a、起 3A
        ↓
[6] grab_3a.sh：v4l2_grab 抓 NV12（默认 2 buf）；可连续第二次
```

硬件 / 软件数据路径：

```text
GC05A2 (MIPI CSI-2)
  → csi2-dphy0 → mipi0-csi2 → rkcif_mipi_lvds
  → rkcif_mipi_lvds_sditf → rkisp_vir0
  → rkisp_mainpath (/dev/videoN) ──► NV12
         ▲
         │ params / stats（与 rkaiq_3A_server 握手）
```

---

# 三、踩过的坑 & 做了什么

## 阶段 A：模块装不上 / 一用就假 panic（ABI / canary）

| 坑 | 现象 | 处理 |
|----|------|------|
| vermagic | 编出 `5.10.226`，板是 `5.10.198` | `switch-kernel-abi.sh 198`，SUBLEVEL=198 |
| `stack_canary` 1208 vs 1192 | `v4l2_event_dequeue` 等一 sleep 就假栈破坏 | `build_cam_kos.sh` 强制 `TSK_STACK_CANARY=1192` 并 objdump 校验 |
| `Unknown symbol cma_alloc` 等 | vb2-cma-sg / frame_vector / iommu 助手未导出 | 自研 `media_deps.ko`，**最先** insmod |

## 阶段 B：能装模块，但二次抓失败 / Oops

| 坑 | 现象 | 处理 |
|----|------|------|
| 假 CMA：`alloc_pages`+`split_page` | 第一次 NV12 OK，第二次 `cma_alloc: fail count=1853 order=11` | 改走 `dma_alloc_from_contiguous()` 真 CMA；grab 默认 `-b 2` |
| `MAX_ORDER=11` | buddy 最大 4MiB，一帧 ~8MiB | 同上，不能靠 buddy 硬抠 8MiB |
| STREAMOFF 与 3A 顺序 | 二次 grab Oops / params dma_fd UAF | STREAMOFF **先** `rkisp_stop_3a_run` **再** `params_stream_stop`；params 仍 open 时 keep IQ CMA |
| 无 IOMMU 仍开 BAY3D 等 | 用物理地址写 DDR IQ → 砸内存 | `is_mmu=0` 时 skip BAY3D/LDCH/CAC/3DLUT；stats 走寄存器 |
| poison ISR / 坏 vb2 | STREAMOFF 后 IRQ 仍 `vb2_buffer_done` | `rkisp_v32_stream_irq_ok` / `rkisp_vb2_buf_ok` |

## 阶段 C：用户态 / 流程

| 坑 | 现象 | 处理 |
|----|------|------|
| 误跑旧 `grab.sh` | 杀 3A / 重搭 link → 灰图断流 | 只用 `start_3a` + `grab_3a` |
| IQ 路径 | SMIOS 无 `/etc/iqfiles` | 默认 `/tmp/iqfiles/`；`start_3a` 拷贝 |
| 分辨率 | 默认 IQ/prepare 不对 | `rkaiq_3A_server` 默认 **2592×1944** |
| glibc | 板 linker 跑不动 BR rkaiq | 私有 `ld-linux --library-path` |
| `Bad media topology media2–15` | rkaiq 刷屏 | **可忽略** |
| CIF `/dev/video1` open fail | 非主路径 | **可忽略** |
| `cam_mclk` 与 sensor 自管电 | 时钟/复位冲突 | sensor 自管 xvclk；insmod 卸 `cam_mclk` |

细节案例见 [`OOT-KO-OOPS-CASES.md`](./OOT-KO-OOPS-CASES.md) 案例 15–20。

---

# 四、改了哪些代码 / 文件

## 1. 只改 `kx/移植/cam/` 副本（不改 SDK isp / external rkaiq）

| 路径 | 改动要点 |
|------|----------|
| `media-deps/cma_shim.c` 等 | 导出 `cma_alloc`→真 CMA；frame_vector / iommu stub |
| `video_rkisp/hw.c` | 无 domain → `is_mmu=0` |
| `video_rkisp/isp_params_v32.c` | skip IQ DDR；keep IQ CMA |
| `video_rkisp/isp_stats_v32.c` | 无 MMU 关 3A_DDR_WRITE |
| `video_rkisp/rkisp.c` | STREAMOFF 先 stop_3a |
| `video_rkisp/capture_v32.c`、`common.h` 等 | IRQ/vb2 防护 |
| `gc05a2/gc05a2.c` | 自管电；去 `mutex_destroy` |
| `camera_engine_rkaiq/.../rkaiq_3A_server.cpp` | IQ 路径 / 2592×1944 / CLI |

## 2. 设备树

| 文件 | 改动要点 |
|------|----------|
| `kx/移植/rockchip/kx_rk3562-camera.dtsi` | 补回 dphy / mipi-csi2 / rkcif / rkisp 等 SoC 节点 |
| `kx_*_v10.dts`（及 smios 同步树） | i2c5/gc05a2、链路 endpoint、`&rkcif`/`&rkisp`/`&rkisp_vir0` okay；mmu 常 disabled |
| 产物 | `kx_kernel-dtb.img` 一类 |

## 3. OOT 交付

| 路径 | 作用 |
|------|------|
| `cam/cam-kos-198/*.ko` | media 整栈 + sensor |
| `cam/cam-kos-198/insmod.sh` | 依赖顺序加载 |
| `start_3a.sh` / `grab_3a.sh` | 3A + 抓帧 |
| `tools/rkaiq/` | 用户态库 / 3A / iqfiles |
| `tools/v4l2_grab`、`media_link_setup` | 自研小工具 |

## 4. 明确未改 / 不能改

- 验签 Image  
- SDK 树内 `.../rockchip/isp/`、`external/camera_engine_rkaiq`（只改拷贝）  
- 不为堵 UAF 做「永远不 free IQ / dummy divert」

---

# 五、程序详解（移植视角）

## 总览

```text
DT 拓扑 ──► platform/i2c 驱动 probe（sensor/dphy/cif/isp）
         ──► media controller 实体与 pad
         ──► 用户态 media_link_setup 建链路
         ──► rkaiq 订阅 STREAM_START/STOP，喂 params/stats
         ──► V4L2 capture STREAMON → DQBUF → NV12 文件
```

## 模块加载（`CONFIG_MEDIA_SUPPORT` 未开）

典型顺序（见 `insmod.sh`）：

```text
media_deps
  → mc → videodev → v4l2-* → vb2-*
  → phy-rockchip-csi2-dphy* → phy-rockchip-mipi-rx
  → video_rkcif → video_rkisp → gc05a2
```

缺符号先补 `media_deps`，不要一上来装 isp。

## 3A 握手（必须对齐）

```text
应用 STREAMON / 抓帧
  → 内核发 CIFISP_V4L2_EVENT_STREAM_START
  → rkaiq sysctl_start
  → … 跑 AE/AWB …
应用 / 驱动 STREAMOFF
  → 先 STREAM_STOP 事件 → rkaiq sysctl_stop
  → 再 params_stream_stop（OOT：此时可 keep IQ CMA）
```

## 对照：出问题看哪

| 现象 | 先看 |
|------|------|
| Unknown symbol | `insmod` 顺序、`media_deps` |
| 假 stack-protector | canary 是否 1192 |
| 二次 REQBUFS 失败 | `cma_alloc` 日志是否 `dma-cma` |
| 二次 Oops | STREAMOFF 顺序、keep IQ、是否 skip BAY3D |
| 灰图 uv0%≈100 | 3A 是否活着、IQ 路径 |
| 无 0x37 UU | DT clocks/gpio、卸 cam_mclk |

---

# 六、怎么编译 & 怎么移植上板

## 环境

```bash
# SDK 根目录
./kx/移植/switch-kernel-abi.sh 198 --with-config
# .ko：主机 aarch64-linux-gnu-
# rkaiq：Buildroot 工具链（build_rkaiq.sh 内）
```

## 编译

```bash
bash ./kx/移植/cam/build_cam_kos.sh
# 内含 canary 强制、media/cif/isp/sensor、并调 build_rkaiq.sh
bash ./kx/移植/rockchip/build_kx_kernel_dtb.sh   # 含相机节点的 DTB
```

产物：`kx/移植/cam/cam-kos-198/`，vermagic 应为 `5.10.198`。

## 上板

```bash
# 先刷相机使能 DTB，reboot
scp -r kx/移植/cam/cam-kos-198 root@板子:/root/
cd /root/cam-kos-198
sh insmod.sh
# 期望：media_deps … DMA-CMA；OOT isp is_mmu=0；0x37 UU

sh start_3a.sh
sh grab_3a.sh /root/cam3a.nv12
sh grab_3a.sh /root/cam3a2.nv12

python3 -c "d=open('/root/cam3a2.nv12','rb').read();y=d[:2592*1944];uv=d[2592*1944:];print('Ymean',round(sum(y)/len(y),1),'UVmean',round(sum(uv)/len(uv),1),'uv0%',round(100*uv.count(0)/len(uv),1))"
```

成功 dmesg 关键字：

```text
media_deps cma_alloc: dma-cma count=1853
OOT isp is_mmu=0 contig=1
OOT no IOMMU: skip BAY3D/LDCH/CAC/3A-DDR
Waiting for 3A on/off …
keep IQ/3DNR/stats CMA (params still open)
```

---

## 附录：路径速查

| 项 | 路径 |
|----|------|
| 移植副本 | `kx/移植/cam/` |
| 上板包 | `kx/移植/cam/cam-kos-198/` |
| 操作 README | `kx/移植/cam/README.md` |
| 框架详解 | `kx/移植/cam框架详解.md` |
| Oops 案例 | `kx/移植/OOT-KO-OOPS-CASES.md`（15–20） |
| 相机 DT | `kx/移植/rockchip/kx_rk3562-camera.dtsi` + 板级 dts |
