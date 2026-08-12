# 一、背景情况
## 约束：多芯系统因验签文件，不能改内核

多核（SMIOS）板上跑的是已验签内核：

| 项      | 现状                                          |
| ------ | ------------------------------------------- |
| 内核     | `5.10.198 #236`，Image / U-Boot 不能刷          |
| DRM    | `# CONFIG_DRM is not set`，无 `/dev/dri`      |
| 现网 DTB | 显示子系统被裁掉（无 vop/lvds/panel）                  |
| 可改项    | DTB 不验签；可加载外部 `.ko`；rootfs 可加用户态            |
| 目标硬件   | RK3562J CORE DDR4 V10，LVDS 屏 GV101 1280×800 |
| 业务目标   | 最终要跑 Qt UI（原开发板走 Weston/`wayland-egl`）      |

## 对照：开发板怎么出画

SDK Buildroot 开发板是：
```
内核 CONFIG_DRM=y + rockchip-drm 编进 Image
  + 完整 DTB（vop/lvds/panel）
  → /dev/dri/card0
  → Weston (drm-backend) → Qt wayland-egl → DSP
```

## 方案演进（认识过程）

1. 无 DRM 就只能自研 FB `.ko` + `linuxfb`。
2. 把 DRM 整栈拆成 OOT `.ko`，按依赖 `insmod`。
3. 实际落地：在「不换验签 Image」前提下，做成了可换 DTB + OOT DRM 模块包（`drm-kos-198`），并最终用 Qt 冒烟验证出画。
#  二、DRM 打通链路的整体流程

```
[1] 换 DTB（补显示节点）
      vop + vop_mmu + lvds + panel-lvds + route + backlight …
        ↓
[2] 用 198 对齐的方式编译 OOT DRM 栈
      drm_deps → fb helpers → drm → kms_helper → panel/phy → rockchipdrm
        ↓
[3] 板上 insmod_drm.sh 按序加载
        ↓
[4] 出现 /dev/dri/card0，rockchip 3.0.0 初始化
      绑定 VOP2 + LVDS，crtc=1
        ↓
[5] modetest 做 modeset + SMPTE 彩条
        ↓
[6] Qt：linuxfb + QT_QPA_FB_DRM=1（DRM dumb，不靠 Mali）
      → 蓝底白字「Qt DRM smoke OK」
```
硬件数据路径：
```
用户态缓冲（dumb / Qt）
  → DRM/KMS（rockchipdrm）
  → VOP2（扫显存，经 IOMMU）
  → LVDS TX
  → panel（1280×800）
  → 背光
```

# 三、踩过的坑 & 做了什么
## 阶段 A：模块根本装不上 / 不跑 init（226 → 198）

| 坑                            | 现象                                      | 处理                                                                                       |
| ---------------------------- | --------------------------------------- | ---------------------------------------------------------------------------------------- |
| vermagic                     | SDK 编出 `5.10.226`，板子是 `5.10.198`        | `Makefile` 里 `SUBLEVEL=198`，按 198 宣称 vermagic                                            |
| `struct module` / 头文件 ABI    | 模块装上但 `module_init` 不跑或行为异常             | 一批头文件从 mainline 198 对齐，原 226 留 `*.bak-226`（`module.h`、`device.h`、`of.h`、`gfp.h`、dma-* 等） |
| `gfp.h` / `__GFP_BITS_SHIFT` | connector 分配异常，`ida_alloc` → -28 ENOSPC | 换成 mainline 198 的 `gfp.h`                                                                |
## 阶段 B：DRM 起来了，但 probe / 显示异常

|坑|现象|处理|
|---|---|---|
|无显示 DT|有 ko 也 probe 不到硬件|做 KX 板级 DT（对齐 `stamp_kx`）：panel / vop / lvds / route / 背光|
|IOMMU deferred / `iommu_domain_alloc` 失败|早期 -12 ENOMEM，bind 卡住|驱动里对 `iommu_domain_alloc` 做 fallback 非 IOMMU；DT 上曾暂时去掉 `iommus`、关 `vop_mmu`|
|非 IOMMU 黑屏|modeset/背光都有，无图（VOP 扫错地址）|DT 恢复 IOMMU（对齐 stamp_kx），保证扫到正确显存|
|plane 分配|RK3562 VOP2 需明确 plane|DT 加 `rockchip,plane-mask`（如 `0x30c`，ESMART 等）|
|fbdev/fbcon ABI|OOT 的 `fb_info` 与内置 fbcon 不匹配 → `fbcon_startup` panic|跳过 `rockchip_drm_fbdev_init`（无 `/dev/fb0`，Qt 改走 DRM）|
## 阶段 C：modetest

|坑|现象|处理|
|---|---|---|
|无 PATH 里的 modetest|找不到命令|用 SDK：`device/rockchip/common/tools/aarch64/modetest`，板上 `./modetest`|
|`modetest -c` 关 fd panic|栈：`drm_release → drm_file_free → put_pid`|OOT 里 `task_pid(current)` 按编译期 `task_struct` 内联，与现网 #236 布局不一致 → `file->pid` 野指针。`drm_file.c` 里 `file->pid = NULL`，跳过 pid 跟踪|
|花屏|有 SMPTE 彩条但色块/黑边错位|链路已通；属 JEIDA/VESA（bus-format）或 porch 时序，不是断链。DT 里 panel 仍多为 `SPWG`，JEIDA 尚未最终落地验证|
## 阶段 D：Qt 验证（显示已通之后）

| 坑                 | 现象                             | 处理                                                                                                                                 |
| ----------------- | ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| glibc             | 板 2.34，BR Qt/DSP 要 2.38        | 用 BR 的 `ld-linux` + `--library-path` 侧载，不换 rootfs                                                                                  |
| `LD_LIBRARY_PATH` | 带着 BR libc 跑系统 `ls` → segfault | 脚本禁止 export，只给 loader 传 library-path                                                                                               |
| 缺库                | 如 `libpcre2-8.so.0`            | 补进 `qt-drm-smoke` runtime                                                                                                          |
| `eglfs`           | `Could not create GBM device`  | `libgbm`/`EGL` 是 Mali 壳；`renderD128` 来自 rockchip DRM 的 `DRIVER_RENDER`，不是 Mali；板无 `/dev/mali`，且 `# CONFIG_DMABUF_HEAPS is not set` |
| 无 linuxfb 插件      | BR Qt `LinuxFB=no`             | 手编 `libqlinuxfb.so`，`QT_QPA_PLATFORM=linuxfb` + `QT_QPA_FB_DRM=1`（DRM dumb）                                                        |
| 字体/xkb            | Fontconfig/xkb 报错              | 把 fonts、xkb、libinput quirks 打进 runtime                                                                                             |
# 四、改了哪些代码 / 文件

## 1. 内核树（编译 OOT 用，不刷进验签 Image）

| 文件                                                         | 改动要点                                                                                                           |
| ---------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `kernel-5.10/Makefile`                                     | `SUBLEVEL=198`，对齐 vermagic                                                                                     |
| `kernel-5.10/include/linux/*.h`（多份）                        | 用 198 布局替换；原 226 备份为 `*.bak-226`（含 `module.h`、`gfp.h`、`device.h`、`of.h`、dma 相关等）                               |
| `kernel-5.10/drivers/gpu/drm/drm_file.c`                   | `file->pid = NULL`，规避 `put_pid` oops                                                                           |
| `kernel-5.10/drivers/gpu/drm/rockchip/rockchip_drm_drv.c`  | ① `iommu_domain_alloc` 失败则 fallback 非 IOMMU；② skip `rockchip_drm_fbdev_init`；③ bind 日志（crtcs/connectors/iommu） |
| `kernel-5.10/drivers/gpu/drm/rockchip/rockchip_drm_vop2.c` | writeback 销毁等加固（避免 unbind                                                                                      |
## 2. 设备树（要上板替换的 DTB）

|文件|改动要点|
|---|---|
|`kx/rockchip/kx_rk3562-display.dtsi`|显示子系统：vop、`vop_mmu`、`iommus`、lvds、route|
|`kx/rockchip/kx_rk3562-core-ddr4-v10.dts`|panel-lvds（1280×800）、背光、`&lvds`/`&vop` okay；恢复 IOMMU；`rockchip,plane-mask`；`bus-format` 现为 `..._SPWG`（花屏待试 JEIDA）|
|产物|`kx_rk3562-core-ddr4-v10.dtb` / `kx_kernel-dtb.img` 一类|

## 3. OOT 模块交付

|路径|作用|
|---|---|
|`czj/drm-kos-198/*.ko`|DRM 整栈 + panel/phy/fb helper 等|
|`czj/drm-kos-198/insmod_drm.sh`|依赖顺序加载|
|`czj/drm-deps/`、`czj/fb-defio/`、`czj/phy-mipi-dphy/` 等|拆出的依赖模块构建辅助|

## 4. Qt 侧（用户态，不改内核）

| 路径                  | 作用                                                   |
| ------------------- | ---------------------------------------------------- |
| `czj/qt-drm-smoke/` | 冒烟程序 + BR runtime + 自建 `libqlinuxfb.so`              |
| `run_qt_smoke.sh`   | `linuxfb` + `QT_QPA_FB_DRM=1`，禁止乱设 `LD_LIBRARY_PATH` |

# 五、程序详解
##  总览

```
DTB 描述硬件拓扑
    ↓ of_platform 枚举
panel-simple / phy / vop2 / lvds 各 platform_driver.probe
    ↓ component 框架    ★★★★★
display-subsystem (rockchip-drm master) bind 所有子设备
    ↓ drm_dev_register
/dev/dri/card0
    ↓ 用户态 ioctl（modetest / Qt linuxfb）
dumb buffer → FB → plane/crtc/connector atomic commit
    ↓
VOP2 扫显存(+IOMMU) → LVDS encoder → panel → 背光
```
## 设备树

Panel（时序 + 格式 + 背光）
```
panel-lvds {
	compatible = "simple-panel";
	...
	bus-format = <MEDIA_BUS_FMT_RGB888_1X7X4_SPWG>;
	display-timings {
		...
		clock-frequency = <56000000>;
		hactive = <1280>;
		vactive = <800>;
		...
	};
	ports {
		panel_in_lvds: endpoint {
			remote-endpoint = <&lvds_out_panel>;
		};
	};
};
```
VOP ↔ LVDS 图（graph）
```
vop: vop@ff400000 {
	compatible = "rockchip,rk3562-vop";
	iommus = <&vop_mmu>;
	...
	vp0_out_lvds: endpoint@2 {
		remote-endpoint = <&lvds_in_vp0>;
	};
};

lvds: lvds {
	compatible = "rockchip,rk3562-lvds";
	lvds_in_vp0: endpoint@0 {
		remote-endpoint = <&vp0_out_lvds>;
	};
};
```
使能+限定 plane
```
&vop { status = "okay"; };
&vp0 {
	rockchip,plane-mask = <(... ESMART0..3 ...)>;
	rockchip,primary-plane = <ROCKCHIP_VOP2_ESMART0>;
};
&vop_mmu { status = "okay"; };
```

`compatible` 决定哪个 driver 匹配；`remote-endpoint` 决定 DRM 里 encoder↔crtc↔connector 怎么连；`bus-format`/timings 决定 LVDS 打包和 mode；`iommus`/`vop_mmu` 决定 VOP 能否正确扫 GEM 缓冲。

## 模块加载：把缺的 DRM 栈补进「没开 CONFIG_DRM」的核

验签核里没有 DRM 符号，所以整栈是 OOT `.ko`。顺序：
```
insmod drm_deps.ko
...
insmod drm.ko
insmod drm_kms_helper.ko
insmod panel-simple.ko
insmod phy_mipi_dphy.ko
insmod phy-rockchip-inno-dsidphy.ko
insmod rockchipdrm.ko
```
对应关系：

| ko                  | 代码角色                                           |
| ------------------- | ---------------------------------------------- |
| `drm.ko`            | DRM 核心：`drm_file`、ioctl、GEM、minor→`/dev/dri/*` |
| `drm_kms_helper.ko` | atomic helper、fb helper（跳过了 fbdev）             |
| `panel-simple.ko`   | 匹配 `compatible = "simple-panel"`               |
| phy                 | LVDS 用的 video phy                              |
| `rockchipdrm.ko`    | VOP2 + LVDS + master `display-subsystem`       |

`rockchipdrm` 初始化时注册子驱动 + master：
```
static int __init rockchip_drm_init(void)
{
	ADD_ROCKCHIP_SUB_DRIVER(vop2_platform_driver, ...);
	ADD_ROCKCHIP_SUB_DRIVER(rockchip_lvds_driver, ...);
	...
	platform_register_drivers(...);
	platform_driver_register(&rockchip_drm_platform_driver); // display-subsystem
}
```
Master 只认：
```
{ .compatible = "rockchip,display-subsystem", },
```
LVDS认：
```
compatible = "rockchip,rk3562-lvds"  → rockchip_lvds.c
```

## Bind：拼出一颗完整的 `drm_device

Master probe 后走 `component_bind_all`，把 VOP、LVDS 等绑进同一个 DRM 设备：
```
ret = component_bind_all(dev, drm_dev);
...
DRM_DEV_INFO(..., "bind: crtcs=%d connectors=%d iommu=%d\n", ...);
...
ret = rockchip_drm_init_iommu(drm_dev);
...
/* 故意不调 rockchip_drm_fbdev_init —— OOT/fbcon ABI 会 panic */
DRM_DEV_INFO(dev, "skip rockchip_drm_fbdev_init (OOT/fbcon ABI)\n");
ret = drm_dev_register(drm_dev, 0);  // → /dev/dri/card0
```
这一步之后用户态看到的对象大致是：
```
crtc (VP0 / VOP2)
  ↑ primary plane (ESMART0, 由 plane-mask 限定)
encoder (LVDS)
connector (LVDS-1) ← panel-simple 提供 mode 列表
```
IOMMU（扫缓冲关键）：
```
private->domain = iommu_domain_alloc(...);
if (!private->domain) {
	/* fallback non-iommu —— 可 bind，但常黑屏 */
	is_support_iommu = false;
	return 0;
}
```
## 打开 `/dev/dri/card0`：`drm_file` 与你们的 pid 补丁

用户态 `open("/dev/dri/card0")` → DRM 分配 `struct drm_file`：
```
struct drm_file *drm_file_alloc(...)
{
	...
	/* OOT：不调 task_pid(current)，避免与现网 task_struct 布局不一致 */
	file->pid = NULL;
	file->minor = minor;
```
## Modeset 路径（modetest / Qt 共用 KMS）

用户态典型 ioctl 链：
```
DRM_IOCTL_MODE_CREATE_DUMB   → 分配扫描缓冲（GEM）
DRM_IOCTL_MODE_MAP_DUMB      → mmap 写像素
DRM_IOCTL_MODE_ADDFB2        → 登记 framebuffer id
DRM_IOCTL_MODE_ATOMIC / SETCRTC / PAGE_FLIP
  → atomic commit
      → crtc enable：VOP 配 dclk、timing、plane 地址
      → encoder enable：LVDS + panel
```
LVDS encoder enable（面板上电顺序） 在：
```
rockchip_lvds_encoder_enable()
{
	drm_panel_prepare(panel);   // enable-gpios、延时等
	rockchip_lvds_enable(lvds); // 配 LVDS/PHY、按 bus-format 打包
	drm_panel_enable(panel);    // 背光等
}
```
`bus-format = SPWG/JEIDA` 在这里影响通道映射——花屏多半卡在这一层，不是 DRM 断链。
VOP2（`rockchip_drm_vop2.c`）在 atomic enable 里：
- 根据 mode 设 `dclk`（你们日志里约 56MHz，对齐 DT `clock-frequency`）
- 把 primary plane 的 GEM IOVA/物理地址写进展寄存器
- 经 `vop_mmu` 做地址转换后扫出 RGB → LVDS
## Qt 冒烟：不走 Mali，走 linuxfb + DRM dumb

开发板 Qt 是 `wayland-egl`（要 GBM/Mali）。多核板没有可用 GPU 用户态路径，所以：
```
qt_smoke (QLabel 全屏)
  → QPA: libqlinuxfb.so
  → QT_QPA_FB_DRM=1
  → QLinuxFbDrmScreen
       open /dev/dri/card0
       CREATE_DUMB ×2（双缓冲）
       AddFB、setMode
       软件把 QImage 画进 mmap 缓冲
       page-flip / setcrtc 提交
```
## 对照：每层看什么代码 / 出问题看哪

|层|关键代码|失败时像什么|
|---|---|---|
|DT|`kx_rk3562-*.dts*`|无设备、probe 不到|
|加载|`insmod_drm.sh` + vermagic/头文件|insmod 失败或 init 不跑|
|Master bind|`rockchip_drm_drv.c` bind/iommu/fbdev|无 card0 / iommu=0 黑屏 / fbcon panic|
|文件生命周期|`drm_file.c`|close 时 `put_pid` oops|
|Encoder/Panel|`rockchip_lvds.c` + `panel-simple`|有 mode 无输出 / 花屏|
|CRTC/Plane|`rockchip_drm_vop2.c`|modeset 失败、花屏、黑屏|
|Qt|`libqlinuxfb.so` + dumb ioctl|有 DRM 但 Qt 不出画；eglfs 则卡在 GBM|

核心一句话：DT 定义拓扑，OOT `rockchipdrm` 用 component 绑成 KMS，用户态只对 `/dev/dri/card0` 做 dumb+atomic；VOP+IOMMU 扫缓冲，LVDS+panel 负责电气与格式。 Qt 冒烟只是这条 KMS 链路上的一个客户端。

## 代码调用流程

```
flowchart TB
  DT["DTB: display-subsystem / vop / vop_mmu / lvds / panel-lvds"]
  INS["insmod: drm.ko → kms_helper → panel/phy → rockchipdrm.ko"]
  INIT["rockchip_drm_init()\n注册 vop2/lvds/master"]
  BIND["rockchip_drm_bind()\ncomponent_bind_all\ninit_iommu / skip fbdev\ndrm_dev_register"]
  DEV["/dev/dri/card0"]
  OPEN["open → drm_file_alloc\nfile->pid=NULL"]
  IOCTL["CREATE_DUMB / ADDFB / ATOMIC"]
  VOP["vop2 atomic_enable\nplane+dclk+IOMMU"]
  LVDS["lvds encoder_enable\npanel prepare/enable"]
  PANEL["LVDS panel + backlight"]

  DT --> INS --> INIT --> BIND --> DEV
  DEV --> OPEN --> IOCTL --> VOP --> LVDS --> PANEL

```




# 六、改了哪些文件的代码
## 内核树 `kernel-5.10`（只用于编 OOT，不刷进验签 Image）
### 版本与配置

| 文件                                  | 改动                                                               |
| ----------------------------------- | ---------------------------------------------------------------- |
| `kernel-5.10/Makefile`              | `SUBLEVEL = 198`（vermagic）                                       |
| `kernel-5.10/board-5.10.198.config` | 从板子 `config.gz` 拷入；再 olddefconfig + 打开 DRM=m 等（以实际 `.config` 为准） |
| `kernel-5.10/.config`               | 本地编译用配置（由 board config 演化）                                       |

### 头文件 ABI 替换（当前文件 ← 198 布局；原 226 为 `*.bak-226`）

目录：`kernel-5.10/include/linux/`
| 当前使用的头文件 | 备份 |
|------------------|------|
| `module.h` | `module.h.bak-226` |
| `gfp.h` | `gfp.h.bak-226` |
| `device.h` | `device.h.bak-226` |
| `of.h` / `of_device.h` | 对应 `.bak-226` |
| `dma-buf.h` / `dma-fence.h` | 对应 `.bak-226` |
| `mm.h` / `mm_types.h` | 对应 `.bak-226` |
| `iommu.h` | `iommu.h.bak-226` |
| `platform_device.h` / `mod_devicetable.h` | 对应 `.bak-226` |
| `phy.h` / `pwm.h` / `timer.h` / `hrtimer.h` / `workqueue.h` | 对应 `.bak-226` |
| `clk-provider.h` / `fwnode.h` / `kernfs.h` / `kobject.h` | 对应 `.bak-226` |
> 共约 21 个 `*.bak-226`。其中 **`gfp.h`、`module.h`** 对“能装上、能 init、connector 不 ENOSPC”最关键。
### DRM / Rockchip 驱动补丁

| 文件                                                         | 改动要点                                                                                                                                             |
| ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `kernel-5.10/drivers/gpu/drm/drm_file.c`                   | `drm_file_alloc()` 中 **`file->pid = NULL`**，避免 `drm_release`→`put_pid` oops                                                                      |
| `kernel-5.10/drivers/gpu/drm/rockchip/rockchip_drm_drv.c`  | ① `rockchip_drm_init_iommu()`：`iommu_domain_alloc` 失败则 fallback 非 IOMMU；② **skip `rockchip_drm_fbdev_init`**；③ bind 时打印 `crtcs/connectors/iommu` |
| `kernel-5.10/drivers/gpu/drm/rockchip/rockchip_drm_vop2.c` | writeback 等路径加固（避免异常 unbind）；与 RK3562 VOP2 出图相关                                                                                                  |

### 设备树（要烧进/替换启动 DTB 的源）

| 文件                                        | 改动要点                                                                                                                                             |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `kx/rockchip/kx_rk3562-display.dtsi`      | 显示子系统：vop、`iommus`→`vop_mmu`、lvds、ports/route、video_phy 等                                                                                        |
| `kx/rockchip/kx_rk3562-core-ddr4-v10.dts` | `panel-lvds` timings/`bus-format`；`&vop`/`&vop_mmu`/`&lvds` okay；`rockchip,plane-mask` / `primary-plane`；USB host；注释说明本板 **USB 触摸**、关 I2C gt1x 等 |
| 产物                                        | `kx_rk3562-core-ddr4-v10.dtb`（及打包进 `kernel-dtb.img` 的流程按项目习惯）                                                                                    |

### OOT （`czj/`）

| 路径                                | 说明                                                                         |
| --------------------------------- | -------------------------------------------------------------------------- |
| `czj/drm-kos-198/`                | 最终上板 ko 包 + `insmod_drm.sh` + checksum                                     |
| `czj/drm-kos-226/`                | 早期 226 vermagic 试验包                                                        |
| `czj/drm-deps/`                   | `drm_deps.ko` 及 shim 源码/`Makefile`                                         |
| `czj/fb-defio/` 等                 | fb_defio 等拆分构建（若存在）                                                        |
| `czj/qt-drm-smoke/`               | Qt 冒烟源码、自建 `libqlinuxfb.so`、BR runtime、`run_*.sh`、`qt-drm-smoke-rt.tar.gz` |
| `czj/qt-drm-smoke/src/main.cpp`   | 双按钮常驻冒烟                                                                    |
| `czj/qt-drm-smoke/main.cpp`       | 触摸坐标测试                                                                     |
| `czj/qt-drm-smoke/linuxfb-build/` | `libqlinuxfb.so` 编译中间产物                                                    |

### 明确未改 / 不能改

- 板上**验签** Kernel Image、U-Boot  
- 多核 rootfs 系统 glibc（不替换为 2.38；Qt 侧载运行）  
- 未启用 Mali / `DMABUF_HEAPS`（故 eglfs 不可用）
### 出图与 fb 的关系（避免误解）

- 当前 Qt：**不依赖 `/dev/fb*`**。  
- `QT_QPA_PLATFORM=linuxfb` + `QT_QPA_FB_DRM=1` → 走 **`/dev/dri/card0` DRM dumb**。  
- 无 fb 是 skip fbdev 的结果，属于设计选择。
## 怎么编译 & 怎么移植上板
### 环境

```text

SDK 根目录：
/home/ChenZhangjin/HZ-RK3562-LINUX_5.10_SDK_20250918_V1.2、

交叉编译（内核模块，示例）：
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-

Qt / 用户态（Buildroot host）：
buildroot/output/rockchip_kx_rk3562_sp/host/bin/aarch64-buildroot-linux-gnu-g++
sysroot: .../host/aarch64-buildroot-linux-gnu/sysroot

```
### 编译内核模块（DRM OOT）

```bash
cd /home/ChenZhangjin/HZ-RK3562-LINUX_5.10_SDK_20250918_V1.2/kernel-5.10

# 若需从板级 config 重建：

# cp board-5.10.198.config .config

# make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- olddefconfig

# 确认 CONFIG_DRM=m、ROCKCHIP DRM=m 等；SUBLEVEL 已为 198

export ARCH=arm64

export CROSS_COMPILE=aarch64-linux-gnu-

# 先保证内核树已 prepare（按你们平时模块编译习惯）

# make modules_prepare   # 或完整 modules 一次生成 Module.symvers

# 编 DRM / rockchip / 相关 phy 等（示例）

make -j$(nproc) M=drivers/gpu/drm modules

make -j$(nproc) M=drivers/gpu/drm/rockchip modules

# panel、phy 等按实际 Kconfig 路径继续 make M=...

# 辅助 deps（示例）

make -C $PWD M=../czj/drm-deps modules

```
将需要的 `.ko` 收集到：

```bash
mkdir -p ../czj/drm-kos-198

# cp 所需 ko → czj/drm-kos-198/

# 保持/更新 insmod_drm.sh
```

检查 vermagic：
```bash

modinfo -F vermagic czj/drm-kos-198/drm.ko

# 应为：5.10.198 ...

```
### 编译设备树

```bash

cd /home/ChenZhangjin/HZ-RK3562-LINUX_5.10_SDK_20250918_V1.2/kx/rockchip

# 按仓库 Makefile / 项目脚本编译 kx_rk3562-core-ddr4-v10.dtb

# 或放入 kernel 的 dtb 编译流程后打包 kernel-dtb.img

make  或者 ./build_kx_kernel_dtb.sh # 若该目录有现成规则

```
按板子启动方式替换 kernel-dtb.img
### 编译 Qt 冒烟 / 触摸程序

```bash

HOST=/home/ChenZhangjin/HZ-RK3562-LINUX_5.10_SDK_20250918_V1.2/buildroot/output/rockchip_kx_rk3562_sp/host

SYSROOT=$HOST/aarch64-buildroot-linux-gnu/sysroot

CXX=$HOST/bin/aarch64-buildroot-linux-gnu-g++

OUT=/home/ChenZhangjin/HZ-RK3562-LINUX_5.10_SDK_20250918_V1.2/czj/qt-drm-smoke

$CXX -O2 -o $OUT/rt/bin/touch_test $OUT/main.cpp \

  --sysroot=$SYSROOT \

  -I$SYSROOT/usr/include/qt5 \

  -I$SYSROOT/usr/include/qt5/QtWidgets \

  -I$SYSROOT/usr/include/qt5/QtGui \

  -I$SYSROOT/usr/include/qt5/QtCore \

  -fPIC -DQT_WIDGETS_LIB -DQT_GUI_LIB -DQT_CORE_LIB \

  -L$SYSROOT/usr/lib -lQt5Widgets -lQt5Gui -lQt5Core \

  -lpthread -ldl

```

`libqlinuxfb.so` 需已在 `rt/usr/lib/qt/plugins/platforms/`（仓库内已构建过）。  

runtime 大包：`czj/qt-drm-smoke/qt-drm-smoke-rt.tar.gz`。
### 移植上板流程
#### 步骤 A：换显示 DTB 并重启
确认 DT 有 vop/lvds；**`vop_mmu` / iommus 为打开状态**。
```
ls /proc/device-tree/vop* /proc/device-tree/*lvds*
```
#### 步骤 B：加载 OOT DRM

```bash

# 主机

scp -r czj/drm-kos-198 root@板子:/root/

# 板子

cd /root/drm-kos-198

sh insmod_drm.sh

ls -l /dev/dri/card0

dmesg | grep -iE 'Initialized rockchip|vop|lvds'

```
#### 步骤 C：modetest 确认

```bash

# 拷贝 SDK aarch64 modetest 到板子当前目录

chmod +x ./modetest

./modetest -M rockchip -c

./modetest -M rockchip -s <connector_id>:1280x800

# 期望：屏上有彩条（花屏可暂忽略）

```
#### 步骤 D：Qt runtime + 程序

```bash

# 主机

scp czj/qt-drm-smoke/qt-drm-smoke-rt.tar.gz root@板子:/tmp/

# 或仅更新二进制：scp .../rt/bin/touch_test root@板子:/tmp/qt-drm-smoke/bin/

# 板子

mkdir -p /tmp/qt-drm-smoke && cd /tmp/qt-drm-smoke

tar xzf /tmp/qt-drm-smoke-rt.tar.gz   # 首次

./run_qt_smoke.sh      # 或 ./run_touch_test.sh

```

环境要点：
```text

QT_QPA_PLATFORM=linuxfb

QT_QPA_FB_DRM=1

QT_PLUGIN_PATH=.../usr/lib/qt/plugins

```
#### 步骤 E：触摸抽检

```bash

cat /proc/bus/input/devices | grep -A5 ILITEK

dd if=/dev/input/event0 bs=24 count=10 | hexdump -C   # 触摸应有 ABS_X/Y

```
## 附录：相关路径速查

```text

板级 config 备份：  kernel-5.10/board-5.10.198.config

OOT 模块包：        czj/drm-kos-198/

DT 源：             kx/rockchip/kx_rk3562-*.dts*

驱动补丁：          kernel-5.10/drivers/gpu/drm/drm_file.c

                    kernel-5.10/drivers/gpu/drm/rockchip/rockchip_drm_drv.c

Qt 冒烟：           czj/qt-drm-smoke/

modetest：          device/rockchip/common/tools/aarch64/modetest

BR 工具链：         buildroot/output/rockchip_kx_rk3562_sp/host/

```







