# picasso（SM7250/lito）主线音频 bring-up 记录

状态（2026-10-20 00:30）：#89 实机已注册 ALSA card，并在 41 秒
`sensor_process` 狗咬后保持 AP 存活、不再自动重启。#88 实机证明
`IPCRTR`/QRTR、ADSP audio PD、
APR、q6afe/q6adm、RX/TX SoundWire 与 WCD9385 均工作；补入
`q6asm-dais/dai@0..2` 后，#87 的 `No dais found in DT` / probe -22 已消失。
#88 无 sound card 的直接原因改为 card DAPM 路由失败：下游三组反向
`MIC BIASx <- Handset/Headset/Secondary Mic` 路由在主线 WCD9385 组件中
没有对应 widget，`devm_snd_soc_register_card()` 因此中止。#89 已删除这三组
不适用于主线模型的路由，补 VA macro `vdd-micb`、APR intents 与 glink SSR
通道，并修复 q6dsp clock provider 的 drvdata 初始化顺序。ADSP
`sensor_process` 40 秒狗咬仍独立存在；#88 在自动 SSR 的
`recovering adsp` 阶段挂死并被外部复位，#89 暂由 initramfs 禁用 ADSP
自动 SSR 以保住 AP shell；该保护已被实机验证。播放/录音路径与
sensor_process 根因仍待测试和处理。

后续 #90 已实机证明 `msm/adsp/sensor_pd` PDR listener/ACK 正常，但
`sensor_process` 仍在 41 秒狗咬。#91 按 SM7225 主线 ADSP 模型补入
`fastrpcglink-apps-dsp` 与 live ADSP stream ID `0x1403..0x1405`。#91/#92
误把 downstream cb10..cb12 节点序号当 mainline reg，导致 sensor FastRPC
invoke 立即死机；#93 已修正 reg=3/4/5，ATTACH_SNS 成功但仍狗咬。#94 补
SMP2P sleepstate 与 apps-awake bit12，见
[ADSP sensor_process/PDR 修复](2026-10-19-adsp-sensor-pd.md)。

## 1. 数据源与架构

按项目数据源等级：

- 下游 4.19 工厂 DTS：`lito-audio.dtsi`、`picasso-audio-overlay.dtsi`
  提供板级路由、GPIO、micbias 与宏地址；
- `live_dts.dts`：提供 ADSP/SMP2P/IPCC、RX/TX/VA macro、SoundWire IRQ、
  WCD9385 slave 地址、复位脚与供电的运行时事实；
- 主线：复用 SM8250 QDSP6 machine、PAS remoteproc、WCD938x、LPASS macro
  与 SoundWire 驱路。

本轮采用主线模型：

```text
remoteproc ADSP (PAS)
  └ glink-edge / APR
      ├ q6afe dais + q6afe clocks
      ├ q6asm dais
      └ q6adm routing

WCD9385
  ├ RX SoundWire slave @ port 4
  └ TX SoundWire slave @ port 3

LPASS macros
  ├ RX macro @0x3200000 + swr1 @0x3210000
  ├ TX macro @0x3220000 + swr2 @0x3230000
  └ VA macro @0x3370000

LPASS LPI pinctrl @0x33c0000 / slew @0x3550000
```

WSA macro 与 WSA881x 在 live/picasso overlay 中均为 disabled，本树不启用。

## 2. 源码修改

- `drivers/remoteproc/qcom_q6v5_pas.c`
  - 新增 `qcom,sm7250-adsp-pas`；
  - 复用 SM8250 ADSP PAS 资源：PAS id 1、LCX/LMX、minidump 5。
- `sound/soc/qcom/sm8250.c`
  - 新增 `qcom,sm7250-sndcard`；
  - 复用 SM8250 QDSP6 machine 行为与 WCD jack 初始化。
- `drivers/soc/qcom/qcom_pd_mapper.c`
  - 新增 `qcom,sm7250`，映射到 SM6350/lito 家族 PD 表；
  - ADSP audio/root PD 均为 instance 74，与 `adspr.jsn/adspua.jsn` 一致。
- bindings：
  - `qcom,sm8150-pas.yaml` 增加带 SM8250 fallback 的 SM7250 ADSP；
  - `qcom,sm8250.yaml` 增加 SM7250 sound card。

## 3. 设备树

新增：

- ADSP reserved memory 的 phandle；
- SMP2P ADSP（SMEM 443/429，IPCC LPASS/SMP2P）；
- ADSP remoteproc + glink/APR + q6afe/q6asm/q6adm；
- ADSP glink 的 `qrtr-requests`/`IPCRTR` 通道与 live intents；
- RX/TX/VA macro 与 RX/TX SoundWire master；
- WCD9385 RX/TX SoundWire slave；
- LPASS LPI pinctrl 的 RX/TX SoundWire active 状态；
- PM8150L BOB 3.008..3.96V，作为 WCD9385 mic-bias 供轨；
- root WCD9385 codec；
- SM7250 sound card 与 DAI links。

板级事实：

- WCD9385 reset：TLMM gpio57，active low；
- RX SWR IRQ：GIC SPI155；
- TX SWR IRQ：GIC SPI166；
- RX/TX macro地址：0x3200000/0x3220000；
- RX/TX SoundWire 地址：0x3210000/0x3230000；
- VA macro：0x3370000；
- LPASS LPI：0x33c0000 + slew 0x3550000；
- micbias1/2/3/4：2750mV；
- AMIC1/2/5 路由与 live/picasso overlay 一致；
- VA DMIC0..5 与 VA ADC 路由与 live/picasso overlay 一致。

DAI links：

- MultiMedia1/2/3 frontend；
- WCD Playback：`RX_CODEC_DMA_RX_0`；
- WCD Capture：`TX_CODEC_DMA_TX_3`；
- VA Capture：`VA_CODEC_DMA_TX_0`。

## 4. 配置

新增：

```text
CONFIG_SND=y
CONFIG_SND_SOC=y
CONFIG_SND_SOC_QCOM=y
CONFIG_SND_SOC_SM8250=y
CONFIG_SND_SOC_QDSP6=y
CONFIG_SND_SOC_WCD938X_SDW=y
CONFIG_SND_SOC_LPASS_RX_MACRO=y
CONFIG_SND_SOC_LPASS_TX_MACRO=y
CONFIG_SND_SOC_LPASS_VA_MACRO=y
CONFIG_SOUNDWIRE=y
CONFIG_SOUNDWIRE_QCOM=y
CONFIG_REGMAP_SOUNDWIRE=y
CONFIG_PINCTRL_LPASS_LPI=y
CONFIG_PINCTRL_SM8250_LPASS_LPI=y
CONFIG_REMOTEPROC=y
CONFIG_QCOM_Q6V5_PAS=y
CONFIG_RPMSG_QCOM_GLINK_SMEM=y
CONFIG_QCOM_APR=y
CONFIG_QCOM_PD_MAPPER=y
CONFIG_QRTR=y
CONFIG_QRTR_SMD=y
```

`scripts/build-boot.sh` 已增加强制检查，避免音频配置静默回退。

## 5. ADSP 固件

live 下游节点为：

```text
qcom,firmware-name = "adsp"
```

主线 DT 使用：

```text
firmware-name = "qcom/sm7250/adsp.mdt"
```

`scripts/make-initramfs.sh` 会自动打包：

```text
scripts/initramfs/firmware/qcom/sm7250/adsp.mdt
scripts/initramfs/firmware/qcom/sm7250/adsp.bXX
```

用户提供的 `android_firmware_dump/firmware_mnt/image/` 已包含配套
`adsp.mdt` 与 `adsp.b00..b32`。34 个文件已复制到：

```text
scripts/initramfs/firmware/qcom/sm7250/
```

源/目标逐文件 SHA256 对照一致。`adsp.mdt` 入口为 `0x89200000`；最后一个
LOAD 段为 `0x8b45d000 + 0x5a3000 = 0x8ba00000`，与 live reserved-memory
`0x89200000..0x8b9fffff` 精确匹配，证明这组固件与当前 ADSP 内存区配套。

`make-initramfs.sh` 已把 34 个文件打包到：

```text
/lib/firmware/qcom/sm7250/adsp.mdt
/lib/firmware/qcom/sm7250/adsp.b00 .. b32
```

若后续从完整 `modem.img` 重新提取，可执行：

```sh
scripts/extract-adsp-firmware.sh modem.img
bash scripts/make-initramfs.sh
bash scripts/build-boot.sh
```

## 6. SmartPA 边界

live/picasso 使用 I2C1 SE8 上的 NXP TFA9874：

- address 0x34；
- reset TLMM gpio58；
- irq TLMM gpio65；
- ROM 有 `tfa98xx.cnt`，头部标识 `TFA9874`。

上游 `tfa989x` 只支持 TFA9890/9895/9897，不支持 TFA9874；小米 OSS 树也
未包含 `nxp,tfa98xx` 驱动源码。因此本轮先完成 WCD9385 耳机/听筒/麦克风
链。扬声器 TFA9874 需要用户提供可审查的 TFA9874 内核驱动源码后再移植，
不应凭 compatible 空挂一个无驱动节点。

## 7. 实机结果与 #87 修复

### 7.1 #86 实机结果

ADSP 固件加载成功：

```text
remoteproc0: Booting fw image qcom/sm7250/adsp.mdt
remoteproc0: remote processor adsp is now up
```

但音频链未建立：

```text
PDR: service lookup for tms/servreg failed: -6
ALSA device list: No soundcards found
```

40 秒后 ADSP 崩溃：

```text
sensor_process ... USER-PD DOG detects stalled initialization
```

复核 live DTS 发现 ADSP glink-edge 还有 `qcom,adsp_qrtr`：

```text
qcom,glink-channels = "IPCRTR"
qcom,low-latency
qcom,intents = <2048 5 8192 3 17408 2>
```

#86 遗漏了该节点，且 `.config` 只有 `CONFIG_QRTR=y`，没有
`CONFIG_QRTR_SMD=y`。因此 AP 与 ADSP 间没有 IPC Router 传输，远程 QMI
服务无法通信，ADSP 内部依赖 QMI 的进程最终狗咬。

### 7.2 #87 修复与构建

已补：

- DT：`qrtr-requests` / `IPCRTR` / low-latency / live intents；
- config：`CONFIG_QRTR_SMD=y`；
- PD mapper：启动成功时打印平台与 domain 数，便于确认本地 servreg
  locator 是否起来。

完整构建通过：

```text
Linux #87
boot_7.3.img SHA256:
2b448c416e3556283e55b64cf60ed3c809e1c68cd9b9205c2d02d6a5adcba2e4
```

回归重点：

```sh
dmesg | grep -iE 'PDM|qrtr|PDR|servreg|remoteproc|apr|q6afe|q6asm|q6adm|soundwire|wcd938'
cat /proc/asound/cards
```

#87 期望 `PDR ... failed: -6` 消失，APR 服务能注册；若仍有失败，先根据
新增 `PDM: platform ... ready` 判断本地 PD mapper 是否启动。

### 7.3 #87 实机结果与 #88 修复

#87 中 `IPCRTR` 已生效：

```text
PDM: platform 'qcom,sm7250' ready with 5 domains
PDR: Indication received from msm/adsp/audio_pd, state: 0x1fffffff
Adding APR/GPR dev: service:4:4
Adding APR/GPR dev: service:4:7
Adding APR/GPR dev: service:4:8
wcd938x_codec: bound sdw:2:...:4
wcd938x_codec: bound sdw:3:...:3
```

说明 PD mapper、QRTR、APR、q6afe、q6adm、RX/TX SoundWire 与 WCD9385
均已工作。启动初期的 `PDR ... -6` 是查询早于 audio PD 上线的竞态；
随后 audio PD indication 已证明它不再是阻塞。

无 sound card 的直接错误：

```text
q6asm-dai ... No dais found in DT
q6asm-dai ... probe failed with error -22
```

#88 修复：

```text
q6asm-dais {
    dai@0 { reg = <0>; };
    dai@1 { reg = <1>; };
    dai@2 { reg = <2>; };
}
```

并按 live DTS 给 VA macro 补：

```text
qcom,dmic-sample-rate = <600000>;
```

#88 `boot_7.3.img` SHA256：
`68e748f70c0e8192d16963f071c82aa7885b9e339547ab6243680886fa83cd7b`。

#87 仍有一个独立问题：ADSP 内 `sensor_process` 在约 40 秒狗咬并触发
SSR。SSR 后 q6afe clock provider 重探时，旧的 macro clock 消费者仍持有
悬空 clock，导致 `clk_q6dsp_prepare()` NULL deref。若 #88 仍复现，需要
优先处理 sensor_process 稳定性或暂时禁用自动 SSR，再修 q6afe clock
provider 的生命周期。

### 7.4 #88 实机结果与 #89 修复

#88 日志（`uart_early.log`，2026-10-19 22:29）中 q6asm DAI 已正常注册，
不再出现：

```text
q6asm-dai ... No dais found in DT
q6asm-dai ... probe failed with error -22
```

但 sound card 仍未注册，决定性错误为：

```text
snd-sm8250 sound: ASoC: Failed to add route Handset Mic(*) -> MIC BIAS1
snd-sm8250 sound: ASoC: Failed to add route Headset Mic(*) -> MIC BIAS2
snd-sm8250 sound: ASoC: Failed to add route Secondary Mic(*) -> MIC BIAS3
ALSA device list: No soundcards found.
```

这不是 q6asm 回归。下游 machine 驱动包含物理 microphone widget，因此其
`"MIC BIASx", "xxx Mic"` 反向路由可解析；主线 WCD9385 组件只提供
`AMIC1/2/5` 输入与 `MIC BIASx` supply，上游 SM8250 DT 也只保留
`"AMICx", "MIC BIASx"` 方向。#89 删除三组反向路由。

同轮确认并修复：

- VA macro 缺 `vdd-micb-supply`，#88 使用 dummy regulator；#89 接到 live
  DTS 已证的 BOB（phandle `0x11c`）。
- APR 通道补 live 的 `qcom,intents = <0x200 0x14>`；glink-edge 补
  `glink_ssr` 通道声明。
- #87 SSR 后 `clk_q6dsp_prepare()` NULL deref 的直接驱动侧问题是
  `q6dsp_clock_dev_probe()` 在注册完 clock provider 之后才 `dev_set_drvdata()`。
  首个 clock 注册/reparent 阶段可能回调 ops，此时 `dev_get_drvdata()`
  仍为 NULL。#89 把 drvdata 赋值移到 clock 注册前。
- #88 在 41.4s 再次出现 `sensor_process ... dog_virtual_user`，随后仅打印
  `remoteproc0: recovering adsp` 即停住，未到达 #87 的 stopped/oops 阶段，
  最终外部复位。为避免每轮音频测试重启，#89 的 `/init` 按 remoteproc
  `name=adsp` 将 recovery 设为 disabled；这是 sensor_process 未解决前的
  bring-up 保护，不是长期方案。

#89 完整构建通过，产物：

```text
boot_7.3.img SHA256:
2440926456895b508952a7bdba4e846bd21c7e25008b3e8b274659885ea56639

runtime DTB SHA256:
7f9956bb527a5d2146e6477b7766d2cc4151b4ed2ca82f3dbd080656a40bf38e
```

回归判据：

```sh
dmesg | grep -iE 'q6asm|Failed to add route|snd-sm8250|ALSA|remoteproc|sensor_process'
cat /proc/asound/cards
cat /proc/asound/devices
ls /dev/snd
cat /sys/class/remoteproc/remoteproc*/name
cat /sys/class/remoteproc/remoteproc*/recovery
```

期望：

- 不再出现三条 `Failed to add route`；
- `sm7250-sndcard` 出现在 `/proc/asound/cards`；
- 41s sensor_process 狗咬仍可能出现，但 AP 不应挂死/重启；
  `/sys/class/remoteproc/remoteproc*/recovery` 对 adsp 应为 `disabled`；
- 后续需单独处理 sensor_process，再恢复自动 SSR 并删除 initramfs 保护。

### 7.5 #89 实机结果（2026-10-19 23:02）

#89 日志对应内核 `g115fe1f0c828-dirty` / 构建时间 22:37:26。核心判据
全部命中：

```text
input: Xiaomi Redmi K30 5G (Picasso) W Headset Jack
       as /devices/platform/sound/sound/card0/input1

ALSA device list:
  #0: Xiaomi Redmi K30 5G (Picasso) WCD9385
```

同时确认：

- #88 的三条 `ASoC: Failed to add route ... (*) -> MIC BIASx` 全部消失；
- q6asm FE DAI 正常注册，无 `-22`；
- RX/TX SoundWire 与 WCD9385 绑定正常；
- Headset Jack input 设备创建成功；
- 触摸、UFS、显示、GPU 启动链保持正常。

41.448s 仍出现独立 ADSP 故障：

```text
EF:sensor_process:0x1:TMR_CLNT_1:0xff:dog_virtual_user
USER-PD DOG detects stalled initialization
remoteproc0: crash detected in adsp
remoteproc0: handling crash #1 in adsp
```

由于 `/init` 已禁用 adsp 自动 recovery，日志停在 `handling crash`，没有
进入 #88 的 `recovering adsp` teardown，也没有 Oops 或整机复位。用户确认
系统这次没有重启。这闭环了两个结论：

1. #88 的重启触发点确实是 automatic SSR 路径；
2. #89 的 initramfs 保护能有效保住 AP shell。

本轮新增观察：

```text
failed to announce new service
failed while handling packet from 5:-2
```

位置在 ADSP up 后、APR audio PD indication 前，来自 QRTR nameserver
向 remote node 广播本地 service 的路径；audio PD/APR 随后正常上线，
因此它不是 sound card 注册阻塞。q6afe 在 5.68s 仍有两次
`cmd = 0x100f6 returned error = 0x1`，需要在实际 PCM open 时再判断是否
影响播放。

下一步：

1. 在 41s 前检查 `/proc/asound/cards`、`devices`、`/dev/snd` 与 card0
   控件，确认用户态可见性；
2. 向 initramfs 加入最小 ALSA PCM 测试工具，先做 playback/capture
   open + hw_params，再进入真实音频流；
3. 回归 #94 的 SMP2P sleepstate 日志；若 60 秒仍狗咬，下一轮
   单独移植 SMP2P sleepstate，而不是把 #89 的 recovery guard 固化；
4. sensor_process 修复后恢复 automatic SSR，并回归 q6dsp clock 生命周期
   修复。

### 7.6 #86 原始构建记录

完整构建通过：

```text
Linux #86
boot_7.3.img SHA256:
41d14bf9fcb69e2130396aee9689ec88a55095dbcdb69d1cb2cbb9f6e5c1cc9f
```

initramfs SHA256：
`0f816c8f21c3053d256a689484ae2761e6c2df0ab6186d9b225ea635428586c7`。
检查产物在 `builds/audio-bringup-checks/`：内核独立补丁、DTB、完整
`.config`、音频 config fragment、ADSP 固件清单与 SHA256 清单。diff
checkpatch 为 0 error；唯一提示是 binding 应与 C 代码拆分提交，属提交
拆分要求。

固件补齐后实机判据：

```sh
dmesg | grep -iE 'remoteproc|adsp|apr|q6afe|q6asm|soundwire|wcd938|lpass|snd'
cat /proc/asound/cards
cat /proc/asound/devices
ls /dev/snd
cat /sys/kernel/debug/remoteproc/remoteproc*/state
```

期望：

- ADSP firmware 加载并 running；
- APR/q6afe/q6asm/q6adm 服务出现；
- RX/TX SoundWire 扫描到 WCD9385；
- WCD9385 component 注册；
- `sm7250-sndcard` 注册为 ALSA card；
- 耳机播放、耳机录音和 AMIC 路径可测试。
