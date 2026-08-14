# OOT 挂 ko 故障案例集（真实串口日志）

来源：RK3562 多核板 **DRM / ALSA / 摄像头(GC05A2+rkaiq)** 外挂移植（现网 `5.10.198 #236`：`# CONFIG_DRM` / `SOUND` / `MEDIA_SUPPORT` is not set；SDK 驱动源码偏 5.10.226）。  
日志均为当时板上原样或与当时一致的关键打印。摄像头案例如无完整 Oops 栈，亦保留 **dmesg/工具输出中的关键行**。

**读日志口诀**：日志几乎从不写 `ABI mismatch`；它只告诉你 **缺符号 / 在哪崩 / 谁在加载**。ABI 是事后根据栈与编译环境推出来的。

---

## 通用排查步骤

1. **串口常开**，`insmod` 一个一个来（或脚本里看最后一个 `(O+)`）。
2. 重启前整段复制，搜：`Unknown symbol` / `Invalid module format` / `Oops` / `Unable to handle` / `Call trace` / `pc :` / `lr :`；相机另搜 `cma_alloc` / `stack-protector` / `REQBUFS`。
3. 看 `Comm:`（常是 `insmod` / `modetest` / `amixer` / `aplay` / `v4l2_grab` / `rkaiq_3A_server`）和 `Modules linked in: xxx(O+)`。
4. 对照修复：缺符号 → 补依赖 ko / 关特性；Oops → 对齐 ABI 头、加固错误路径、绕开现核没有的子系统；有设备无声/无卡 → 查 DT / MFD / mixer；相机无图/二次抓失败 → CMA / 3A 握手 / IOMMU。

---

## 案例 1：`bad vermagic` / `Invalid module format`

### 打印

```text
[  369.953269] drm_panel_orientation_quirks: bad vermagic: kernel tainted.
[  369.960699] Disabling lock debugging due to kernel taint
```

```text
[ 1478.973225] drm_kms_helper: version magic '5.10.226 SMP preempt mod_unload aarch64' should be '5.10.198 SMP preempt mod_unload aarch64'
insmod: ERROR: could not insert module drm_kms_helper.ko: Invalid module format
```

### 排查

| 线索 | 含义 |
|------|------|
| `5.10.226` vs `5.10.198` | 模块 vermagic 与正在跑的核不一致 |
| 无 `-f` → `Invalid module format` | 内核拒绝装载 |
| 有 `-f` → `bad vermagic` + tainted | 强制加载（需 `MODULE_FORCE_LOAD`），**不等于 ABI 已对齐** |

### 修复

- 短期：`insmod -f`（现核开了 force 才能用）。
- 正确：编译树 `SUBLEVEL=198`，用 `switch-kernel-abi.sh 198`，产出 vermagic=`5.10.198` 的 `drm-kos-198`。

---

## 案例 2：`Unknown symbol`（缺依赖，尚未 Oops）

### 打印 A — DRM 核心缺 HDMI / videomode / fence

```text
[  445.769581] drm: Unknown symbol hdmi_vendor_infoframe_init (err -2)
[  445.776849] drm: Unknown symbol hdmi_drm_infoframe_init (err -2)
[  445.783646] drm: Unknown symbol hdmi_avi_infoframe_init (err -2)
[  445.790435] drm: Unknown symbol of_get_videomode (err -2)
[  445.796552] drm: Unknown symbol dma_fence_signal_timestamp (err -2)
insmod: ERROR: could not insert module drm.ko: Unknown symbol in module
```

### 打印 B — KMS helper 缺 FB 绘图 / deferred-io

```text
[ 1507.693515] drm_kms_helper: Unknown symbol cfb_copyarea (err -2)
[ 1507.700342] drm_kms_helper: Unknown symbol fb_deferred_io_init (err -2)
[ 1507.707764] drm_kms_helper: Unknown symbol fb_deferred_io_cleanup (err -2)
[ 1507.715557] drm_kms_helper: Unknown symbol fb_sys_write (err -2)
...
insmod: ERROR: could not insert module drm_kms_helper.ko: Unknown symbol in module
```

### 打印 C — PHY 缺 mipi-dphy helper

```text
[  646.334256] phy_rockchip_inno_dsidphy: Unknown symbol phy_mipi_dphy_config_validate (err -2)
insmod: ERROR: could not insert module phy-rockchip-inno-dsidphy.ko: Unknown symbol in module
```

### 排查

1. `insmod` **失败但系统还活着** → 不是 panic，是链接依赖没满足。
2. `grep 符号 /proc/kallsyms`：没有 → 现核当初没编进该功能（现网 `# CONFIG_DRM is not set`，很多只在 =y 里的代码也不在）。
3. 用 `modprobe --show-depends` / 看 `.ko` 的 `modinfo -F depends` 理清顺序。

### 修复

| 缺的符号族 | 做法 |
|------------|------|
| `hdmi_*` / `of_get_videomode` / 部分 fence | 做 `drm_deps.ko` 把本该 =y 的代码 OOT 导出 |
| `cfb_*` / `sys_*` / `fb_sys_*` | 先装 `cfbfillrect` 等 FB 辅助 ko |
| `fb_deferred_io_*` | 做 `fb_defio` OOT（或关依赖该符号的路径） |
| `phy_mipi_dphy_*` | 先装 `phy_mipi_dphy.ko`，再装 `phy-rockchip-inno-dsidphy.ko` |

最终顺序固化在 `drm-kos-198/insmod_drm.sh`（或现 `insmod.sh`）。

---

## 案例 3：挂 `panel_simple` 立刻 Oops（OF 匹配踩野指针）

### 打印

```text
root@SMIOS:~/drm-kos-198# sh insmod_drm.sh
[ 3807.915133] Unable to handle kernel paging request at virtual address 00000000ffffffff
...
[ 3807.968919] Internal error: Oops: 96000004 [#1] PREEMPT SMP
[ 3807.975142] Modules linked in: panel_simple(O+) drm_kms_helper(O) drm(O) ...
[ 3808.040398] CPU: 2 PID: 3281 Comm: insmod Tainted: GF O 5.10.198 #236
[ 3808.063730] pc : __of_device_is_compatible+0xac/0x164
[ 3808.069372] lr : of_match_node+0x48/0xb0
```

### 排查

| 线索 | 含义 |
|------|------|
| `Unable to handle ... 00000000ffffffff` | 典型野指针/坏指针解引用 |
| `Internal error: Oops` | 这是 **Oops**（后面常升级成重启） |
| `panel_simple(O+)` + `Comm: insmod` | 正在装 panel 时炸 |
| `pc : __of_device_is_compatible` | 在 DT compatible 匹配里崩 → 传进 OF 的设备/节点指针不可信 |

根因方向：OOT 模块按 **226 头文件**理解的 `struct device` / `platform_device` / OF 相关布局，与现网 **198 #236** 不一致，probe 一开始就踩错偏移。

### 修复

- 用板上 config + **198 ABI 头**重编（`switch-kernel-abi.sh 198`，`abi-headers/`）。
- 不要只改 `SUBLEVEL` 字符串却继续用 226 结构体布局。

---

## 案例 4：挂 `rockchipdrm` — deferred probe 后 OF 再 Oops

### 打印

```text
[   48.473375] rockchip-vop2 ff400000.vop: deferred probe timeout, ignoring dependency
[   48.485502] Unable to handle kernel paging request at virtual address 0000000300000004
...
[   48.539300] Internal error: Oops: 96000004 [#1] PREEMPT SMP
[   48.545523] Modules linked in: rockchipdrm(O+) phy_rockchip_inno_dsidphy(O) ...
[   48.612830] CPU: 2 PID: 2935 Comm: insmod ...
[   48.636161] pc : __of_device_is_available.part.0+0x30/0xc4
[   48.642280] lr : of_device_is_available+0x38/0x6c
```

### 排查

1. 先有业务警告：`deferred probe timeout` → 依赖（时钟/电源/phy/iommu 等）没按时就绪，驱动仍往下走。
2. 紧接着在 `of_device_is_available` 崩 → 又是 **OF + 坏指针**，和案例 3 同族（ABI / 半初始化设备）。
3. `(O+)` 锁定罪魁是 `rockchipdrm`。

### 修复

- 继续对齐 198 ABI；保证 DT 里 VOP/LVDS/PHY/供电节点完整且 `status=okay`。
- 依赖 ko 顺序正确（含 `phy_mipi_dphy`、`inno-dsidphy`）。
- 必要时先修 deferred 依赖，再谈 bind。

---

## 案例 5：LVDS bind 失败 `-28`，错误路径 NULL 指针 Oops

### 打印

```text
[  539.722656] rockchip-vop2 ... vp0 assign plane mask: 0x0, primary plane phy id: -1
[  539.735396] rockchip-vop2 ... *ERROR* writeback connector init failed
[  539.746569] rockchip-drm display-subsystem: bound ff400000.vop (...)
[  539.756979] ... rockchip_lvds_bind ... *ERROR* failed to initialize connector: -28
[  539.769804] ... failed to bind ff030000.syscon:lvds (...): -28
[  539.783187] Unable to handle kernel NULL pointer dereference at virtual address 0000000000000000
...
[  539.838158] Internal error: Oops: 96000004 [#1] PREEMPT SMP
[  539.844374] Modules linked in: rockchipdrm(O+) ...
[  539.911681] Comm: insmod ...
[  539.935109] pc : drm_encoder_cleanup+0x28/0xc8 [drm]
[  539.940697] lr : vop2_unbind+0x150/0x178 [rockchipdrm]
```

### 排查（分层）

| 层 | 日志 | 含义 |
|----|------|------|
| 诱因 | `plane mask: 0x0` / writeback failed / connector `-28` | `-28` = `ENOSPC`，IDR/IDA 分配失败（当时还叠加 **226 `gfp.h`/XA 标志与 198 不一致**） |
| 二次伤害 | `NULL pointer` + `pc: drm_encoder_cleanup` + `lr: vop2_unbind` | bind 失败走 unbind，对**未初始化完的 encoder**做 cleanup → 空指针 |
| 类型 | `Oops` | 不是“温和返回错误”，错误路径写炸了 |

### 修复

1. **ABI**：`gfp.h` 等换回与 198 一致（`XA_FLAGS` / ida 行为），重编 `drm.ko` / `rockchipdrm.ko`。
2. **DT**：补齐 plane-mask 等，避免 `plane mask: 0x0`。
3. **代码加固**：`vop2_unbind` / writeback 失败路径不要对 NULL encoder 调 `drm_encoder_cleanup`。
4. 验证：能看到 `bound ... lvds` 且不再进这条 Oops。

---

## 案例 6：bind 已成功，却在 `fbcon_startup` 空指针（fbdev ABI）

### 打印

```text
[   59.812905] ... bind: crtcs=1 connectors=2 iommu=0
[   59.824945] rockchip-drm display-subsystem: failed to parse loader memory
[   59.836429] ... vop2_crtc_atomic_enable ... 1280x800 ... dclk: 56000000
...
[   59.852380] Unable to handle kernel NULL pointer dereference at virtual address 0000000000000000
[   59.852410] Internal error: Oops: 96000004 [#1] PREEMPT SMP
[   59.852488] Comm: insmod ...
[   59.852504] pc : fbcon_startup+0x64/0x2a4
[   59.852509] lr : do_take_over_console+0x80/0x200
```

### 排查

1. 前面已经 `bound vop` / `bound lvds`，说明 **KMS 主路径在往前走**。
2. 炸在 `fbcon_startup` → OOT DRM 去接现核 **fbcon/fbdev** 时 ABI/状态不对（现核本来就没按 DRM+fbcon 这套编）。
3. `iommu=0` 是另一问题（黑屏），但这次直接死因是 fbcon。

### 修复

- 在 `rockchip_drm_drv.c` **跳过 `rockchip_drm_fbdev_init`**（日志会出现 `skip rockchip_drm_fbdev_init (OOT/fbcon ABI)`）。
- Qt 走 `linuxfb + QT_QPA_FB_DRM=1`（DRM dumb），不依赖 `/dev/fb0` fbcon。

成功时类似：

```text
[  186.628605] ... skip rockchip_drm_fbdev_init (OOT/fbcon ABI)
[  186.642426] [drm] Initialized rockchip 3.0.0 20140818 for display-subsystem on minor 0
... /dev/dri/card0 ...
```

---

## 案例 7：`modetest` 关 fd 时 `put_pid` Oops（task_struct / drm_file ABI）

### 打印

```text
[  195.380720] CPU: 1 PID: 3027 Comm: modetest Tainted: G O 5.10.198 #236
[  195.422882] pc : put_pid.part.0+0x1c/0xb4
[  195.433568] lr : put_pid+0x14/0x2c
Call trace:
  put_pid.part.0+0x1c/0xb4
  put_pid+0x14/0x2c
  drm_release+0x6c/0x128 [drm]
```

（前序：`modetest -c` 已能列出 connector，退出关设备时崩。）

### 排查

| 线索 | 含义 |
|------|------|
| `Comm: modetest` | 不是 insmod，是用户态关 `/dev/dri/card0` |
| `drm_release` → `put_pid` | `drm_file` 释放时对 `file->pid` 做 `put_pid` |
| 野/坏 pid | OOT `drm.ko` 里取 `task_pid(current)` 时，对 **#236 `task_struct` 布局**理解不对 → 存进 `file->pid` 的是垃圾 |

### 修复

- `drivers/gpu/drm/drm_file.c`：`file->pid = NULL`（OOT 场景跳过 pid 跟踪），重编 `drm.ko`。
- 再跑 `modetest -c` / `-s` 应能正常退出。

---

# 音频（ALSA / RK809）案例

> 显示案例 1～7 的 vermagic / Unknown symbol / `put_pid` 方法论对音频同样适用。下面是 sound 外挂时踩过的点。

---

## 案例 8：现核无 ALSA — 无 `/proc/asound`（尚未 Oops）

### 打印 / 现象

```text
# zcat /proc/config.gz | grep CONFIG_SOUND
# CONFIG_SOUND is not set

# ls /proc/asound
ls: /proc/asound: No such file or directory
```

### 排查

| 线索 | 含义 |
|------|------|
| `# CONFIG_SOUND is not set` | Image 内无整棵 ALSA，不是“少装一个 codec ko” |
| 无 `/proc/asound` | soundcore 都未起来 |

与 DRM「无 `/dev/dri`」同类：**子系统被裁掉**，只改 DTS/`aplay` 不够。

### 修复

- `sound-fragment.config` 打开 `SOUND/SND/ASoC/RK817/SAI/...=m`，`build_snd_kos.sh` 产出 `snd-kos-198`。
- 换含 `sai` / `rk809-sound` / `pmic@20/codec` 的 DTB 后重启，再 `sh insmod.sh`。

---

## 案例 9：模块都装了仍无声卡 — MFD 无 `rk817-codec` 设备

### 打印 / 现象

```text
# cat /proc/asound/cards
--- no soundcards ---

# ls /sys/bus/platform/drivers/rk817-codec/
# （空，或 bind 不上）

# 有 DT 节点时：
# cat /proc/device-tree/i2c*/pmic@20/codec/compatible
rockchip,rk809-codec rockchip,rk817-codec
```

有时 dmesg 仅有 SAI/ASoC 片段，**没有** `rk817_probe: chip_name:...`。

### 排查

1. DT 有 `codec` 子节点 ≠ 内核里有 platform 设备。  
2. 现网 `rk808.ko` 在裁 SOUND 时往往 **去掉了 `rk817-codec` MFD cell** → probe 时不会按 OF 创建子设备。  
3. `snd-soc-rk817` 需要 parent=`RK809 I2C` 且 drvdata=`struct rk808`（regmap）。

### 修复

- `rk809_codec_bridge.ko`：在 `0-0020` 下按 OF 创建 `rk817-codec` platform 设备。  
- `insmod` 顺序：… → `rk809_codec_bridge` → `snd-soc-rk817` → `multicodecs`。  

成功时类似：

```text
rk809_codec_bridge: created rk817-codec parent=0-0020
rk817_probe: chip_name:0x80 ...
# cat /proc/asound/cards
 0 [rockchiprk809  ]: ...
```

---

## 案例 10：`amixer`/`aplay` 关 fd 时 `put_pid` Oops（与 DRM 案例 7 同源）

### 打印

```text
Unable to handle kernel paging request at virtual address ...
Internal error: Oops: ... PREEMPT SMP
CPU: ... Comm: amixer Tainted: G O 5.10.198 #236
pc : put_pid.part.0+0x...
lr : put_pid+0x...
Call trace:
  put_pid.part.0+...
  put_pid+...
  snd_ctl_release+...    # 或 pcm 路径上的 release/detach
```

（前序：`amixer`/`aplay` 可能已执行完一次操作，**退出关 `/dev/snd/*` 时崩**；严重时体感整机重启。）

### 排查

| 线索 | 含义 |
|------|------|
| `Comm: amixer` / `aplay` | 用户态关 control/pcm fd，不是 `insmod` 瞬间 |
| `snd_ctl_release` / pcm detach → `put_pid` | ALSA 对 `ctl->pid` / `substream->pid` 做 `put_pid` |
| 与 DRM `drm_release`→`put_pid` 同模式 | OOT 按 SDK 头取 `task_pid(current)`，与现网 **#236 `task_struct` 布局**不一致 → 存的是垃圾 pid |

### 修复

- `kernel-5.10/sound/core/control.c`：`ctl->pid = NULL`（open 时不跟踪）。  
- `kernel-5.10/sound/core/pcm.c`：attach 时 `substream->pid = NULL`。  
- 重编 `snd.ko` / `snd-pcm.ko`，装入 `snd-kos-198`。  

（思路对齐案例 7 的 `drm_file.c`。）

---

## 案例 11：`snd_card_new` 警告 `!module`（card->owner）

### 打印

```text
WARNING: CPU: ... at sound/core/init.c:... snd_card_new+0x...
...
WARN_ON(!module)
```

出现在 multicodecs 注册声卡时（具体 offset 随编译变化）。

### 排查

- Machine 驱动创建 `snd_card` 时 **`card->owner` 未设**，ALSA 核心 `WARN_ON(!module)`。  
- 不一定立刻 Oops，但说明 card 生命周期/引用不完整，应修。

### 修复

- `sound/soc/rockchip/rockchip_multicodecs.c`：`card->owner = THIS_MODULE;`  
- 重编 `snd-soc-rockchip-multicodecs.ko`。

---

## 案例 12：有声卡、`amixer` 仍报 `Invalid CTL hw:0`（用户态，非内核 Oops）

### 打印

```text
amixer: Mixer hw:0 load error: Invalid argument
# 或
ALSA lib ... Invalid CTL hw:0
```

此时 `/dev/snd/controlC0` 可能已存在，`cat /proc/asound/cards` 也能看到卡。

### 排查

1. 板 rootfs **没有**完整 `/usr/share/alsa/alsa.conf`，侧载 BR 的 `amixer` 找不到 `hw` 插件定义。  
2. 这不是内核 ABI，是 **用户态 alsa-lib 配置**。  
3. 若再 `export LD_LIBRARY_PATH=BR的libc`，还会污染系统命令（混 glibc）。

### 修复

- 使用包内 `tools/run.sh`：设 `ALSA_CONFIG_PATH=.../alsa.conf`，并用 `ld-linux-aarch64.so.1 --library-path ...` 启动 `amixer`/`aplay`/`mpg123`。  
- `play.sh` / `insmod.sh` 后设 `Playback Path` 走这套工具。

---

## 案例 13：PCM「播完」但喇叭无声 — `Playback Path`

### 现象

- `aplay`/`play.sh` 返回 0，耗时正常。  
- 喇叭无声；`Playback Path` 若为 `SPK`（本板外置功放）常无效。

### 排查

- 本板：RK809 → HPL/HPR + **GPIO3_C6 PA_CTL** 外置功放。  
- DT：`use-ext-amplifier` + `spk-ctl-gpios`。  
- mixer：需 **`Playback Path=SPK_HP`**（非单独 `SPK`）。

### 修复

```text
# 经 tools/run.sh
amixer -D hw:0 cset name='Playback Path' 'SPK_HP'
```

`insmod.sh` / `play.sh` 已内置该步骤。

---

## 案例 14：编 sound 时 modpost `exported twice`（宿主机，非板上 Oops）

### 打印

```text
FATAL: modpost: .../snd: 'xxx' exported twice. Previous export was in vmlinux
```

（与编 DRM 时 `drmm_mode_config_init exported twice` 同类。）

### 排查

- `Module.symvers.nodrm` 仍误带一批标成 `vmlinux` 的 ALSA 导出（历史 SOUND=y 残留）。  
- OOT 再 export 同名符号 → modpost FATAL。

### 修复

- `build_snd_kos.sh`：从 `Module.symvers.nodrm` **剥掉 sound 树 EXPORT 的符号** 再 `make M=sound`。  
- `KBUILD_MODPOST_WARN=1` 处理不完整 symvers 上的 undefined 警告。

---

## 案例 15：`stack_canary` 偏移错误 → 假 stack-protector（摄像头）

### 打印

```text
# 现象：v4l2_event_dequeue / DQEVENT 一睡眠就 panic（假栈破坏）
# 或 objdump 可见模块里大量：
ldr     xN, [xM, #1208]     # 错：本地 kernel tree 的 TSK_STACK_CANARY
# Image #236 实际是：
# task_struct.stack_canary @ 1192
```

### 排查

| 线索 | 含义 |
|------|------|
| 本地 `asm-offsets.h` 常为 `1208` | 用 SDK tree `.config` 算出的布局 |
| 现网 `#236` headers 为 `1192` | 与板子 `task_struct` 一致 |
| 错 offset 的模块一旦 `schedule()` | 读错 canary → 假 “stack corruption” |

### 修复

- `build_cam_kos.sh`：编前把 `include/generated/asm-offsets.h` 的 `TSK_STACK_CANARY` 打成 **1192**。  
- 校验：`videodev.ko` / `video_rkcif.ko` / `video_rkisp.ko` / phy 等 objdump 应为 `#1192`，不应再有 `#1208`。  
- 参考：`kx/smios-sdk-v2.0/usr/src/linux-headers-5.10.198/include/generated/asm-offsets.h`。

---

## 案例 16：media 缺导出 → `Unknown symbol cma_alloc` / `frame_vector_*` / `rockchip_iommu_*`

### 打印

```text
videobuf2_cma_sg: Unknown symbol cma_alloc (err -2)
videobuf2_cma_sg: Unknown symbol cma_release (err -2)
# 或
Unknown symbol get_vaddr_frames / frame_vector_create / rockchip_iommu_enable ...
```

### 排查

- 现核 `CONFIG_MEDIA_SUPPORT` 未开，整套 V4L2/vb2 都是 OOT。  
- Image 里 **有** DMA-CMA（约 32MB），但 **`cma_alloc` 未导出给模块**；vb2-cma-sg 却硬依赖它。  
- `frame_vector` / 部分 `rockchip_iommu_*` 同样可能未导出。

### 修复

- 先装 **`media_deps.ko`**（`cam/media-deps/`）：  
  - `cma_shim.c`：导出 `cma_alloc`/`cma_release`，内部走 **`dma_alloc_from_contiguous()`**（真 CMA）  
  - `frame_vector.c`：补 frame_vector API  
  - `iommu_shim.c` /（按需）`sysstatus_shim.c`：空实现导出  
- `insmod.sh`：**`media_deps` 必须最先**。

---

## 案例 17：假 CMA → 第二次 `grab_3a` `REQBUFS` 失败（无 NV12）

### 打印

```text
media_deps cma_alloc: fail count=1853 order=11
# 或早期假池：
media_deps cma pool: 0 x 16MiB + 0 x 8MiB
# 用户态：
VIDIOC_REQBUFS failed ...
# 第一次 grab 可能成功，第二次写不出文件
```

### 排查

| 线索 | 含义 |
|------|------|
| `count=1853` | 2592×1944 NV12 ≈ **8MiB**（1853 pages） |
| `#236` `MAX_ORDER=11` | buddy 单次最大 **4MiB**（order < 11） |
| 假实现：`alloc_pages`+`split_page` | 第一次抠走 8MiB 后，碎片化再也拼不回 |

### 修复

- **不要** 用 buddy 假扮 CMA。  
- `cma_shim`：优先 `dma_alloc_from_contiguous()`；不够再 `alloc_contig_range`。  
- `grab_3a.sh` 默认 **`-b 2`**（少占连续内存）。  
- 成功标志：`media_deps cma_alloc: dma-cma count=1853`。

---

## 案例 18：第二次抓拍 Oops（3A / params STREAMOFF 顺序与 IQ CMA）

### 打印

```text
# 典型：第二次 grab_3a 或 STREAMOFF 路径
Unable to handle kernel paging request at virtual address ...
# Comm: v4l2_grab / rkaiq_3A_server
# 或 params/stats 缓冲已释放后 3A 仍持有 dma_fd mmap → UAF
```

### 排查

- OOT vb2/CMA **不会像内置 media 那样长期钉住** IQ 页。  
- SDK 默认 STREAMOFF 顺序若先 `params_stream_stop` 再通知 3A，OOT 上易在 3A 仍映射时拆缓冲。  
- 无 IOMMU 时若仍写 BAY3D/LDCH 等 DDR 模块，还会砸内存。

### 修复（`kx/移植/cam/video_rkisp/` 副本）

1. `hw.c`：无 IOMMU domain → `is_mmu=0`。  
2. `isp_params_v32.c`：`rkisp_oot_skip_iq_ddr()` 跳过 BAY3D/LDCH/CAC/3DLUT；params 仍 open 时 **keep IQ CMA**。  
3. `isp_stats_v32.c`：无 MMU 不使能 `3A_DDR_WRITE`。  
4. `rkisp.c`：STREAMOFF **先 `rkisp_stop_3a_run`，再 `rkisp_params_stream_stop`**。  
5. `capture_v32.c` / `common.h`：`rkisp_v32_stream_irq_ok` / `rkisp_vb2_buf_ok` 防 poison ISR。

成功 dmesg 片段：

```text
OOT isp is_mmu=0 contig=1 reserved=0
OOT no IOMMU: skip BAY3D/LDCH/CAC/3A-DDR, 3A AE/AWB on
Waiting for 3A on/off … ms
keep IQ/3DNR/stats CMA (params still open)
```

---

## 案例 19：sensor 电 / `cam_mclk` 与自管电冲突

### 打印

```text
# probe 失败、无 0x37 UU、无图
Failed to get xvclk ...
# 或同时启用 cam_mclk + sensor clocks 时行为不定
```

### 排查

- 方案冲突：要么 **cam_mclk.ko** 管 MCLK/reset/pwdn，要么 **gc05a2 自管**（DT 写 `clocks=xvclk` + gpio）。  
- 本仓可用路径：**sensor 自管**；`insmod.sh` 会尝试 `rmmod cam_mclk`。

### 修复

- DTB 用方案1（sensor 拥有 xvclk/reset/pwdn）；不要与 `cam_mclk0` 同时 okay。  
- OOT `gc05a2.c`：去掉未导出的 `mutex_destroy` 等。  
- `i2cdetect -y 5` 见 `0x37` 为 `UU`。

---

## 案例 20：用户态 / 流程误用（非内核 Oops，但「不出彩图」）

### 打印

```text
# rkaiq 刷屏（可忽略）
Bad media topology media2
Bad media topology media3
...
# CIF 侧偶发
Failed to open /dev/video1
# 若误跑旧 grab.sh：会杀 3A / 重搭 link → 灰图或断流
```

### 排查

| 现象 | 含义 |
|------|------|
| `Bad media topology media2–15` | rkaiq 扫多余 media 节点，**可忽略** |
| CIF `video1` open fail | 非主路径，**可忽略** |
| 有 NV12 但 uv 全 0 / 灰图 | 3A 未跑或被杀掉；IQ 路径不对 |
| glibc 版本不够 | SMIOS 系统 linker 跑不动 Buildroot 编的 rkaiq |

### 修复

- 只用：`insmod.sh` → `start_3a.sh` → `grab_3a.sh`（**不要** 3A 起来后再跑旧 `grab.sh`）。  
- IQ：`/tmp/iqfiles/`；`rkaiq_3A_server` 默认 2592×1944。  
- `start_3a.sh`：私有 `ld-linux` + RT 配额。  
- 校验彩图：`Ymean`/`UVmean` 正常，`uv0%` 不要接近 100。

---

## 案例对照速查

| 现象 | 类型 | 优先动作 |
|------|------|----------|
| `bad vermagic` / `Invalid module format` | 版本字符串 | `-f` 或 vermagic=198 |
| `Unknown symbol xxx` | 缺依赖 / 现核无导出 | 补 ko 或 deps shim，调顺序 |
| `pc` 在 `of_*`，`(O+)` 某驱动 | Oops，OF/设备 ABI | 切 198 ABI 头重编 |
| bind `-28` + `drm_encoder_cleanup` | Oops，失败路径 + IDA/ABI | 换 `gfp.h`、加固 unbind、修 DT plane-mask |
| `pc: fbcon_startup` | Oops，fbdev 耦合 | skip `fbdev_init` |
| `pc: put_pid`，`Comm: modetest` | Oops，`drm_file`/task ABI | `file->pid = NULL` |
| 无 `/proc/asound` | SOUND 整栈未开 | OOT `snd-kos-198` + 音频 DTB |
| 有 DT codec、无声卡 | MFD 无 cell | `rk809_codec_bridge` |
| `pc: put_pid`，`Comm: amixer`/`aplay` | Oops，ALSA ctl/pcm pid ABI | control.c / pcm.c 置 pid NULL |
| `WARN_ON(!module)` @ `snd_card_new` | card->owner | multicodecs `THIS_MODULE` |
| `Invalid CTL hw:0` | 用户态缺 alsa.conf | `tools/run.sh` + `ALSA_CONFIG_PATH` |
| 播完无声 | 通路/功放 | `Playback Path=SPK_HP` |
| modpost `exported twice` | 假 vmlinux 符号 | 剥 `Module.symvers` |
| DQEVENT / 假 stack-protector | canary 1208 vs 1192 | `build_cam_kos.sh` 强制 1192 |
| `Unknown symbol cma_alloc` 等 | media 未导出 | 先装 `media_deps.ko` |
| `cma_alloc: fail … order=11` / 二次 grab 无文件 | 假 CMA / 碎片 | 真 DMA-CMA + `-b 2` |
| 二次 grab Oops / params UAF | 3A 与 STREAMOFF | 先 stop_3a；keep IQ；skip IQ DDR |
| sensor 无 UU / 无 xvclk | 电与时钟方案冲突 | sensor 自管；卸 `cam_mclk` |
| `Bad media topology` / CIF video1 | 噪声日志 | 可忽略 |
| NV12 灰图 / uv0%≈100 | 3A/IQ/流程 | `start_3a`+`grab_3a`；查 IQ |
| 只有半截日志就重启 | 可能 panic/看门狗 | 仍按最后一个 `(O+)` / `Comm:` 缩小范围 |

---

## 和「一挂就重启」的关系

上面带 `Internal error: Oops` 的案例，在现网常会 **接着 panic 或复位**，体感就是「挂完重启」。  
串口里先出现的是 **Oops**；是否写成 `Kernel panic - not syncing` 取决于当时能否刷完、以及 `panic_on_oops` / 看门狗。

排查时：**先当 Oops 读 `pc`/`lr`/`(O+)`/`Comm:`，再谈修 ABI 或错误路径**——这就是当时你贴日志、我们改 ABI/补丁的真实过程。

音频侧补充：很多「没声」**不是 Oops**（案例 8/9/12/13）；先确认有没有 card、有没有 bridge、Path 对不对，再和 `put_pid` 类崩溃分开处理。

摄像头侧补充：很多「第二次没图」先是 **CMA/`REQBUFS`**（案例 17），不一定是 Oops；二次才崩再按案例 18 查 3A/STREAMOFF。完整流程见 [`cam/README.md`](./cam/README.md)、[`OOT-PORTING-GUIDE.md`](./OOT-PORTING-GUIDE.md)。
