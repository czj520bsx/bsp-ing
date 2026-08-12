# DRM 框架通关指南（通用核心 + Rockchip 三层源码）

> 目标：先建立 **Linux DRM/KMS 通用模型**，再对照本仓库 **Rockchip OOT 移植源码** 吃透三层挂载与出图。  
> 代码根：`kernel-5.10/drivers/gpu/drm/`，板级 DT：`kx/rockchip/`。

---
# 第一部分：DRM 通用核心（与芯片无关）

## 1. DRM 解决什么问题

显示子系统要回答：

1. 显存里有没有一帧像素？  
2. 显示控制器按什么时序去扫？  
3. 扫出来的信号走哪条物理接口到屏？

老接口是 **fbdev**（`/dev/fb0`）：一块缓冲 + 简单 ioctl。  
现代接口是 **DRM/KMS**：

- **DRM**：设备节点、缓冲（GEM）、权限、ioctl 分发  
- **KMS（Kernel Mode Setting）**：在内核里配置分辨率、图层、连接器（mode setting）

用户态（modetest、Weston、Qt `linuxfb+FB_DRM`、Xorg 等）都通过 `/dev/dri/card*` 操作 KMS。

> 本板 Qt 出图走的是 DRM dumb，**不依赖** `/dev/fb0`。

---
## 2. 核心对象模型

显示管道：

```text
          ┌─────────┐
  FB ────►│  Plane  │──┐
          └─────────┘  │
                       ▼
                   ┌────────┐     ┌─────────┐     ┌────────────┐
                   │  CRTC  │────►│ Encoder │────►│ Connector  │──► 物理屏
                   └────────┘     └─────────┘     └────────────┘
                      ▲
                      │ mode（分辨率/刷新率/porch）

```

| 对象              | 含义                                 | 用户态大致对应                          |
| --------------- | ---------------------------------- | -------------------------------- |
| **Plane**       | 一层图（主图层 / 叠加层 / 鼠标层等）              | `drmModeSetPlane` / atomic plane |
| **CRTC**        | 扫描引擎 + 时序发生器                       | mode、active                      |
| **Encoder**     | 把像素流编成 HDMI/DP/LVDS/DSI…           | 通常随 connector 一起用                |
| **Connector**   | “插了什么口/什么屏”、有哪些 mode、是否 connected  | `modetest -c`                    |
| **Framebuffer** | 一帧图像的句柄（宽高、pitch、格式、指向的缓冲）         | AddFB                            |
| **GEM object**  | 内核管理的显存对象（可 mmap、可 export dma-buf） | dumb / gem handle                |

关系：
```text

Plane 贴上 FB → 交给 CRTC 扫 → Encoder 编码 → Connector 到屏

```

一次完整“亮屏”至少要：有 FB、有 mode、plane/crtc/connector 绑定正确、encoder 打开。

---
## 3. 设备节点与会话：`drm_device` / `drm_minor` / `drm_file`

### 3.1 `drm_device`

驱动调用 `drm_dev_alloc()` + `drm_dev_register()` 后，内核出现：

```text

/dev/dri/card0     # 主节点：modeset + 通常也能做渲染相关（视驱动）

/dev/dri/renderD128 # 可选：纯渲染节点（无 modeset）

```

Rockchip 注册时驱动名等见：

```text

kernel-5.10/drivers/gpu/drm/rockchip/rockchip_drm_drv.c

  DRIVER_NAME "rockchip"

  struct drm_driver rockchip_drm_driver
  
```

`drm_driver` 里挂着：dumb_create、ioctl、fops、feature 标志（MODESET/GEM/ATOMIC/RENDER…）。
### 3.2 `drm_file`（每个 open 一次）

用户态 `open("/dev/dri/card0")` → 内核 `drm_file_alloc()`。
本仓库 OOT 补丁（现网 `task_struct` ABI）：

```text

drivers/gpu/drm/drm_file.c

  file->pid = NULL;   // 避免 close 时 put_pid oops

```

`drm_file` 上挂着该客户端的 FB 列表、event、权限等；`close` → `drm_release` → `drm_file_free`。
### 3.3 ioctl 分发（用户态入口）

用户态 libdrm 把请求打成 ioctl，通用层在 `drm_ioctl()` 分发到：

- DRM 核心表（mode/gem/…）  
- 驱动私有 ioctl（若有）

对显示最常用的（概念名）：

| 能力     | 典型 ioctl / 语义                            |
| ------ | ---------------------------------------- |
| 查资源    | get resources / get connector / get mode |
| 分配扫描缓冲 | `CREATE_DUMB` + `MAP_DUMB`               |
| 登记 FB  | `ADDFB` / `ADDFB2`                       |
| 配显示    | 老接口 `SETCRTC`；新接口 **Atomic** commit      |
| 换页     | page_flip / atomic nonblocking           |

读代码可从：`drivers/gpu/drm/drm_ioctl.c`、`drm_mode_config.c`、`drm_atomic_*.c`。

---
## 4. Mode Config：对象都挂在哪

每个 `drm_device` 有一份 `drm_mode_config`：

```text

mode_config
  ├── plane_list / num_total_plane
  ├── crtc_list / num_crtc
  ├── encoder_list
  ├── connector_list / num_connector
  ├── funcs（atomic_check / atomic_commit / fb_create …）
  └── 各种全局 property

```

驱动在 bind 阶段：

1. `drmm_mode_config_init()`  
2. 各子模块 `drm_crtc_init` / `drm_universal_plane_init` / `drm_encoder_init` / `drm_connector_init`  
3. `drm_mode_config_reset()` 建初始 state  

Rockchip master bind 里顺序见下文「第二层」。

---
## 5. Atomic KMS（现代配屏模型）

老接口：多次 ioctl 拼一次 modeset（易中间态、难回滚）。  
新接口：**Atomic**：

```text

准备一套 drm_atomic_state
  ├── crtc_state（mode、active、…）
  ├── plane_state（fb、src/dst、…）
  └── connector_state（crtc 绑定、…）

atomic_check（校验）

atomic_commit（一次性提交；可异步 + 等 vblank）

```

驱动实现：

- `mode_config.funcs->atomic_check`  
- `mode_config.funcs->atomic_commit`  
- 各 crtc/plane helper 的 `atomic_enable` / `atomic_update` / `atomic_flush`  

排障时：“modeset 失败”多在 check；“commit 成功但黑屏”多在 enable/地址/时钟。

---
## 6. 缓冲：dumb / GEM / dma-buf / IOMMU

### 6.1 dumb buffer（无 GPU 也够用）

为 **CPU 可写、CRTC 可扫** 的简单缓冲设计：

```text

CREATE_DUMB → 得到 handle、pitch、size

MAP_DUMB    → mmap 偏移

用户态写像素

ADDFB       → framebuffer

```

Rockchip：
  
```text

rockchip_drm_driver.dumb_create = rockchip_gem_dumb_create

```

Qt `QT_QPA_FB_DRM=1`、`modetest -s` 都走这条路。
### 6.2 GEM

更通用的缓冲对象：refcount、mmap、与 FB 关联、可 export 成 **dma-buf** 给别的设备（GPU/视频）。  
显示只关心：最终 FB 指向的物理/IOVA 地址能否被扫描器读到。
### 6.3 IOMMU

若 CRTC（VOP）通过 IOMMU 访问内存：

- 驱动要把 GEM 映射进 IOMMU 域，plane 写的是 **IOVA**  
- 域建失败或 DT 关掉 IOMMU，却仍按“有 IOMMU”编程 → 常见 **背光亮、黑屏**

本仓库：`rockchip_drm_init_iommu()` + DT `vop`/`vop_mmu`。

---
## 7. 与 fbdev 的关系

|     | fbdev                                 | DRM/KMS              |
| --- | ------------------------------------- | -------------------- |
| 节点  | `/dev/fb0`                            | `/dev/dri/card0`     |
| 模型  | 单缓冲为主                                 | plane/crtc/connector |
| 本板  | **故意 skip** `rockchip_drm_fbdev_init` | 唯一出图路径               |

DRM 可可选提供 fbdev 兼容层（`drm_fb_helper`）；OOT 与内置 fbcon ABI 冲突故关闭。

**插件名叫 `linuxfb` ≠ 必须有 `/dev/fb0`**；加 `QT_QPA_FB_DRM=1` 即走 DRM。

---
## 8. 通用（核心文件）

```text

drivers/gpu/drm/

  drm_drv.c          # drm_device / minor 生命周期

  drm_file.c         # open/release、drm_file

  drm_ioctl.c        # ioctl 入口

  drm_mode_config.c  # mode_config

  drm_crtc.c / plane.c / encoder.c / connector.c

  drm_atomic*.c      # atomic

  drm_gem*.c         # GEM

  drm_fb_helper.c    # 可选 fbdev 仿真

  drm_of.c           # DT helper

```

芯片相关在 `drivers/gpu/drm/<vendor>/`（本仓库 Rockchip）。

---
# 第二部分：结合源码的三层（Rockchip 实战）

总览：

```text

① OF compatible     驱动认硬件、probe

② component         多设备合成 drm_device → /dev/dri/card0

③ of_graph + KMS    管道拓扑 + dumb/atomic 出图

```

对应启动：`insmod rockchipdrm.ko`（及依赖 ko）+ 显示 DTB okay。

---
## 第一层：驱动注册 + DT compatible 匹配

### 1.1 模块入口注册两类 platform_driver

```text

文件：rockchip_drm_drv.c → rockchip_drm_init()

```


```text

ADD_ROCKCHIP_SUB_DRIVER(vop2_platform_driver, ...)

ADD_ROCKCHIP_SUB_DRIVER(rockchip_lvds_driver, ...)

...

platform_register_drivers(rockchip_sub_drivers, ...)   // 子设备

platform_driver_register(&rockchip_drm_platform_driver) // master

```

| 角色     | name          | compatible                                     |
| ------ | ------------- | ---------------------------------------------- |
| Master | rockchip-drm  | `rockchip,display-subsystem`                   |
| 子      | rockchip-vop2 | `rockchip,rk3562-vop`（见 `rockchip_vop2_reg.c`） |
| 子      | rockchip-lvds | `rockchip,rk3562-lvds`（见 `rockchip_lvds.c`）    |

### 1.2 匹配算法（总线级）

```text

DT status=okay 节点 → platform_device

platform_driver.of_match_table 中 compatible 命中 → 调用 .probe

```
  
板级 DT：

```text

kx_rk3562-display.dtsi

  display-subsystem { compatible = "rockchip,display-subsystem"; ports = <&vop_out>; }

  vop@ff400000 { compatible = "rockchip,rk3562-vop"; ... vop_out: ports { ... } }

  lvds { compatible = "rockchip,rk3562-lvds"; ... }

kx_rk3562-core-ddr4-v10.dts

  &vop / &lvds / &vop_mmu / panel-lvds → okay

```

`status=disabled` → 不创建有效设备 → 不会 probe（故 HDMI/DSI 可关）。
### 1.3 第一层结束状态

- 各 `probe` 已进入（或 defer）  
- **尚无**完整可用的 modeset 管道（子设备此时多半只 `component_add`）  
- Master 的 `ports` 属性：`of_parse_phandle(np, "ports", i)` 取到的是 **`&vop_out` 指向的节点**（VOP 下 `vop_out: ports`），用于确认有可用 VOP、以及是否看 `iommus`
---
## 第二层：component 拼装成一颗 DRM 设备

### 2.1 子 probe：报名

```text

rockchip_vop2_reg.c  vop2_probe()

  → component_add(dev, &vop2_component_ops)   // .bind = vop2_bind

rockchip_lvds.c      probe 末尾

  → component_add(dev, &rockchip_lvds_component_ops)  // .bind = rockchip_lvds_bind

```

含义：零件就绪，等 master。
### 2.2 Master probe：名单 + 当 master

```text

rockchip_drm_platform_probe()

  rockchip_drm_platform_of_probe()   // ports / iommu 策略

  rockchip_drm_match_add()

    // 遍历 rockchip_sub_drivers[]，platform_find_device_by_driver

    // component_match_add(..., compare_dev, d)

  component_master_add_with_match(dev, &rockchip_drm_ops, match)

    // rockchip_drm_ops.bind = rockchip_drm_bind

```
  
名单来源：已挂在 rockchip 子驱动上的 device（不是再扫一遍 graph 列零件）。
### 2.3 齐活：`rockchip_drm_bind`

```text

rockchip_drm_drv.c → rockchip_drm_bind()

  drm_dev_alloc(&rockchip_drm_driver, dev)

  drmm_mode_config_init / rockchip_drm_mode_config_init

  component_bind_all(dev, drm_dev)     // 同一 drm_dev 传给各零件

    → vop2_bind()                      // crtc + plane

    → rockchip_lvds_bind()             // encoder + connector + panel

  rockchip_drm_init_iommu()

  skip rockchip_drm_fbdev_init()       // 本仓库 OOT 补丁

  drm_dev_register()                   // → /dev/dri/card0

```

`vop2_bind`（`rockchip_drm_vop2.c`）：`of_device_get_match_data()` 取 `rk3562_vop`，注册 plane/crtc，解析输出 ports。  

`rockchip_lvds_bind`：`drm_of_find_panel_or_bridge`、`drm_encoder_init`、`drm_connector_init`、`attach_encoder`。

dmesg：`bound ...vop` / `bound ...lvds`、`Initialized rockchip 3.0.0`。
### 2.4 第二层结束状态


```text

/dev/dri/card0

mode_config 上已有：crtc、plane、encoder、connector

用户态可 open / 枚举资源

尚未必有“正在扫的那帧”（需第三层 commit）

```

---
## 第三层：of_graph 拓扑 + KMS 出图
### 3.1 Graph 把管道接好

DT：
  
```text

vp0_out_lvds  ←remote-endpoint→  lvds_in_vp0

lvds_out_panel ←→ panel_in_lvds

```

代码：

```text

rockchip_drm_of_find_possible_crtcs()  // rockchip_drm_drv.c

  for_each_endpoint_of_node

    of_graph_get_remote_port

    drm_of_crtc_port_mask → encoder->possible_crtcs

rockchip_lvds_bind

  drm_of_find_panel_or_bridge → simple-panel

  panel timings / bus-format → connector modes

```

板级还限定：

```text

&vp0 {

  rockchip,plane-mask = <ESMART0..3>;

  rockchip,primary-plane = <ESMART0>;

}

```

逻辑管道：

```text

Plane(ESMART0) → CRTC(VP0) → Encoder(LVDS) → Connector(LVDS-1) → Panel

```

### 3.2 用户态一次出图（dumb + modeset/atomic）

```text

open(/dev/dri/card0)

CREATE_DUMB + mmap          → 可写缓冲

填像素（Qt / modetest）

ADDFB                       → fb_id

SETCRTC 或 Atomic commit

  → plane 绑定 fb

  → crtc 设 mode（1280x800 @ panel clock）

  → encoder/connector enable

```

内核侧（概念调用链）：

```text

atomic_commit

  → plane atomic_update     // 写扫出地址（IOVA）、格式

  → crtc atomic_enable      // dclk、timing（vop2）

  → encoder enable          // rockchip_lvds_encoder_enable

       drm_panel_prepare → LVDS/PHY on → drm_panel_enable（背光）

```

之后 VOP 按 dclk 持续扫 FB → LVDS → 屏。
### 3.3 与本板用户态的对应

| 用户态                              | 路径                                            |
| -------------------------------- | --------------------------------------------- |
| modetest                         | 直接 libdrm KMS                                 |
| Qt `linuxfb` + `QT_QPA_FB_DRM=1` | `QLinuxFbDrmScreen`：dumb + modeset，**无 Mali** |
| Qt eglfs                         | 要 GBM/Mali/`dma_heap`，本板当前不可用                 |

---

# 第三部分：吃透框架的加深专题（三层之后）


三层是**主线**。暂未拓展更深层细节，层次如下：
## A. Atomic 细节

- `drm_atomic_state` 生命周期  

- `atomic_check` vs `atomic_commit` / `commit_tail`  

- nonblocking、vblank、out-fence  

- 读：`drm_atomic_helper.c` + vop2 的 `atomic_*` 回调  
## B. GEM / 地址

- `rockchip_gem_dumb_create` 分配路径  

- FB → GEM → DMA/IOVA  

- `rockchip_drm_init_iommu`、attach device  

- 黑屏优先查：iommu=0？domain？map？  

## C. VOP2 / LVDS 硬件语义


- VP、plane 类型、win 寄存器（`rockchip_vop2_reg.c` 数据表）  

- `bus-format` SPWG/JEIDA 对通道映射（花屏）  

- dclk 与 `clock-frequency`  

## D. DRM 核心边角

- master/auth、lease、render 节点  

- writeback connector、damage、async flip  

- fbdev helper（本板关闭，仅作对比）  

## E. 用户态栈

- libdrm API  

- Qt QPA：`linuxfb` DRM backend vs `eglfs_kms`  

- Weston drm-backend（公版路径）  

---
# 第四部分：路径

## 阶段 1：对象与 ioctl（通用）

1. 默写 plane/crtc/encoder/connector/fb/gem 关系图  

2. 板上 `modetest -c` 对照每个对象  

3. 读 `drm_file.c` open/release；`drm_ioctl` 入口扫一眼  

## 阶段 2：三层源码（本仓库）

1. `rockchip_drm_init` → 两张 register  

2. `vop2_probe` / lvds `component_add`  

3. `rockchip_drm_platform_probe` → `match_add` → `rockchip_drm_bind` → `component_bind_all`  

4. DT graph + `possible_crtcs` + panel  

5. 自己画：从 `insmod` 到 `card0` 的时序图  

## 阶段 3：出图与排障

1. 跟一次 `CREATE_DUMB`→`ADDFB`→commit（可结合 ftrace/动态 printk）  

2. 对比：IOMMU on/off、错误 bus-format  

3. Qt：确认 `FB_DRM=1`、BR loader，与 `/dev/fb*` 无关  

## 阶段 4：加深（可选）

1. 精读一个 `vop2_crtc_atomic_enable`  

2. 精读 `rockchip_lvds_encoder_enable`  

3. 对照 mainline Documentation：`Documentation/gpu/drm-kms.rst`（若树内有）  

---
# 第五部分：速查表

## 关键路径

```text

通用核心：

  kernel-5.10/drivers/gpu/drm/drm_*.c

Rockchip：

  rockchip_drm_drv.c      # master、match、bind、iommu、driver 结构体

  rockchip_vop2_reg.c     # compatible / component_add

  rockchip_drm_vop2.c     # vop2_bind、atomic、扫出

  rockchip_lvds.c         # lvds bind/enable、panel

  rockchip_drm_gem.c      # dumb/gem（文件名以树内为准）

DT：

  kx/rockchip/kx_rk3562-display.dtsi

  kx/rockchip/kx_rk3562-core-ddr4-v10.dts

用户态：

  czj/qt-drm-smoke/

  device/rockchip/common/tools/aarch64/modetest

```

## 三层总结

| 层 | 一句话 |
|----|--------|
| ① | `compatible` 决定哪个 `platform_driver.probe` 服务哪个 DT 节点 |
| ② | `component` 把多个 probe 过的设备 bind 进同一 `drm_device` 并 `register` |
| ③ | `of_graph` 定拓扑；dumb+atomic 把 FB 交给 VOP 扫描、LVDS 送出 |

  

## 本板特例

- OOT：226 驱动源码 + 198 vermagic/头文件 ABI  

- `file->pid = NULL`；skip fbdev  

- 显示必须 DT 开 IOMMU（曾关过导致黑屏）  

- Qt：`linuxfb` + `FB_DRM=1` + BR `ld-linux` 侧载  

---
# 结语

- **DRM 通用核心**：对象模型 + mode_config + atomic + GEM/dumb + 节点/`drm_file`。  

- **三层源码**：把通用模型落到 Rockchip「怎么挂上、怎么出图」。  

- **完全拿下**：主线吃透三层后，按 A–E 专题推进；用板上现象验证，而不是只通读 `vop2.c` 一万行。

文档版本与仓库移植状态同步；细节补丁以源码与 `DRM-OOT-PORTING.md` 为准。