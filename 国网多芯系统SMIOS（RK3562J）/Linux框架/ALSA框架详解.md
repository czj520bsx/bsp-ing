# ALSA / ASoC 框架通关指南（通用核心 + Rockchip 三层源码）

> 目标：先建立 **Linux ALSA / ASoC 通用模型**，再对照本仓库 **Rockchip OOT 音频源码** 吃透 SAI + RK809 + multicodecs 挂载与出声。  
> 代码根：`kernel-5.10/sound/`，板级 DT：`kx/rockchip/`。

---

# 第一部分：ALSA / ASoC 通用核心

## 1. ALSA 解决什么问题

音频子系统要回答：

1. 用户态写的 PCM 样本往哪送？  
2. 硬件按什么时钟/格式（率、位宽、通道）搬数据？  
3. 模拟通路（喇叭/耳机/麦）怎么在 mixer 里切？

老接口是 **OSS**；现代用户态走 **ALSA**：

- **ALSA Core**：`/dev/snd/*`、声卡、PCM、control、定时器  
- **ASoC（ALSA System on Chip）**：把 SoC DAI、Codec、板级 Machine 拆开再组装

用户态（`aplay`/`amixer`/`mpg123`、Pulse/PipeWire、GStreamer）最终都通过 `/dev/snd/pcmC*D*p` 等与内核对话。

> 本板外挂栈：**ASoC machine = rockchip-multicodecs**，CPU DAI = SAI0（**数字音频接口**），Codec = RK817/RK809（**编解码芯片**）。

---

## 2. 核心对象模型

把播放想成一条管道：

```text
  用户态 PCM write
        │
        ▼
  ┌──────────┐     ┌────────────┐     ┌────────────┐
  │ snd_pcm  │────►│   DMA/CPU  │────►│  Codec DAC │──► 模拟/功放 ──► 喇叭
  │ (ALSA)   │     │    DAI     │     │            │
  └──────────┘     └────────────┘     └────────────┘
        ▲                 ▲                  ▲
        │                 │                  │
   snd_card          CPU DAI driver     codec driver
        │
        └── Machine driver（绑定 DAI link + DAPM 路由）
```

| 对象          | 含义                                    | 用户态大致对应                |
| ----------- | ------------------------------------- | ---------------------- |
| **Card**    | 一张声卡（`/proc/asound/cards`）            | `hw:0` / `card0`       |
| **PCM**     | 播放/录音流（设备号、子流）                        | `aplay -D hw:0,0`      |
| **Control** | mixer 枚举/音量（kcontrol）                 | `amixer -c 0 cset ...` |
| **DAI**     | Digital Audio Interface（I2S/SAI/PDM…） | 通常对用户透明                |
| **Codec**   | DAC/ADC + 模拟通路 + DAPM widgets         | Playback Path 等        |
| **Machine** | 板级：把 CPU DAI + Codec 链成 card          | DT `compatible`        |

ASoC 口诀：

```text
Machine 绑 DAI link → CPU DAI 出数字流 → Codec 转模拟 → DAPM 选喇叭/耳机
```

一次完整“出声”至少要：card 注册成功、PCM 设备存在、时钟/格式协商通、mixer 通路打开、DMA 在跑。

---

## 3. 设备节点与会话：`snd_card` / `snd_pcm` / `snd_ctl`

### 3.1 `snd_card`

Machine/`snd_soc_register_card` 成功后，内核出现类似：

```text
/proc/asound/cards          # 0 [rockchiprk809]: ...
/dev/snd/controlC0          # mixer
/dev/snd/pcmC0D0p           # playback
/dev/snd/timer              # （可选）
```

本板 card 名来自 multicodecs + DT `model`：

```text
kernel-5.10/sound/soc/rockchip/rockchip_multicodecs.c
  compatible: "rockchip,multicodecs-card"
DT: model = "rockchip,rk809-codec";
```

### 3.2 打开与 `pid`（本仓库 OOT 补丁）

用户态 `open("/dev/snd/controlC0")` → `snd_ctl_open`；PCM 同理。

现网 `task_struct` / `task_pid` ABI 与 SDK 头不一致时，`put_pid` 会 oops（与 DRM `drm_file.c` 同源）。本仓库：

```text
sound/core/control.c   ctl->pid = NULL
sound/core/pcm.c       substream->pid = NULL
```

### 3.3 ioctl / 插件（用户态入口）

| 能力 | 典型路径 |
|------|----------|
| 查卡/PCM | `/proc/asound`、`pcmC*D*` |
| mixer | `SNDRV_CTL_*`（amixer） |
| 播 PCM | `SNDRV_PCM_*`（aplay） |
| 设备名解析 | **alsa-lib 插件**：`hw`/`plughw`/`sysdefault` 定义在 `alsa.conf` |

**重要：** 仅有内核节点不够。侧载 `amixer`/`aplay` 若找不到 `alsa.conf`，会报 `Invalid CTL hw:0`（插件未加载）。本包用：

```text
ALSA_CONFIG_PATH=.../usr/share/alsa/alsa.conf
tools/run.sh → ld-linux + --library-path（勿 export LD_LIBRARY_PATH 污染系统）
```

---

## 4. ASoC 三层

```text
┌─────────────────────────────────────────────────────┐
│  Machine（板级 card）                                 │
│  rockchip_multicodecs / simple-audio-card …         │
│  - dai-link：cpu ↔ codec                             │
│  - mclk / format / routing                           │
└───────────────┬─────────────────────┬───────────────┘
                │                     │
       ┌────────▼────────┐   ┌────────▼────────┐
       │  CPU DAI        │   │  Codec          │
       │  SAI / I2S …    │   │  rk817-codec    │
       │  DMA + 时钟     │   │  DAC + DAPM     │
       └─────────────────┘   └─────────────────┘
```

| 层 | 本板驱动 | 典型 DT |
|----|----------|---------|
| Machine | `snd-soc-rockchip-multicodecs.ko` | `rk809_sound` / `compatible = "rockchip,multicodecs-card"` |
| CPU DAI | `snd-soc-rockchip-sai.ko` | `sai@ff800000` |
| Codec | `snd-soc-rk817.ko` | `pmic@20/codec` → platform `rk817-codec` |

注册顺序依赖：先有 `snd-soc-core`，再有 CPU/Codec 组件，最后 Machine probe 才能匹配 DAI。

---

## 5. DAPM 与 Playback Path（为什么要设 mixer）

**DAPM（Dynamic Audio Power Management）** 用 widget/route 描述模拟通路；kcontrol（如 `Playback Path`）改变路径。

本板外置功放走 **HP 模拟输出 + GPIO PA**：

```text
amixer cset name='Playback Path' 'SPK_HP'
```

| 取值 | 含义（本板） |
|------|----------------|
| `SPK` | 内置喇叭路径语义；外置功放板上常**无声** |
| `SPK_HP` | 走 HP 输出 + 外置功放（**已验证有声**） |
| `HP` | 耳机路径 |

DT 侧：

```text
use-ext-amplifier;          # codec 知道外置功放
spk-ctl-gpios = <&gpio3 RK_PC6 ...>;  # PA_CTL
```

没有正确的 Path / PA GPIO，PCM 可能“跑完”但喇叭无声。

---

## 6. 时钟与主从（简图）

本板常见：

```text
mclkout_sai0 ──MCLK──► RK809 codec
sai0 BCLK/LRCK/DATA  ◄─► codec I2S
```

DT 里 `mclk-fs`、`clocks = <&mclkout_sai0>`、`assigned-clocks` 决定主时钟。  
`clk-out.ko`（`ROCKCHIP_CLK_OUT`）提供 `mclkout_sai0`，须在 SAI/codec 前加载。

---

# 第二部分：本仓库 Rockchip 三层对照

## 1. Codec：`snd-soc-rk817` + bridge

### 1.1 公版 vs 多核板

| | 公版（SOUND=y） | 多核现网（SOUND 裁掉） |
|--|----------------|------------------------|
| MFD | `rk808` 注册 `rk817-codec` cell | **无** codec cell |
| DT | `pmic@20/codec` | 补回节点仍不够 |
| 设备 | probe 自动有 platform 设备 | 需 **`rk809_codec_bridge.ko`** 创建 |

Bridge 要点（`czj/snd/rk809-codec-bridge/`）：

- 找 I2C 上 `rk809`/`0-0020`  
- 在其下按 OF 创建 `rk817-codec` platform 设备  
- **parent 必须是 PMIC**，以便 `snd-soc-rk817` 取 `struct rk808` / regmap  

```text
dmesg: rk809_codec_bridge: created rk817-codec parent=0-0020
dmesg: rk817_probe: chip_name:0x80 ...
```

### 1.2 源码位置

```text
kernel-5.10/sound/soc/codecs/rk817_codec.c
compatible: "rockchip,rk817-codec"
```

注意日志里可能有 `Has no ADC channel`（仅播放场景可忽略）。

---

## 2. CPU DAI：Rockchip SAI

```text
kernel-5.10/sound/soc/rockchip/rockchip_sai.c
DT: sai@ff800000  (kx_rk3562-audio.dtsi)
模块: snd-soc-rockchip-sai.ko
依赖: snd-soc-rockchip.ko、DMA、clk-out
```

多核 `rk3562.dtsi` 曾裁 sai → 由 `kx_rk3562-audio.dtsi` 恢复。

---

## 3. Machine：`rockchip_multicodecs`

```text
kernel-5.10/sound/soc/rockchip/rockchip_multicodecs.c
compatible: "rockchip,multicodecs-card"
模块: snd-soc-rockchip-multicodecs.ko
```

本仓库补丁：

```text
card->owner = THIS_MODULE;   // 避免 snd_card_new WARN_ON(!module)
```

DT 片段概念：

```text
rk809_sound {
  compatible = "rockchip,multicodecs-card";
  rockchip,codec = <&rk809_codec>;
  rockchip,cpu = <&sai0>;
  ...
};
```

缺少 `rockchip,audio-routing` 时可能有 warn，**仅播放**通常仍可工作。

---

## 4. ALSA Core 模块链

`insmod_snd.sh` 顺序（概念）：

```text
clk-out.ko
soundcore.ko → snd.ko → snd-timer / snd-hrtimer
→ snd-pcm → snd-pcm-dmaengine
→ snd-soc-core
→ snd-soc-rockchip → snd-soc-rockchip-sai
→ rk809_codec_bridge → snd-soc-rk817
→ snd-soc-rockchip-multicodecs
```

任一环缺失：可能只有部分设备、无 card、或 codec unbound。

---

## 5. 用户态播放路径

```text
MP3 文件
  → mpg123（侧载）解码成 PCM
  → alsa-lib（hw:0,0）
  → /dev/snd/pcmC0D0p
  → SAI DMA → RK809 DAC → PA_CTL → 喇叭
```

WAV 则直接 `aplay`。  
`setup_path.sh` 在播前固定 `Playback Path=SPK_HP`。

---

# 第三部分：排障速查

| 现象 | 优先查 |
|------|--------|
| 无 `/proc/asound` | ko 未加载 / SOUND 栈未起 |
| 有 asound 无 card | Machine/codec 未 bind；DT status；bridge 是否创建设备 |
| `rk817-codec` 无设备 | DTB 无 codec 子节点；或无 bridge；parent 错 |
| `Invalid CTL hw:0` | 缺 `ALSA_CONFIG_PATH` / `alsa.conf` |
| aplay 秒退但无声 | Path 不是 `SPK_HP`；PA GPIO；音量 mute |
| `put_pid` oops | 是否用了未打 pid 补丁的 `snd.ko`/`snd-pcm.ko` |
| vermagic 拒绝 | 未 `switch-kernel-abi.sh 198` 或混用 226 ko |

板上快速确认：

```bash
cat /proc/asound/cards
ls /dev/snd/
ls /sys/bus/platform/drivers/rk817-codec/
dmesg | grep -E 'rk809_codec_bridge|rk817_probe|multicodecs|sai'
```

---



