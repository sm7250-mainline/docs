# 2026-10-20：ADSP sensor_process 排查与修改全记录

记录时间：2026-10-20 00:33 CST

本文是今天 `sensor_process` 专项的完整 chronological 记录，承接
2026-10-19 晚间的 #89/#90 音频与 PDR 实验。更细的代码级证据和每轮产物
见：

- [2026-10-19/20 ADSP sensor_process/PDR 详细记录](2026-10-19-adsp-sensor-pd.md)
- [音频链 bring-up](audio-bringup.md)

## 0. 当前结论

当前已确认：

1. **sensor PD PDR ACK 不是剩余根因。**  
   #90 已看到 `msm/adsp/sensor_pd` locator、listener、UP indication 和 ACK。

2. **ADSP FastRPC transport 已打通。**  
   #93 使用正确 SID 后，`fastrpcglink-apps-dsp`、cb3/cb4/cb5 以及
   `ATTACH_SNS` invoke 均成功。

3. **#91/#92 的立即死机根因已确认。**  
   错误把 downstream 平台节点序号 cb10/cb11/cb12 当成 mainline FastRPC
   `reg`/SID 使用，导致 DSP 收到 `SID 10 << 32` 的错误 consolidated IOVA。

4. **`sensor_process` 41 秒初始化狗咬仍存在。**  
   #93 中 shell 稳定、FastRPC attach 成功，但 41.475 秒仍触发
   `dog_virtual_user`。

5. **当前待测候选是 SMP2P sleepstate。**  
   #94 已按 live/downstream 补入 `sleepstate` / `sleepstate_see` entry 和
   apps-awake bit12 keepalive；完整构建通过，尚未实机回归。

当前 AP 侧保护仍未移除：

```text
/init 会把 adsp remoteproc recovery 设为 disabled
```

原因依然是历史 automatic SSR teardown 会挂死并被外部复位；在
`sensor_process` 稳定前不能恢复。

## 1. 记录边界与数据源

### 1.1 软件与板卡

```text
board: Xiaomi Redmi K30 5G / picasso
SoC:   Qualcomm SM7250 / lito
kernel: linux-7.3-rc6
```

当前内核本地构建字符串示例：

```text
Linux version 7.3.0-rc6-sm7250-parkyu-g115fe1f0c828-dirty ...
#99 SMP PREEMPT Tue Oct 20 00:32:36 CST 2026
```

内核 build counter 与项目迭代号不是同一套编号；本文按项目迭代号 #89..#94
记录。

### 1.2 数据源等级

继续沿用项目规则：

1. runtime/board 事实以 `live_dts.dts` 为唯一权威；
2. 行为语义以下游 4.19 工厂内核为准；
3. 主线 SM8250/SM7225 只用于架构模型对照，不盲抄硬件 ID；
4. 本轮所有 vendor 数据均来自本地已解包 ROM/固件，没有通过 adb 连接手机。

### 1.3 主要输入

```text
uart_early.log
live_dts.dts
kernel_devicetree-picasso-r-oss/factory_downstream(android)_4.19_dts/
Xiaomi_Kernel_OpenSource-picasso-r-oss/
android_firmware_dump/firmware_mnt/image/adsp.*
tools/rom-extract/work/vendor.img
```

`uart_early.log` 每轮实机后被覆盖，本文保留了关键输出摘录。

## 2. 轮次总览

| 轮次 | 主要假设 / 修改 | 实机结果 | 结论 |
| --- | --- | --- | --- |
| #89 | 音频 card 修复；禁用 ADSP automatic SSR | WCD9385 ALSA card 注册；AP 不再重启 | `sensor_process` 41 秒狗咬独立存在 |
| #90 | 补 `msm/adsp/sensor_pd` PDR listener/ACK | sensor PD UP 且 ACK 成功 | 仍 41 秒狗咬；PDR ACK 不是唯一依赖 |
| #91 | 补 ADSP FastRPC glink/cb；自动 CREATE_STATIC `sensorspd` | FastRPC topology probe 成功；进入可用 shell 前死机 | CREATE_STATIC 路径危险； topology 本身有效 |
| #92 | 默认改为 ATTACH_SNS，延迟 5 秒执行 | shell 出现，5 秒后仍死机 | 所有 sensor FastRPC invoke 均触发底层错误 |
| #93 | cb10..12 转换为 mainline reg/SID 3/4/5 | 不再死机；sid3 ATTACH_SNS 成功 | FastRPC 打通；但 41 秒狗咬仍存在 |
| #94 | 补 SMP2P sleepstate/sleepstate_see/bit12 | 已构建，待实机 | 当前候选 |

## 3. #89/#90 基线回顾

### 3.1 #89 已达成的音频链

#89 后启动日志已有：

```text
ALSA device list:
  #0: Xiaomi Redmi K30 5G (Picasso) WCD9385
```

同时：

```text
input: Xiaomi Redmi K30 5G (Picasso) W Headset Jack
```

说明 QRTR、PD mapper、APR、q6afe/q6asm/q6adm、RX/TX SoundWire 与
WCD9385 均已工作。

#89 保留问题：

```text
EF:sensor_process:0x1:TMR_CLNT_1:0xff:dog_virtual_user
USER-PD DOG detects stalled initialization
```

### 3.2 #90：最小 sensor PD PDR keepalive

下游 `adsprpc.c` 中有：

```c
#define SENSORS_PDR_ADSP_SERVICE_LOCATION_CLIENT_NAME "sensors_pdr_adsprpc"
#define SENSORS_PDR_ADSP_SERVICE_NAME                  "tms/servreg"
#define ADSP_SENSORPD_NAME                             "msm/adsp/sensor_pd"
```

live DT 的 downstream FastRPC 平台节点带有：

```dts
qcom,fastrpc-adsp-audio-pdr;
qcom,fastrpc-adsp-sensors-pdr;
```

因此 #90 新增：

```text
drivers/soc/qcom/qcom-adsp-sensor-pd.c
Documentation/devicetree/bindings/soc/qcom/qcom,adsp-sensor-pd.yaml
```

并在 DT 中加入：

```dts
adsp-sensor-pd {
    compatible = "qcom,adsp-sensor-pd";
    nvmem-cells = <&adsp_variant>;
    nvmem-cell-names = "adsp_variant";
};
```

驱动执行：

```c
pdr_add_lookup(pdr, "tms/servreg", "msm/adsp/sensor_pd");
```

### 3.3 #90 实机结果

成功链路：

```text
qcom-adsp-sensor-pd: adsp_variant fuse value: 0x03
qcom-adsp-sensor-pd: tracking tms/servreg -> msm/adsp/sensor_pd

PDR: Located msm/adsp/sensor_pd using tms/servreg: instance=74
PDR: Listener registered for msm/adsp/sensor_pd, state: 0x7fffffff
PDR: Indication received from msm/adsp/sensor_pd, state: 0x1fffffff

qcom-adsp-sensor-pd:
sensor PD msm/adsp/sensor_pd state up (0x1fffffff)

PDR: Acknowledging msm/adsp/sensor_pd transaction 1
```

但仍在约 41.5 秒狗咬：

```text
EF:sensor_process:0x1:TMR_CLNT_1:0xfc:dog_virtual_user
```

与 #89 的 `0xff` 相比变为 `0xfc`，说明新增 listener 改变了初始化状态，但
不能把该值直接解释为 bitmask。结论是仍有其他启动依赖未满足。

## 4. 固件逆向

### 4.1 ADSP split image 重建方法

`adsp.mdt` / `adsp.b00` 为 ELF32 Hexagon 头和 program headers：

```text
Entry point: 0x89200000
Program headers: 34
```

Qualcomm split image 的映射关系：

```text
adsp.b00 -> ELF header + phdrs + phdr 0 payload
adsp.b01 -> phdr 1
adsp.b02 -> phdr 2
...
```

重建原则：

1. 以 `adsp.b00` 为基础；
2. 解析每个 phdr 的 `p_offset` / `p_filesz`；
3. 将对应 `adsp.bNN` 写入 reconstructed ELF 的 `p_offset`；
4. 空洞补零；
5. 用 LLVM 21 反汇编：

```sh
/usr/lib/llvm-21/bin/llvm-objdump \
  -d --triple=hexagon-unknown-elf adsp-reconstructed.elf
```

本轮重建时踩过一个坑：phdr 1 的 ELF type 是 `NULL` 但仍对应真实
`adsp.b01` payload，不能按 “非 LOAD 全跳过” 处理，否则会覆盖 ELF header。

### 4.2 固件关键字符串

重建后检索到：

```text
fastrpcglink-apps-dsp
fastrpc_send_early_signal
sns_user_pd_init.c:init countdown %d
rcinit_lite_task.c:COMMON:RCInit Processing Done %c
msm/adsp/sensor_pd
sensor_process
sleepstate
sleepstate_see
Remote_proc_state
Servreg ack timedout
USER-PD DOG detects stalled initialization
```

这些字符串支持以下模型：

```text
ADSP image 内置 sensor_process
  ├─ sensor PD / servreg handshake
  ├─ FastRPC / early signal
  ├─ sns user PD init countdown / RCInit
  └─ remote processor state / sleepstate
```

#90 只解决了 servreg ACK，未解决 countdown 的其他等待项。

## 5. vendor userspace 逆向

使用本地 EROFS 工具读取 `tools/rom-extract/work/vendor.img`，未连接手机。

### 5.1 early_hal 服务

发现：

```text
/etc/init/vendor.sensors.sscrpcd.rc
```

内容关键项：

```text
service vendor.sensors /vendor/bin/sscrpcd sensorspd
    class early_hal
```

说明 sensor FastRPC client 在 Android early_hal 阶段启动，而不是等普通
sensor HAL 才启动。

### 5.2 ARM64 ELF 行为

逆向 `/vendor/bin/sscrpcd`：

1. `dlopen("libadsp_default_listener.so")`；
2. `dlsym("adsp_default_listener_start")`；
3. 调用该函数。

逆向 `libadsp_default_listener.so`：

1. 构造特殊 remote handle 名称：

```text
'":;./\createstaticpd: + sensorspd
```

2. 设置：

```text
ADSP_LISTENER_MEM_CACHE_SIZE=1048576
```

3. `remote_handle_open()`；
4. 打开/调用 `adsp_default_listener`；
5. 通过 `geteventfd` 长期 poll，保持 client 存活。

继续逆向 `libadsprpc.so` 可见，特殊前缀由用户库识别；真正传给 DSP 的
static PD 名是后缀 `sensorspd`。

这解释了为什么只有 kernel PDR listener 不够：下游还有 early_hal userspace
的 FastRPC create/attach/持 fd 契约。

## 6. 主线架构对照

### 6.1 SM8250

SM8250 不是本机直接模型：

```text
SM8250 ADSP: audio / FastRPC
SM8250 SLPI: sensor_process
```

主线启用独立 SLPI：

```dts
&slpi {
    status = "okay";
    firmware-name = "qcom/sm8250/slpi.mbn";
};
```

其 FastRPC 节点在 SLPI edge，`label = "sdsp"`，stream ID 是
`0x541..0x543`。

### 6.2 SM7225 / SM6350

SM7225 更接近本机：

```text
SM7225 ADSP 集成 sensor firmware
ADSP glink edge 内直接有 fastrpc 子节点
```

主线 SM6350 ADSP FastRPC 模型：

```dts
fastrpc {
    compatible = "qcom,fastrpc";
    qcom,glink-channels = "fastrpcglink-apps-dsp";
    label = "adsp";
    qcom,non-secure-domain;

    compute-cb@3 { ... };
    compute-cb@4 { ... };
    compute-cb@5 { ...; qcom,nsessions = <5>; };
};
```

该对照确立了 #91 的方向：补主线 FastRPC transport 和 ADSP compute
banks。

## 7. live FastRPC 事实与 #91 初版实现

### 7.1 live glink 通道

live ADSP glink edge：

```dts
qcom,msm_fastrpc_rpmsg {
    compatible = "qcom,msm-fastrpc-rpmsg";
    qcom,glink-channels = "fastrpcglink-apps-dsp";
    qcom,intents = <0x64 0x40>;
};
```

主线 binding 未声明 `qcom,intents`，#91 先按 SM7225 主线模型使用默认
intent 行为。

### 7.2 live ADSP banks

downstream 平台节点中的 ADSP 专用 banks：

```text
cb10: label adsprpc-smd, apps_smmu stream 0x1403
cb11: label adsprpc-smd, apps_smmu stream 0x1404
cb12: label adsprpc-smd, apps_smmu stream 0x1405, shared-cb = 5
```

#91 初版将其写成：

```dts
compute-cb@10 {
    reg = <10>;
    iommus = <&apps_smmu 0x1403 0x0>;
};

compute-cb@11 {
    reg = <11>;
    iommus = <&apps_smmu 0x1404 0x0>;
};

compute-cb@12 {
    reg = <12>;
    iommus = <&apps_smmu 0x1405 0x0>;
    qcom,nsessions = <5>;
};
```

这个映射后来被证明是错误：cb10..12 是 downstream 平台容器中的节点序号，
不是 mainline FastRPC SID。

### 7.3 #91 helper

新增：

```text
scripts/fastrpc-sensor-attach.c
scripts/initramfs/fastrpc-sensor-attach
```

静态 AArch64 helper 使用 NDK r28c 编译：

```sh
aarch64-linux-android23-clang \
  -O2 -Wall -Wextra -Werror -static \
  -o scripts/initramfs/fastrpc-sensor-attach \
  scripts/fastrpc-sensor-attach.c
llvm-strip scripts/initramfs/fastrpc-sensor-attach
```

初版行为：

1. 等待 `/dev/fastrpc-adsp`；
2. `FASTRPC_IOCTL_INIT_CREATE_STATIC`：

```text
name="sensorspd"
memlen=1MiB
```

3. 再 `FASTRPC_IOCTL_INIT_ATTACH_SNS`；
4. 保持 fd。

### 7.4 #91 实机结果

FastRPC topology 全部成功：

```text
qcom,fastrpc ... channel probe: domain=adsp id=0 secure=0 vmids=0 poll=0

qcom,fastrpc-cb ... compute bank sid=10 ... total_sessions=1
qcom,fastrpc-cb ... compute bank sid=11 ... total_sessions=2
qcom,fastrpc-cb ... compute bank sid=12 ... total_sessions=7

qcom,fastrpc ... probe complete:
sessions=7 users=0 node=fastrpc-adsp
```

但最后 durable 输出只有：

```text
fastrpc-sensor-attach: /dev/fastrpc-adsp opened
```

随后系统在可用 shell 稳定前死机。没有：

```text
create-static request
create-static complete
```

当时的初步结论是 CREATE_STATIC / 1MiB remote heap 路径危险，还不能否定
FastRPC topology。

## 8. #92：隔离 CREATE_STATIC 与 ATTACH_SNS

#92 修改 helper：

```sh
# 默认，仅 ATTACH_SNS：
/bin/fastrpc-sensor-attach

# shell 稳定后手动 CREATE_STATIC：
/bin/fastrpc-sensor-attach --create-static
```

并把 `/init` 改为：

```sh
(
    sleep 5
    exec /bin/fastrpc-sensor-attach
) &
```

目的：

1. shell 先出现；
2. 5 秒后再执行最小 invoke；
3. 如果仍死机，则说明不是 CREATE_STATIC 独有问题。

### #92 实机结果

shell 确实出现，5 秒后死机。最后输出：

```text
fastrpc-sensor-attach: /dev/fastrpc-adsp opened (mode=attach-sns
```

行尾 `)` 和换行都未发出，随后没有 attach 结果。

结论：

```text
CREATE_STATIC 不是唯一触发条件；
任何 sensor FastRPC invoke 都会触发更底层错误。
```

## 9. #93：找到立即死机的真正根因

### 9.1 mainline `reg` 的语义

主线 `drivers/misc/fastrpc.c`：

```c
static inline u64 fastrpc_sid_offset(struct fastrpc_channel_ctx *cctx,
                                     struct fastrpc_session_ctx *sctx)
{
        return (u64)sctx->sid << cctx->soc_data->sid_pos;
}
```

默认：

```text
soc_data.sid_pos = 32
```

#91/#92 的 cb3 stream 使用 `reg=<10>`，因此发给 DSP 的地址前缀是：

```text
SID 10 << 32 = 0x0000000a00000000
```

但 stream ID `0x1403` 对应 DSP 侧 SID 3，应为：

```text
SID 3 << 32 = 0x0000000300000000
```

ADSP 按 SID 3 解析 context bank 时收到 SID 10 的合成地址，会在内核
timeout/error 打印前触发低级内存/保护故障。

### 9.2 主线平台交叉验证

主线已有映射：

```text
SM6350 CDSP:
compute-cb@3 -> apps_smmu 0x1403
compute-cb@4 -> apps_smmu 0x1404
compute-cb@5 -> apps_smmu 0x1405

SDM845 CDSP:
compute-cb@3 -> apps_smmu 0x1403
compute-cb@4 -> apps_smmu 0x1404
compute-cb@5 -> apps_smmu 0x1405
```

因此 live 的正确转换是：

```text
downstream cb10 -> mainline compute-cb@3, reg=3,  stream 0x1403
downstream cb11 -> mainline compute-cb@4, reg=4,  stream 0x1404
downstream cb12 -> mainline compute-cb@5, reg=5,  stream 0x1405
```

### 9.3 #93 DT 修正

当前 runtime DT：

```dts
compute-cb@3 {
    compatible = "qcom,fastrpc-compute-cb";
    reg = <3>;
    iommus = <&apps_smmu 0x1403 0x0>;
};

compute-cb@4 {
    compatible = "qcom,fastrpc-compute-cb";
    reg = <4>;
    iommus = <&apps_smmu 0x1404 0x0>;
};

compute-cb@5 {
    compatible = "qcom,fastrpc-compute-cb";
    reg = <5>;
    iommus = <&apps_smmu 0x1405 0x0>;
    qcom,nsessions = <5>;
};
```

### 9.4 #93 实机结果

不再死机。FastRPC bank 日志变为：

```text
compute bank sid=3 nsessions=1 ... total_sessions=1
compute bank sid=4 nsessions=1 ... total_sessions=2
compute bank sid=5 nsessions=5 ... total_sessions=7
```

5 秒后：

```text
[    7.099160] qcom,fastrpc ... user open: secure=0 sid=3 domain=0
[    7.099201] qcom,fastrpc-cb ... attach request: pd=2 sid=3
[    7.100774] qcom,fastrpc-cb ... attach complete: pd=2 sid=3
```

这证明：

- glink channel 正常；
- cb3/stream `0x1403` 正常；
- consolidated IOVA 编码正确；
- AP 可向 sensor PD 发送 `ATTACH_SNS`；
- #91/#92 立即死机根因确认。

但 41.475 秒仍狗咬：

```text
[   41.475101] PDM: service 'sensor_process' crash:
EF:sensor_process:0x1:TMR_CLNT_1:0xfc:dog_virtual_user

[   41.505595] USER-PD DOG detects stalled initialization
[   41.505652] remoteproc0: crash detected in adsp
```

## 10. #94：SMP2P sleepstate 实验

### 10.1 live/downstream 证据

live ADSP SMP2P edge 除 master/slave kernel 外还有：

```dts
sleepstate-out {
    qcom,entry-name = "sleepstate";
    #qcom,smem-state-cells = <1>;
};

qcom,sleepstate-in {
    qcom,entry-name = "sleepstate_see";
    interrupt-controller;
    #interrupt-cells = <2>;
};
```

独立 client：

```dts
qcom,smp2p_sleepstate {
    compatible = "qcom,smp2p-sleepstate";
    qcom,smem-states = <&sleepstate_smp2p_out 0>;
    interrupt-parent = <&sleepstate_smp2p_in>;
    interrupts = <0 0>;
    interrupt-names = "smp2p-sleepstate-in";
};
```

下游驱动：

```text
Xiaomi_Kernel_OpenSource-picasso-r-oss/drivers/soc/qcom/smp2p_sleepstate.c
```

probe 时固定：

```c
#define PROC_AWAKE_ID 12
qcom_smem_state_update_bits(state, BIT(PROC_AWAKE_ID),
                            BIT(PROC_AWAKE_ID));
```

注意：live DT 中的 `<sleepstate-out 0>` 是 legacy state 参数；下游并不用它
选择 awake bit，而是固定更新 bit 12。#94 保留 bit 12，并在日志里打印 DT
参数和实际更新位，避免再次混淆。

### 10.2 当前 DT

当前加入：

```dts
sleepstate {
    qcom,entry-name = "sleepstate";
    #qcom,smem-state-cells = <1>;
};

sleepstate-see {
    qcom,entry-name = "sleepstate_see";
    interrupt-controller;
    #interrupt-cells = <2>;
};

smp2p-sleepstate {
    compatible = "qcom,smp2p-sleepstate";
    qcom,smem-states = <&sleepstate 0>;
    interrupt-parent = <&sleepstate_see>;
    interrupts = <0 0>;
    interrupt-names = "smp2p-sleepstate-in";
};
```

### 10.3 当前 driver

新增：

```text
drivers/soc/qcom/qcom-smp2p-sleepstate.c
```

行为：

1. `devm_qcom_smem_state_get()` 获取 `sleepstate`；
2. `of_irq_get_byname(..., "smp2p-sleepstate-in")` 获取 `sleepstate_see` IRQ；
3. 请求 rising-edge threaded IRQ；
4. 固定置 `BIT(12)`；
5. remove 时清零；
6. 当前内核未启用 suspend，因此保持 boot 期间常 awake。

### 10.4 #94 中间修正

#94 开发中有三个中间版本，均未实机：

1. **初版直接移植 PM notifier / wakeup source**

   构建通过，但检查对象文件发现当前内核没有 `CONFIG_PM_SLEEP`，主线
   `wakeup_source_register()` inline stub 返回 `NULL`，driver 实际会在
   probe 中返回 `-ENOMEM`。该问题在刷机前通过 object/disassembly 检查发现。

2. **移除 PM/wakeup 依赖**

   改为常 awake 后，又发现初版使用 DT 参数 bit0 更新 state；这与下游固定
   bit12 的行为不一致，随即修正。

3. **最终清理**

   修正 indentation 和 unreachable return，得到当前待测镜像。

这轮再次验证了 “构建通过不等于 probe path 有效”，必须检查最终对象代码。

## 11. 当前修改清单

### 11.1 内核 / DT

| 文件 | 目的 |
| --- | --- |
| `arch/arm64/boot/dts/qcom/sm7250-xiaomi-picasso.dts` | qfprom adsp variant、sensor PDR keepalive 节点、FastRPC cb3/4/5、SMP2P sleepstate entry/client、此前音频修复 |
| `drivers/soc/qcom/qcom-adsp-sensor-pd.c` | 最小 `msm/adsp/sensor_pd` PDR listener/ACK |
| `drivers/soc/qcom/qcom-smp2p-sleepstate.c` | ADSP sleepstate apps-awake bit12 keepalive |
| `drivers/soc/qcom/pdr_interface.c` | locator/listener/indication/ACK 日志 |
| `drivers/soc/qcom/smp2p.c` | inbound/outbound entry 注册与远端匹配日志 |
| `drivers/misc/fastrpc.c` | channel/cb/open/create-static/attach invoke 日志 |
| `net/qrtr/ns.c` | nameserver packet 错误细节 |
| `sound/soc/qcom/qdsp6/q6dsp-lpass-clocks.c` | #89 的 clock provider drvdata 顺序修复 |
| `drivers/soc/qcom/Kconfig` / `Makefile` | 新 driver 配置与构建 |

### 11.2 bindings

```text
Documentation/devicetree/bindings/soc/qcom/qcom,adsp-sensor-pd.yaml
Documentation/devicetree/bindings/soc/qcom/qcom,smp2p-sleepstate.yaml
Documentation/devicetree/bindings/soc/qcom/qcom,smp2p.yaml
```

`qcom,smp2p.yaml` 扩展允许：

```text
sleepstate
sleepstate-see
```

### 11.3 initramfs / 脚本

| 文件 | 目的 |
| --- | --- |
| `scripts/fastrpc-sensor-attach.c` | 静态 AArch64 helper 源码 |
| `scripts/initramfs/fastrpc-sensor-attach` | stripped static executable |
| `scripts/make-initramfs.sh` | 打包 `/bin/fastrpc-sensor-attach` |
| `scripts/initramfs-init.sh` | shell 启动 5 秒后执行默认 ATTACH_SNS |
| `scripts/build-boot.sh` | 强制检查 FastRPC/sleepstate/sensor-PD 配置 |

helper 当前默认模式：

```sh
/bin/fastrpc-sensor-attach
```

只执行：

```text
FASTRPC_IOCTL_INIT_ATTACH_SNS
```

手动危险实验：

```sh
/bin/fastrpc-sensor-attach --create-static
```

### 11.4 当前必需 config

```text
CONFIG_QCOM_ADSP_SENSOR_PD=y
CONFIG_QCOM_FASTRPC=y
CONFIG_QCOM_SMP2P_SLEEPSTATE=y
CONFIG_QCOM_PDR_HELPERS=y
CONFIG_QCOM_PD_MAPPER=y
CONFIG_QCOM_APR=y
CONFIG_QCOM_IPCC=y
CONFIG_QCOM_AOSS_QMP=y
CONFIG_RPMSG_QCOM_GLINK_SMEM=y
CONFIG_QRTR=y
CONFIG_QRTR_SMD=y
```

## 12. 排查日志设计

### 12.1 sensor PD / PDR

```text
adsp-sensor-pd: adsp_variant fuse value: 0x...
adsp-sensor-pd: tracking tms/servreg -> msm/adsp/sensor_pd
PDR: Located ...
PDR: Listener registered ...
PDR: Indication received ...
PDR: Acknowledging ...
```

### 12.2 FastRPC

```text
channel probe: domain=adsp id=0 secure=0 vmids=0 poll=0
compute bank sid=3 nsessions=1 dma_bits=32 total_sessions=1
compute bank sid=4 nsessions=1 dma_bits=32 total_sessions=2
compute bank sid=5 nsessions=5 dma_bits=32 total_sessions=7
probe complete: sessions=7 users=0 node=fastrpc-adsp

user open: secure=0 sid=3 domain=0
attach request: pd=2 sid=3
attach complete: pd=2 sid=3
```

异常路径：

```text
create-static heap alloc failed
create-static invoke failed
attach failed
DSP PD notification received
No context ID matches response
channel removed
```

### 12.3 SMP2P sleepstate

```text
registered outbound SMP2P entry 'sleepstate'
registered inbound SMP2P entry 'sleepstate_see'

matched early inbound SMP2P entry 'sleepstate_see', initial value 0x...
matched inbound SMP2P entry 'sleepstate_see', initial value 0x...

qcom-smp2p-sleepstate:
ADSP sleepstate ready: DT state bit 0, apps-awake bit 12 asserted

qcom-smp2p-sleepstate: sleepstate_see edge irq=...
```

这些日志可以区分：

- DT/config 未生效；
- outbound entry 未注册；
- client probe defer/fail；
- ADSP 未创建远端 `sleepstate_see` entry；
- bit12 已置但 sensor 仍狗咬；
- 远端 sleepstate edge 已到达。

## 13. 构建与验证记录

### 13.1 标准构建

修改 initramfs 后必须先执行：

```sh
bash scripts/make-initramfs.sh
```

再完整执行：

```sh
bash scripts/build-boot.sh
```

不能使用 `--skip-kernel`，因为内核内嵌 initramfs 和 FastRPC/sleepstate C
代码都会变化。

### 13.2 当前验证结果

当前 #94 候选镜像已通过：

```text
scripts/build-boot.sh
```

验证项：

- `qcom-smp2p-sleepstate.c` checkpatch：0 errors / 0 warnings；
- `fastrpc.c` diff checkpatch：0 errors / 0 warnings；
- runtime DTB 反编译确认：
  - FastRPC cb3/cb4/cb5；
  - stream `0x1403/0x1404/0x1405`；
  - sleepstate/sleepstate_see；
  - smp2p-sleepstate client；
- reserved-memory `no-map` 数量仍为 22；
- `CONFIG_QCOM_SMP2P_SLEEPSTATE=y`；
- initramfs cpio 包含 `/bin/fastrpc-sensor-attach`；
- helper 为 stripped static AArch64 executable；
- Image 中包含 sleepstate/FastRPC 排查日志；
- manifest 全部断言通过。

### 13.3 当前 #94 产物

```text
boot_7.3.img SHA256:
e954c55bd798622544aff449c1f997d4f59a30801eb3c54c86ee52f54620658f

runtime DTB SHA256:
dbe7a698e722a25055fe7aa40cd3d6d00bde0ab7d5373afb1639660d30ad0863

Image SHA256:
9076ef1de14786bc94c167c79e1418f4e3e1c6aeb25ca2cddf533dd1798a92a3

initramfs SHA256:
6111b8b285c042bdc659a6ae9abf6f2eb39c7fed12db4878982fccb332557043

fastrpc-sensor-attach SHA256:
048836acf05771783091ca7c617568eaa919b782a1dc9e698675ab1d03486bc2
```

### 13.4 关键历史产物

| 轮次 | 状态 | boot SHA256 |
| --- | --- | --- |
| #90 | 实机：PDR ACK 成功，仍狗咬 | `64ea315029081d1933889658349f7ae6a881f74e7b21a2e1be754b66819d97ec` |
| #91 初版 FastRPC only | 未按此 exact 产物继续实机 | `20917f382840cf2aa72364ea4ad41749e2577fd74a91be31c6ba2d54a1b78519` |
| #91 auto CREATE_STATIC | 实机：topology 成功，进入 shell 前死机 | `d960b56fcf4b0245fa685141162fd540da06993658789b0322433074ea1ccb53` |
| #92 ATTACH_SNS only | 构建后继续调整 delay | `00859ce2d03e1be86308e7545544be25fad17d5e14575ca8e67499162f814c5e` |
| #92 5 秒 delay | 实机：shell 后 5 秒死机 | `99ec98ccccaac87fec81a68271fe51b7c6e210fe9c508488124071ef1850eaca` |
| #93 reg=3/4/5 | 实机：ATTACH_SNS 成功，仍 41 秒狗咬 | `92d87e5e8809e4e3d8e507c404e87ae58767ec1e39cf8e823355ae313c006cfb` |
| #94 初版 sleepstate | 未刷机，发现 PM stub 问题 | `6ded88476afb47e105f9f8ba7be1b2cfcb89c9ae058b859d4b317370dc8633ac` |
| #94 去 PM stub | 未刷机，发现 bit0/bit12 解释问题 | `fb5d00a6a36a4c6249ce776755a46f6de96b50c4664f343041e79991574dee32` |
| #94 bit12 修正 | 未刷机，仅代码清理前 | `0d3d95bd0bd07adfd81a92901778b6fe52aa8fdf523398f446caefc8395ea496` |
| #94 当前清理版 | 待实机 | `e954c55bd798622544aff449c1f997d4f59a30801eb3c54c86ee52f54620658f` |

## 14. #94 实机回归步骤

刷入：

```sh
fastboot boot boot_7.3.img
```

至少观察 60 秒。重点命令：

```sh
dmesg | grep -iE \
  'smp2p|sleepstate|fastrpc|sensor-attach|PDR:|sensor_process|remoteproc|dog_virtual'

cat /sys/kernel/debug/remoteproc/remoteproc*/state
```

### 14.1 必须出现的 sleepstate 证据

```text
registered outbound SMP2P entry 'sleepstate'
registered inbound SMP2P entry 'sleepstate_see'

qcom-smp2p-sleepstate:
ADSP sleepstate ready: DT state bit 0, apps-awake bit 12 asserted
```

理想情况下还会看到：

```text
matched early inbound SMP2P entry 'sleepstate_see', initial value 0x...
```

### 14.2 必须保留的 FastRPC 证据

```text
compute bank sid=3 ...
compute bank sid=4 ...
compute bank sid=5 ...

fastrpc ... attach request: pd=2 sid=3
fastrpc ... attach complete: pd=2 sid=3
```

### 14.3 成功标准

60 秒内没有：

```text
sensor_process ... dog_virtual_user
remoteproc0: crash detected in adsp
```

## 15. 若 #94 仍失败的后续路线

按优先级：

1. **完整复刻 `libadsp_default_listener.so` 行为**  
    不仅 ATTACH_SNS，还要执行 corresponding remote invoke 并保持 eventfd
    listener。

2. **继续逆向 `sns_user_pd_init.c:init countdown`**  
    找出 countdown 等待的具体 signal / remote state。

3. **采集 remoteproc coredump / minidump**  
    在 41 秒狗咬前开启 coredump，确认 USER-PD 栈和等待对象。

4. **核查 ADSP firmware variant**  
    live adsp-loader 使用 `adsp-fw-names = "adsp2"`、
    `adsp-fw-bit-values = <0x2>`；当前 fuse 读数为 `0x03`。需要确认分区中
    `adsp.mdt` 是否就是该 variant 应选图像。

5. **恢复 automatic SSR**  
    只有 `sensor_process` 不再狗咬后才能测试 recovery，并回归 q6dsp clock
    provider 生命周期。

## 16. 经验与防回归规则

1. **downstream 平台节点序号不能直接当 mainline `reg`。**  
   cb10..12 只是 downstream 平台容器里的子节点序号；mainline `reg` 是
   FastRPC SID，应与 stream ID 低位对应。

2. **每个危险 ioctl 必须独立隔离。**  
   CREATE_STATIC 与 ATTACH_SNS 分开测试，才定位到共同底层 SID 错误。

3. **先保 shell，再自动执行可能致命的 remote invoke。**  
   #92 的 5 秒 delay 让 “shell 是否已出现” 和 “invoke 是否死机” 可以分开
   归因。

4. **构建通过不等于 probe path 有效。**  
   #94 初版的 PM/wakeup inline stub 问题就是通过对象代码检查发现的。

5. **保持 live DT 权威，但要理解属性语义。**  
   `<sleepstate-out 0>` 的 `0` 不代表下游实际更新 bit0；下游代码固定更新
   bit12。

6. **不把 automatic SSR 与未稳定的 ADSP firmware 混测。**  
   当前继续禁用 recovery，避免历史 teardown 挂死干扰归因。
