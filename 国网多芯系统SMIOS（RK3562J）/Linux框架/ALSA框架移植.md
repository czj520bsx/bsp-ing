# SMIOS 多核板 ALSA/RK809 外挂移植记录

> 目标：在**不能刷验签 Kernel/U-Boot** 的 RK3562 多核板（SMIOS）上，用可换 DTB + OOT `.ko` 打通 RK809 喇叭，并跑通 WAV/MP3 播放。  
> 板上内核：`5.10.198 #236`，`# CONFIG_SOUND is not set`。  
> SDK 驱动源码树：`kernel-5.10`（Rockchip **5.10.226** 音频）+ 198 ABI 对齐（复用 DRM 的 `switch-kernel-abi.sh`）。  
> 板型：Rockchip RK3562J CORE DDR4 V10（KX；外置功放 HPL/HPR + `GPIO3_C6` PA_CTL）。
### 双链路切换（226 SDK 整编 ↔ 198 OOT）

与 DRM **同一套** ABI 切换脚本：

```bash
./czj/drm/switch-kernel-abi.sh status
./czj/drm/switch-kernel-abi.sh 198 --with-config   # 编 snd-kos-198
./czj/drm/switch-kernel-abi.sh 226                 # SDK 整编 / stamp_kx
```

| 模式 | 头文件库 | `SUBLEVEL` | 用途 |
|------|----------|------------|------|
| **226** | `czj/drm/abi-headers/226/` | 226 | SDK 整编 Image / modules / DTB |
| **198** | `czj/drm/abi-headers/198/` | 198 | OOT 音频，vermagic 对齐现网 `5.10.198` |

- 已编好的 `czj/snd/snd-kos-198/*.ko` **不受**切换影响。  
- 现网验签 Image 仍是 198；**不要**把 226 Image 刷进验签分区；音频拓扑优先只换 DTB。

---

## 一、做了什么（时间线 / 工作内容）

### 1.1 从板子诊断（起点）

在板上：

```bash
uname -a
# Linux SMIOS 5.10.198 #236 ...

zcat /proc/config.gz | grep -E 'CONFIG_SOUND|CONFIG_SND[=_]|CONFIG_MODULES'
# 关键结论：# CONFIG_SOUND is not set

ls /proc/asound                    # No such file or directory
find /lib/modules/$(uname -r) -name 'snd*.ko*'   # 空
cat /proc/device-tree/compatible
# rockchip,rk3562j-core-ddr4-v10 ...
ls /proc/device-tree/rk809-sound   # 当时无（旧 DTB）
```

与 DRM 对照：

| 项 | DRM 当时 | 音频当时 | 含义 |
|----|----------|----------|------|
| 框架 Kconfig | `# CONFIG_DRM is not set` | `# CONFIG_SOUND is not set` | Image 内无整棵栈 |
| `CONFIG_MODULES` | y | y | 可外挂 `.ko` |
| 运行中 DTB | 曾无 vop/lvds | 曾无 sai/rk809-sound/codec | 必须换/补 DTB |
| Image/U-Boot | 验签不能刷 | 同 | 只换 DTB + ko |

### 1.2 方案演进（认识过程）

1. 最初：以为只改 DTS/`aplay` 即可 → 现核无 ALSA，`/proc/asound` 都不存在。  
2. 对照 DRM：把 **整棵 SOUND/ASoC** 做成 OOT `.ko`（`CONFIG_SOUND=m` 等），再 `insmod`。  
3. 实际落地：**可换音频 DTB + `snd-kos-198` + `rk809_codec_bridge` + 侧载 alsa-utils/mpg123**。

公版对照：`CONFIG_SOUND=y` 编进 Image + `rk809_sound` + Buildroot `amixer`/`aplay`。  
多核板不能仿公版刷核，只能外挂。

### 1.3 编译环境对齐 5.10.198

在 `kernel-5.10` 上（与 DRM 相同底座）：

1. `board-5.10.198.config` 做底 → merge `czj/snd/sound-fragment.config` → `olddefconfig`。  
2. 打开音频相关为 **模块**：`SOUND`/`SND`/`SND_SOC`/`ROCKCHIP_SAI`/`MULTICODECS`/`RK817`、`ROCKCHIP_CLK_OUT` 等 `=m`。  
3. `SUBLEVEL = 198` + `abi-headers/198`（`switch-kernel-abi.sh 198`）。  
4. `Module.symvers`：现网基准用 `Module.symvers.nodrm`，但其中误含 SOUND=y 时留下的 ALSA 符号 → 编 sound 前 **剥掉** 与 `sound/` 导出冲突的 vmlinux 项；`KBUILD_MODPOST_WARN=1`（nodrm 本就不完整，同 DRM 类做法）。

**注意：** 驱动 `.c` 主体仍是 **226 Rockchip 音频树**；198 贡献 vermagic + ABI 头。

### 1.4 设备树：恢复音频拓扑

多核 `rk3562.dtsi` 裁掉了 sai/mclkout。新增/维护：

- `kx/rockchip/kx_rk3562-audio.dtsi`：`mclkout_sai0`、`sai0`、`rk809_sound`（默认 disabled）  
- `kx/rockchip/rk3562-rk809.dtsi`：补 `pmic@20/codec`（`rockchip,rk817-codec`）+ `use-ext-amplifier` + `spk-ctl-gpios`  
- `kx/rockchip/kx_rk3562-core-ddr4-v10.dts`：打开 `&sai0` / `&rk809_sound`，PA_CTL 脚 `GPIO3_C6` pinctrl  

编译：`kx/rockchip/build_kx_kernel_dtb.sh` → `kx_kernel-dtb.img`。

重启后应看到：

```bash
ls /proc/device-tree/rk809-sound /proc/device-tree/sai@ff800000
cat /proc/device-tree/i2c*/pmic@20/codec/compatible   # 含 rk817-codec
```

过程中踩过：

- 只刷了部分音频节点、**没有 codec 子节点** → sai 能 bind，但无 `rk817-codec` 设备。  
- 有 codec OF 节点，现网 **`rk808.ko` 无 `rk817-codec` MFD cell**（SOUND 裁掉时一并去掉）→ 仍无 platform 设备 → 写了 `rk809_codec_bridge.ko`。

### 1.5 OOT 模块包与加载

```text
czj/snd/snd-kos-198/
  *.ko + insmod_snd.sh + tools/ + setup_path.sh / play_*.sh
```

加载顺序（见 `insmod_snd.sh`）：

```text
clk-out → soundcore → snd → timer/hrtimer → pcm → pcm-dmaengine
→ snd-soc-core → snd-soc-rockchip → sai
→ rk809_codec_bridge → snd-soc-rk817 → multicodecs
```

成功标志：

```text
/proc/asound/cards  →  rockchiprk809
/dev/snd/controlC0  pcmC0D0p
dmesg: rk809_codec_bridge: created rk817-codec parent=0-0020
dmesg: rk817_probe: chip_name:0x80 ...
```

### 1.6 驱动侧关键补丁（OOT 稳定跑）

| 问题 | 处理 |
|------|------|
| `amixer`/`aplay` 关 fd 时 `put_pid` oops | `sound/core/control.c`、`pcm.c`：`ctl->pid` / `substream->pid = NULL`（同 DRM `drm_file.c`：`task_pid` ABI） |
| `snd_card_new` `WARN_ON(!module)` | `rockchip_multicodecs.c`：`card->owner = THIS_MODULE` |
| 现网 MFD 无 codec cell | `czj/snd/rk809-codec-bridge/`：按 OF 创建 `rk817-codec` platform 设备，parent=`0-0020` |
| 板 rootfs 无 alsa-utils / 无 `alsa.conf` | `snd-kos-198/tools/`：侧载 amixer/aplay/mpg123 + BR libc + **`usr/share/alsa/alsa.conf`**；`run.sh` 设 `ALSA_CONFIG_PATH` |

### 1.7 用户态验证：通路 → WAV → MP3

1. **通路**  
   - `Playback Path = SPK_HP`（外置功放；`SPK`  alone 无声已在 stamp_kx 验证）  
   - 必须用打包的 `tools/run.sh`（否则缺 `alsa.conf` → `hw` 插件不存在）

2. **WAV**  
   - `sh play_wav.sh ./test_440.wav` → 约 2s、440Hz 蜂鸣  

3. **MP3**  
   - `sh play_mp3.sh /root/xxx.mp3`（侧载 `mpg123`）

**禁止**对 BR 库 `export LD_LIBRARY_PATH=...` 后跑系统命令（会混 glibc）。  
必须：`tools/lib/ld-linux-aarch64.so.1 --library-path ... tools/bin/xxx`（已封在 `tools/run.sh`）。

### 1.8 当前状态摘要

| 项 | 状态 |
|----|------|
| OOT ALSA + `/proc/asound` | 已通 |
| `rockchiprk809` 声卡 | 已通 |
| `rk809_codec_bridge` | 已通 |
| 喇叭出声（WAV / 已验证） | 已通 |
| MP3（mpg123 侧载） | 工具已备，按文件验证即可 |
| 验签 Image | 未改 |
| 开机自动 insmod | 产品化待做 |

---

## 二、改了哪些文件的代码（明确清单）

### 2.1 内核树 `kernel-5.10`（只用于编 OOT，不刷进验签 Image）

#### 版本与配置

| 文件 | 改动 |
|------|------|
| `kernel-5.10/Makefile` | `SUBLEVEL = 198`（与 DRM OOT 共用切换） |
| `czj/snd/sound-fragment.config` | 在 board config 上打开 SOUND/SND/ASoC/RK817/CLK_OUT=`m` |
| `kernel-5.10/.config` | board-5.10.198 + fragment + olddefconfig |

#### 头文件 ABI

与 DRM 相同：`switch-kernel-abi.sh 198` 安装 `czj/drm/abi-headers/198/`。

#### ALSA / Rockchip 驱动补丁

| 文件 | 改动要点 |
|------|----------|
| `kernel-5.10/sound/core/control.c` | `snd_ctl_open`：`ctl->pid = NULL`；release 时条件 `put_pid` |
| `kernel-5.10/sound/core/pcm.c` | attach：`substream->pid = NULL`；detach 条件 `put_pid` |
| `kernel-5.10/sound/soc/rockchip/rockchip_multicodecs.c` | `card->owner = THIS_MODULE` |

#### 设备树（刷进/替换启动 DTB 的源）

| 文件 | 改动要点 |
|------|----------|
| `kx/rockchip/kx_rk3562-audio.dtsi` | sai0、mclkout_sai0、rk809_sound 骨架 |
| `kx/rockchip/rk3562-rk809.dtsi` | `codec` 子节点 + `use-ext-amplifier` + `spk-ctl-gpios` |
| `kx/rockchip/kx_rk3562-core-ddr4-v10.dts` | include audio dtsi；`&sai0`/`&rk809_sound` okay；PA_CTL pinctrl |
| `kx/rockchip/build_kx_kernel_dtb.sh` | 校验 audio dtsi 存在并打包 FIT |
| 产物 | `kx_rk3562-core-ddr4-v10.dtb` / `kx_kernel-dtb.img` |

### 2.2 OOT 交付与辅助工程（`czj/snd/`）

| 路径 | 说明 |
|------|------|
| `snd-kos-198/` | 上板 ko + `insmod_snd.sh` + `tools/` + 播放脚本 |
| `rk809-codec-bridge/` | `rk809_codec_bridge.ko` 源码 |
| `sound-fragment.config` | Kconfig fragment |
| `build_snd_kos.sh` | 一键：merge config、剥 symvers、编 sound/clk-out、打包 |
| `音频外挂移植记录.md` | 本文 |
| `音频框架通关指南.md` | ALSA/ASoC 框架 + 本板三层 |

### 2.3 明确未改 / 不能改

- 板上**验签** Kernel Image、U-Boot  
- 多核 rootfs 系统 glibc（不替换为 BR 2.38；播放器用侧载）  
- 不替换现网 `rk808.ko`（只补 bridge 创建设备）

### 2.4 音频与 PMIC 的关系（避免误解）

- RK809 **电源/regulator** 仍由现网 `rk808` MFD 管。  
- **Codec** 在公版是 MFD cell；现网裁 SOUND 后 cell 没了 → bridge 补 platform 设备。  
- Codec 驱动需要 **parent drvdata = `struct rk808`**（regmap），故 bridge 必须挂到 `0-0020`。

---

## 三、怎么编译 & 怎么移植上板

### 3.1 环境假设

```text
SDK 根目录：
/home/ChenZhangjin/HZ-RK3562-LINUX_5.10_SDK_20250918_V1.2

交叉编译：
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-

用户态工具来自：
buildroot/output/rockchip_kx_rk3562_sp/target/{usr/bin,usr/lib,usr/share/alsa,lib}
```

### 3.2 编译内核模块（推荐脚本）

```bash
cd /home/ChenZhangjin/HZ-RK3562-LINUX_5.10_SDK_20250918_V1.2
./czj/drm/switch-kernel-abi.sh 198 --with-config
bash ./czj/snd/build_snd_kos.sh
# 单独编 bridge：
cd czj/snd/rk809-codec-bridge && make
cp rk809_codec_bridge.ko ../snd-kos-198/
```

检查 vermagic：

```bash
modinfo -F vermagic czj/snd/snd-kos-198/snd.ko
# 5.10.198 SMP preempt mod_unload aarch64
```

### 3.3 编译设备树

```bash
cd kx/rockchip
bash ./build_kx_kernel_dtb.sh
# 产物：kx_kernel-dtb.img（只刷 kernel-dtb 分区）
```

### 3.4 用户态工具包（`tools/`）

内含：`amixer`/`aplay`/`mpg123`、BR `libc`/`libasound`、`usr/share/alsa/alsa.conf`、`run.sh`、`test_440.wav`。

缺 `alsa.conf` 时即使有 `/dev/snd/controlC0`，也会报 `Invalid CTL hw:0`（插件未定义）。

### 3.5 移植上板流程（复现清单）

#### 步骤 A：换音频 DTB 并重启

1. 刷含 codec/sai/rk809-sound 的 `kx_kernel-dtb.img`。  
2. 重启。  
3. 确认：

```bash
cat /proc/device-tree/rk809-sound/status          # okay
cat /proc/device-tree/i2c*/pmic@20/codec/compatible | xxd   # rk817-codec
```

#### 步骤 B：加载 OOT 音频

```bash
scp -r czj/snd/snd-kos-198 root@板子:/root/
cd /root/snd-kos-198
# 若 tools 单独更新：
# tar xzf tools.tgz
sh insmod_snd.sh
cat /proc/asound/cards
```

#### 步骤 C：通路 + 出声

```bash
sh setup_path.sh                 # Playback Path=SPK_HP
time sh play_wav.sh ./test_440.wav   # ~2s，有蜂鸣
sh play_mp3.sh /root/xxx.mp3
```

### 3.7 换板 / 换核注意

- **同 5.10.198 + 同系列 ABI**：可复用 `snd-kos-198`。  
- **内核版本/# 变了**：重对齐 vermagic/头文件并重编 ko。  
- **功放拓扑不同**：改 `Playback Path` / `spk-ctl-gpios` / 是否 `use-ext-amplifier`。  
- **不要**把未开 SOUND 的旧 DTB 与新 ko 混用，或漏刷 codec 节点。

---
