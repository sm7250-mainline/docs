# 2026-10-19：ADSP sensor_process 剩余狗咬的下游对照修复

状态（2026-10-20 00:33）：#89 已注册 WCD9385 ALSA card，但 ADSP 仍在约
41 秒报 `sensor_process` USER-PD 初始化狗咬。本轮详细对照下游 4.19
adsprpc、`live_dts.dts` 与 ADSP 固件字符串后，确认一个此前遗漏的运行时
依赖：下游会为 `msm/adsp/sensor_pd` 注册 PDR listener 并确认状态。
#90 已实机回归：sensor PD locator/listener/indication/ACK 全链路工作，但
`sensor_process` 仍在 41.5 秒狗咬。#91 继续按 SM7225 主线模型补入 ADSP
FastRPC glink 通道、live 推导的 compute context banks，并按下游
`sscrpcd sensorspd` 逆向结果加入最小 sensor static-PD bootstrap；#91 实机
证明 FastRPC 拓扑全部 probe 成功，但自动 CREATE_STATIC 在进入可用 shell
前触发死机。#92 已把自动动作收窄为 ATTACH_SNS，CREATE_STATIC 仅保留为
手动调试模式，并把自动 attach 延迟 5 秒以保证 shell 先出现；#92 实机证明
shell 稳定 5 秒后仍死机。逆向主线 FastRPC SID 编码后确认 #91/#92 把
mainline `reg` 误用为 live 的 cb10..cb12 节点编号；#93 已按 stream ID 低位
修正为 reg 3/4/5。#93 实机证明 SID 3 的 ATTACH_SNS 已成功且 shell 稳定，
但 41 秒 sensor_process 狗咬仍存在；#94 已按 live/downstream 补入
SMP2P sleepstate / sleepstate_see 与 apps-awake bit12 keepalive，完整构建
通过，待实机回归。

本文件的按天 chronological 汇总见
[2026-10-20 ADSP sensor_process 排查与修改全记录](2026-10-20-adsp-sensor-process-debug-record.md)。

## 1. #89 后的边界

#89 成功项：

- WCD9385 ALSA card 注册；
- Headset Jack input 创建；
- q6asm FE DAI 无 `-22`；
- 三条错误方向的 microphone route 已消失；
- 禁用 ADSP automatic SSR 后，AP 不再因 recovery teardown 重启。

#89 保留问题：

```text
41.448s
EF:sensor_process:0x1:TMR_CLNT_1:0xff:dog_virtual_user
USER-PD DOG detects stalled initialization
remoteproc0: crash detected in adsp
remoteproc0: handling crash #1 in adsp
```

由于 recovery 已禁用，日志停在 `handling crash`，没有进入 #88 的
`recovering adsp` 挂死路径。该问题独立于 ALSA card 注册。

## 2. 下游与 live DTS 对照

### 2.1 live DTS 的关键差异

`live_dts.dts` 中 ADSP glink edge 不仅有 `IPCRTR`、APR 和 glink SSR，还
声明了 fastrpc 通道：

```text
/soc/qcom,glink/adsp/qcom,msm_fastrpc_rpmsg {
    qcom,glink-channels = "fastrpcglink-apps-dsp";
};
```

更重要的是，live 的 fastrpc 平台节点带有：

```text
/soc/qcom,msm_fastrpc {
    compatible = "qcom,msm-fastrpc-compute";
    qcom,fastrpc-adsp-audio-pdr;
    qcom,fastrpc-adsp-sensors-pdr;
};
```

本轮之前的主线 DT 只复用了 APR/audio PD：

```text
avs/audio -> msm/adsp/audio_pd
```

没有等价执行：

```text
tms/servreg -> msm/adsp/sensor_pd
```

### 2.2 下游 adsprpc 的实际行为

下游 `drivers/char/adsprpc.c` 定义：

```c
#define SENSORS_PDR_ADSP_SERVICE_LOCATION_CLIENT_NAME "sensors_pdr_adsprpc"
#define SENSORS_PDR_ADSP_SERVICE_NAME                  "tms/servreg"
#define ADSP_SENSORPD_NAME                             "msm/adsp/sensor_pd"
```

probe 检测到 `qcom,fastrpc-adsp-sensors-pdr` 后：

1. 调 `get_service_location()` 查询 `tms/servreg`;
2. 在返回 domain list 中寻找 `msm/adsp/sensor_pd`;
3. 调 `service_notif_register_notifier()` 注册 listener;
4. 收到 UP/DOWN indication 后更新状态。

下游日志字符串也会明确打印：

```text
adsprpc: ... PDR notifier for msm/adsp/sensor_pd registered
adsprpc: ... msm/adsp/sensor_pd is up on adsp
```

### 2.3 ADSP 固件侧证据

对用户提供的 `adsp.b*` 逐段 `strings` 检查，能看到：

```text
msm/adsp/sensor_pd
sensor_process
Servreg ack timedout. Check coredump.err.aux_msg
Servreg ack timedout:%s:State:0x%x
USER-PD DOG detects stalled initialization
```

这与 #89 实机现象相符：`sensor_process` 启动后等待 sensor PD 的
servreg 状态确认；AP 侧没有 listener 时，固件侧等待 ack，最终 USER-PD
虚拟狗咬。

## 3. #90 修复

主线 `fastrpc.c` 明确不支持下游 static-PD notification 框架；在没有
用户态 fastrpc/sensor client 的 initramfs 阶段，直接移植整个 fastrpc 设备
不是最小修复。因此本轮新增一个仅保留 sensor PDR listener 的小驱动：

```text
drivers/soc/qcom/qcom-adsp-sensor-pd.c
```

行为：

1. probe 时读取 `adsp_variant` fuse 并打印；
2. 调 `pdr_handle_alloc()`；
3. 调：

```c
pdr_add_lookup(pdr, "tms/servreg", "msm/adsp/sensor_pd");
```

4. PDR core 收到 indication 后调用回调；
5. 回调返回后，PDR core 发送 servreg ACK。

这复刻的是下游 `qcom,fastrpc-adsp-sensors-pdr` 的最小必要行为，不引入
fastrpc 用户态依赖。

## 4. 新增 DT 与配置

DT 新增：

```text
adsp-sensor-pd {
    compatible = "qcom,adsp-sensor-pd";
    nvmem-cells = <&adsp_variant>;
    nvmem-cell-names = "adsp_variant";
    status = "okay";
};
```

同时按 live DTS 补 qfprom cell：

```text
adsp_variant@210 {
    reg = <0x213 0x1>;
    bits = <0x1 0x2>;
};
```

新增 binding：

```text
Documentation/devicetree/bindings/soc/qcom/qcom,adsp-sensor-pd.yaml
```

新增配置：

```text
CONFIG_QCOM_ADSP_SENSOR_PD=y
```

`scripts/build-boot.sh` 已把它加入强制检查，避免后续 olddefconfig 静默
回退。

## 5. 详细排查日志

### sensor PD driver

```text
adsp-sensor-pd: adsp_variant fuse value: 0xXX
adsp-sensor-pd: tracking tms/servreg -> msm/adsp/sensor_pd
adsp-sensor-pd: sensor PD msm/adsp/sensor_pd state up (0x1fffffff)
```

### PDR core

本轮在 `pdr_interface.c` 增加三层日志：

```text
PDR: Located <path> using <service>: instance=... service_data_valid=...
PDR: Listener registered for <path>, state: 0x...
PDR: Acknowledging <path> transaction <id>
```

这些日志可以区分：

- locator 没上线；
- domain list 中找不到 `msm/adsp/sensor_pd`;
- listener 注册失败；
- indication 收到但 ACK 发送异常；
- sensor PD 状态一直是 UNINIT/DOWN；
- sensor PD 已 UP 但 `sensor_process` 仍狗咬。

### QRTR nameserver

#89 有一条无细节的：

```text
failed while handling packet from 5:-2
```

本轮把 QRTR nameserver 失败日志扩展为包含控制命令名和返回值：

```text
failed while handling <hello/new-server/new-lookup/...> packet
from <node>:<port>: <ret>
```

便于确认它是否与 sensor PD/PDR 相关。

## 6. 本地验证

- `qcom-adsp-sensor-pd.o`、`pdr_interface.o`、`qrtr/ns.o` AArch64 编译
  通过；
- 新 C 文件与 binding checkpatch 均为 0 errors / 0 warnings；
- 自包含 DTB 编译通过，no-map 仍为 22；
- 反编译确认 `/adsp-sensor-pd` 与 `adsp-variant@210` 均存在；
- 完整 `scripts/build-boot.sh` 通过，manifest 全部断言为 True。

产物：

```text
boot_7.3.img SHA256:
64ea315029081d1933889658349f7ae6a881f74e7b21a2e1be754b66819d97ec

runtime DTB SHA256:
ae684037895387e843d1bbe81df47d9f26afe6b9efd1bf15e360392be6c6355d

Image SHA256:
ceae92fb3cae77e8738c0ed18d4bbf321b87142cbf8e8828f22d9df728206d5c
```

## 7. 实机判据

成功路径应依次出现：

```text
adsp-sensor-pd: adsp_variant fuse value: 0x...
adsp-sensor-pd: tracking tms/servreg -> msm/adsp/sensor_pd
PDR: Located msm/adsp/sensor_pd using tms/servreg: instance=74
PDR: Listener registered for msm/adsp/sensor_pd, state: 0x...
adsp-sensor-pd: sensor PD msm/adsp/sensor_pd state up (0x1fffffff)
PDR: Acknowledging msm/adsp/sensor_pd transaction ...
```

随后 60 秒内不应再出现：

```text
sensor_process ... dog_virtual_user
remoteproc0: crash detected in adsp
```

如果 sensor PD 已 UP 但仍狗咬，下一优先差异是：

1. live/downstream 的 `qcom,smp2p-adsp/sleepstate_see` 与下游
   `CONFIG_QCOM_SMP2P_SLEEPSTATE=y`;
2. 完整 fastrpc glink/compute-cb 拓扑；
3. ADSP firmware variant 选择（live adsp-loader 依据 `adsp_variant` 选择
   `adsp2`，本轮先只实装 fuse 读数与日志）。

## 8. #90 实机结果与 #91 FastRPC 修复

### 8.1 #90 结果

最新 `uart_early.log` 证明 #90 的 sensor PDR 依赖本身已经满足：

```text
qcom-adsp-sensor-pd: adsp_variant fuse value: 0x03
qcom-adsp-sensor-pd: tracking tms/servreg -> msm/adsp/sensor_pd
PDR: Located msm/adsp/sensor_pd using tms/servreg: instance=74
PDR: Listener registered for msm/adsp/sensor_pd, state: 0x7fffffff
PDR: Indication received from msm/adsp/sensor_pd, state: 0x1fffffff
qcom-adsp-sensor-pd: sensor PD msm/adsp/sensor_pd state up (0x1fffffff)
PDR: Acknowledging msm/adsp/sensor_pd transaction 1
```

但 41.482 秒仍然狗咬：

```text
EF:sensor_process:0x1:TMR_CLNT_1:0xfc:dog_virtual_user
USER-PD DOG detects stalled initialization
```

与 #89 的 `TMR_CLNT_1:0xff` 相比，#90 变为 `0xfc`。这说明新增 listener
确实改变了 sensor PD/初始化等待状态；不能直接把该值解释为 bitmask，但
现象与“初始化 countdown/RCInit 仍有依赖未满足”的模型一致。结论是：
**缺失的不是 sensor PD ACK，而是另一个 AP 侧启动依赖。**

### 8.2 SM8250 / SM7225 主线对照

#### SM8250

`sm8250.dtsi` 的传感器架构不是直接参照对象：

- ADSP 主要承载 audio/FastRPC；
- 传感器在独立 SLPI 上运行，`&slpi` 加载 `slpi.mbn`；
- SM8250 主线 FastRPC 节点位于 SLPI glink edge，`label = "sdsp"`，
  使用 `apps_smmu 0x541..0x543`；
- 本机 SM7250/lito 固件把 sensor process 集成在 ADSP image 内，架构更接
  近 SM7225/SM6350。

#### SM7225 / SM6350

`sm6350.dtsi` 的 ADSP glink edge 中有主线 FastRPC 子节点：

```dts
fastrpc {
    compatible = "qcom,fastrpc";
    qcom,glink-channels = "fastrpcglink-apps-dsp";
    label = "adsp";
    qcom,non-secure-domain;

    compute-cb@3 { iommus = <&apps_smmu 0x1003 0x0>; };
    compute-cb@4 { iommus = <&apps_smmu 0x1004 0x0>; };
    compute-cb@5 { iommus = <&apps_smmu 0x1005 0x0>; qcom,nsessions = <5>; };
};
```

SM7225 Fairphone FP4 仅启用 ADSP firmware，没有单独 sensor PD keepalive
driver。也就是说，最接近本机的主线模型是：

```text
ADSP glink edge
  ├ IPCRTR / QRTR
  ├ APR/audio PD
  └ fastrpcglink-apps-dsp + ADSP compute context banks
```

而不是只保留 PDR listener。

### 8.3 live/downstream 精确差异

live ADSP glink edge 声明：

```dts
qcom,msm_fastrpc_rpmsg {
    compatible = "qcom,msm-fastrpc-rpmsg";
    qcom,glink-channels = "fastrpcglink-apps-dsp";
    qcom,intents = <0x64 0x40>;
};
```

live 平台 FastRPC 节点中 ADSP 专用 context banks 为 cb10..cb12：

```text
cb10: adsprpc-smd, apps_smmu 0x1403
cb11: adsprpc-smd, apps_smmu 0x1404
cb12: adsprpc-smd, apps_smmu 0x1405, shared-cb = 5
```

#90 之前的主线 DT 完全没有该 ADSP FastRPC rpmsg 子节点，且
`CONFIG_QCOM_FASTRPC` 未启用。下游 `adsprpc` platform probe 除了注册
audio/sensor PDR listener 外，还会创建 FastRPC rpmsg transport 和这些
context banks。#90 只复刻了 PDR listener，没有复刻 transport 拓扑。

### 8.4 固件逆向证据

按 `adsp.mdt` program headers 将 `adsp.b00..b32` 重建成 ELF32 Hexagon
image 后检索，可见：

```text
fastrpcglink-apps-dsp
fastrpc_send_early_signal
sns_user_pd_init.c:init countdown %d
rcinit_lite_task.c:COMMON:RCInit Processing Done %c
msm/adsp/sensor_pd
sleepstate
sleepstate_see
Remote_proc_state
```

这支持两个判断：

1. sensor user PD 初始化存在 FastRPC/AP 侧 early-signal 或 transport 依赖；
2. sleepstate/remote processor state 是下一个独立候选，不应与 #91 混测。

### 8.5 下游 userspace 逆向：缺失的 `sscrpcd sensorspd`

继续检查 ROM vendor 分区（本地已解出的 EROFS `vendor.img`）发现：

```text
/etc/init/vendor.sensors.sscrpcd.rc:
service vendor.sensors /vendor/bin/sscrpcd sensorspd
    class early_hal
```

也就是说，下游 Android 在 early_hal 阶段就会启动一个 sensor FastRPC
client，而不是等普通 sensor HAL 再连接 ADSP。

对 vendor ARM64 ELF 做静态逆向：

1. `sscrpcd` `dlopen()` `libadsp_default_listener.so`；
2. 调用导出函数 `adsp_default_listener_start()`；
3. `libadsp_default_listener.so` 构造：

   ```text
   '":;./\createstaticpd: + sensorspd
   ```

   并传给 `remote_handle_open()`；

4. 它同时设置：

   ```text
   ADSP_LISTENER_MEM_CACHE_SIZE=1048576
   ```

5. 随后打开/调用 `adsp_default_listener`，并通过 `geteventfd` 长期 poll。

继续逆向 `libadsprpc.so` 可见，`'":;./\createstaticpd:` 是用户库识别的
特殊前缀；库名/PD 名真正传给 DSP 侧的是前缀后的 `sensorspd`，对应内核
FastRPC 的 create-static-process 语义。

这解释了 #90 为什么“sensor PD 已 UP 且 ACK 已发出”但
`sns_user_pd_init.c:init countdown` 仍未归零：PDR ACK 只是服务状态握手，
下游还有 early_hal userspace 创建/附着 sensor static PD 并保持 FastRPC fd
的启动契约。最小 initramfs 完全没有这个 client。

### 8.6 #91 修改

DT 在 ADSP `glink-edge` 内新增主线模型节点：

```dts
fastrpc {
    compatible = "qcom,fastrpc";
    qcom,glink-channels = "fastrpcglink-apps-dsp";
    label = "adsp";
    qcom,non-secure-domain;

    compute-cb@10 { reg = <10>; iommus = <&apps_smmu 0x1403 0x0>; };
    compute-cb@11 { reg = <11>; iommus = <&apps_smmu 0x1404 0x0>; };
    compute-cb@12 {
        reg = <12>;
        iommus = <&apps_smmu 0x1405 0x0>;
        qcom,nsessions = <5>;
    };
};
```

当时把 reg 误认为 live/downstream cb10..cb12 的节点编号；stream ID 则按
live 使用 `0x1403..0x1405`，没有照抄 SM6350 ADSP 的 `0x1003..0x1005`。
这个 reg 解释是 #91/#92 立即死机的直接错误，#93 已修正为 3/4/5。
主线 `qcom,fastrpc` binding 未声明 live 的
`qcom,intents`，因此 #91 先使用主线默认 intent 行为，与 SM7225 保持一致。

配置新增：

```text
CONFIG_QCOM_FASTRPC=y
```

并在 `scripts/build-boot.sh` 中加入强制检查。

除主线 transport 外，#91 还加入一个最小 userspace bootstrap，避免直接把
Android bionic/vendor listener 栈搬进 initramfs：

```text
scripts/fastrpc-sensor-attach.c
scripts/initramfs/fastrpc-sensor-attach
```

静态 AArch64 helper 的行为：

1. 最多等待 `/dev/fastrpc-adsp` 120 秒；
2. 调 `FASTRPC_IOCTL_INIT_CREATE_STATIC`：

   ```text
   name="sensorspd"（包含结尾 NUL）
   memlen=1MiB
   ```

   1MiB 来自下游 `ADSP_LISTENER_MEM_CACHE_SIZE=1048576`；
3. 用第二个 fd 调 `FASTRPC_IOCTL_INIT_ATTACH_SNS`；
4. 长期 `pause()`；即使第二个 attach ioctl 失败，也不关闭 create fd，
   避免 static PD 被立即拆除。

`scripts/make-initramfs.sh` 将 helper 打包为
`/bin/fastrpc-sensor-attach`；`/init` 在 ADSP SSR guard 后后台启动它。
这不是完整复刻 `libadsp_default_listener.so` 的 eventfd listener，只是
复刻下游 early_hal sensor PD create/attach/持 fd 契约；因此日志会明确
显示每个 ioctl 结果。

### 8.7 #91 排查日志

`drivers/misc/fastrpc.c` 新增一次性/限流日志：

```text
fastrpc ... channel probe: domain=adsp id=0 secure=0 vmids=0 poll=0
fastrpc ... compute bank sid=10 nsessions=1 dma_bits=32 total_sessions=1
fastrpc ... compute bank sid=11 nsessions=1 dma_bits=32 total_sessions=2
fastrpc ... compute bank sid=12 nsessions=5 dma_bits=32 total_sessions=7
fastrpc ... probe complete: sessions=7 users=0 node=fastrpc-adsp
fastrpc ... user open: secure=0 sid=10 domain=0
fastrpc ... create-static request: name=sensorspd namelen=10 memlen=1048576 sid=10
fastrpc ... create-static complete: name=sensorspd heap=... size=0x100000
fastrpc ... user open: secure=0 sid=11 domain=0
fastrpc ... attach complete: pd=2 sid=11
fastrpc-sensor-attach: /dev/fastrpc-adsp opened
fastrpc-sensor-attach: create-static sensorspd/1MiB ret=0 errno=0 (Success)
fastrpc-sensor-attach: attach-sensors ret=0 errno=0 (Success)
fastrpc-sensor-attach: sensor PD bootstrap active; keeping fds open
```

异常路径额外打印：

```text
DSP PD notification received, len=... addr=...
No context ID matches response: ctx=... len=... addr=...
channel removed
```

这些日志可区分：

- FastRPC DT/config 未生效（没有 channel probe）；
- glink 远端没有 announce `fastrpcglink-apps-dsp`（同样没有 probe）；
- context bank 缺失或 `of_platform_populate()` 失败（没有三条 compute bank
  日志或 sessions 不是 7）；
- 静态 PD notification 到达但初始化仍未完成（看到 DSP PD notification 后
  狗咬）；
- FastRPC 响应上下文异常（No context ID matches response）。
- create-static 或 attach-sensors 失败（helper 与内核各打印 errno/err）。

### 8.8 #91 本地验证

- `fastrpc.c` diff checkpatch：0 errors / 0 warnings；
- `fastrpc-sensor-attach.c` 以 NDK r28c `-Wall -Wextra -Werror -static`
  编译为 stripped AArch64 可执行文件（409KiB）；
- 自包含 DTB 编译通过，反编译确认 FastRPC 与 cb10..cb12 均存在；
- no-map reserved region 仍为 22；
- `.config` 确认 `CONFIG_QCOM_FASTRPC=y`；
- initramfs cpio 确认包含 `/bin/fastrpc-sensor-attach`；
- `strings Image` 确认新增 FastRPC 排查日志已编入内核；
- 完整 `scripts/build-boot.sh` 通过，manifest 全部断言为 True。

产物：

```text
boot_7.3.img SHA256:
d960b56fcf4b0245fa685141162fd540da06993658789b0322433074ea1ccb53

runtime DTB SHA256:
4b354c222cc52ef1d40e859f35223bcadaf74c1430ace3abce34006b39ddfd02

Image SHA256:
d279dc8b337fc298020b297a0d804fb4d4f31f4bad2654cdd3c5c90048826ee3

initramfs SHA256:
5bffd817f161f0353eb2ace2292a772dd51fb90eb9d41f17572b976239a4117a

fastrpc-sensor-attach SHA256:
1c6b87ab3dfee931ccfde00866a42ec5ab53766abf4d4dfdecfca146cb3d7f27
```

内核本地版本字符串仍显示 `#89`，这是既有 build counter 行为；本文档按项目
迭代号称本产物为 #91。

### 8.9 #91 实机判据

至少观察到 60 秒：

```sh
dmesg | grep -iE 'fastrpc|PDR:|sensor_process|remoteproc|dog_virtual'
cat /sys/kernel/debug/remoteproc/remoteproc*/state
```

成功：

```text
channel probe ... domain=adsp
compute bank sid=10/11/12
probe complete: sessions=7
create-static sensorspd/1MiB ret=0
attach-sensors ret=0
remoteproc0: running
```

且没有：

```text
sensor_process ... dog_virtual_user
remoteproc0: crash detected in adsp
```

若 FastRPC 完整 probe、create-static 与 attach-sensors 均成功后仍在 40 秒
狗咬，#92 应单独移植 live/downstream 的
`qcom,smp2p-adsp/sleepstate-out`、`sleepstate_see` 与下游
`smp2p_sleepstate.c`。该驱动 probe 时会把 sleepstate-out bit12 置为
AP awake；不要把 sleepstate 与 FastRPC 混在同一轮，否则无法归因。

## 9. #91 实机结果与 #92 保护性修复

### 9.1 #91 实机结果

最新 `uart_early.log` 证明主线 FastRPC 拓扑本身已经完全建立：

```text
qcom,fastrpc ... channel probe: domain=adsp id=0 secure=0 vmids=0 poll=0
qcom,fastrpc-cb ... sid=10 ... total_sessions=1
qcom,fastrpc-cb ... sid=11 ... total_sessions=2
qcom,fastrpc-cb ... sid=12 ... total_sessions=7
qcom,fastrpc ... probe complete: sessions=7 users=0 node=fastrpc-adsp
```

说明：

- `qcom,fastrpc` rpmsg driver 成功匹配；
- ADSP 已 announce `fastrpcglink-apps-dsp`；
- cb10/cb11/cb12 及 `0x1403..0x1405` stream ID 均可 probe；
- `of_platform_populate()` 正常；
- #91 的 DT/config 与主线 SM7225 模型方向正确。

但最后一条 durable 输出是：

```text
fastrpc-sensor-attach: /dev/fastrpc-adsp opened
```

随后没有：

```text
fastrpc ... create-static request: ...
fastrpc-sensor-attach: requesting CREATE_STATIC ...
fastrpc ... create-static complete: ...
```

系统在用户可交互 shell 稳定前死机。由于 helper 在 open 后的下一个动作就是
`FASTRPC_IOCTL_INIT_CREATE_STATIC`，#91 的直接嫌疑是 CREATE_STATIC 的
remote invoke/1MiB remote heap 路径，而不是 FastRPC glink 通道或 context
banks 本身。

### 9.2 #92 修复：默认只做 ATTACH_SNS

#92 不再在 `/init` 阶段自动执行 CREATE_STATIC。helper 改为两种模式：

```sh
# 默认、initramfs 自动执行：
/bin/fastrpc-sensor-attach

# 仅在 UART shell 稳定后手动执行：
/bin/fastrpc-sensor-attach --create-static
```

`/init` 先启动 UART shell，5 秒后才后台执行 helper。默认模式只调用：

```text
FASTRPC_IOCTL_INIT_ATTACH_SNS
```

该路径不分配 1MiB remote heap，也不请求 DSP 创建新的 static process，只
附着既有 sensors PD。它仍有内核端 10 秒 invoke timeout，并新增：

```text
fastrpc ... attach request: pd=2 sid=...
fastrpc ... attach complete: pd=2 sid=...       # 或 attach failed
fastrpc-sensor-attach: requesting ATTACH_SNS
fastrpc-sensor-attach: attach-sensors ret=... errno=...
```

无论 attach 成功或失败，helper 都保持 fd 打开并 `pause()`，避免立即释放
可能已经建立的 sensor PD client。

CREATE_STATIC 仍保留在 `--create-static` 手动模式中，用于在 shell 可用后
单独验证 #91 的危险路径；不要在下一次自动回归中混入。

### 9.3 #92 本地验证与产物

- `fastrpc.c` diff checkpatch：0 errors / 0 warnings；
- helper 以 NDK r28c `-Wall -Wextra -Werror -static` 编译；
- initramfs cpio 仍包含 `/bin/fastrpc-sensor-attach`；
- runtime DTB no-map 仍为 22；
- 完整 `scripts/build-boot.sh` 通过，manifest 全部断言为 True。

```text
boot_7.3.img SHA256:
99ec98ccccaac87fec81a68271fe51b7c6e210fe9c508488124071ef1850eaca

runtime DTB SHA256:
4b354c222cc52ef1d40e859f35223bcadaf74c1430ace3abce34006b39ddfd02

Image SHA256:
2637d2a65a21ae1fe152896b82ef4734b41ac9d18dd30921b0ed42a67aecb21c

initramfs SHA256:
6111b8b285c042bdc659a6ae9abf6f2eb39c7fed12db4878982fccb332557043

fastrpc-sensor-attach SHA256:
048836acf05771783091ca7c617568eaa919b782a1dc9e698675ab1d03486bc2
```

### 9.4 #92 实机判据

首要目标恢复稳定 UART shell：

```text
fastrpc-sensor-attach: /dev/fastrpc-adsp opened (mode=attach-sns)
fastrpc ... attach request: pd=2 sid=...
fastrpc ... attach complete/failed
```

然后观察至少 60 秒：

```sh
dmesg | grep -iE 'fastrpc|PDR:|sensor_process|remoteproc|dog_virtual'
cat /sys/kernel/debug/remoteproc/remoteproc*/state
```

若 ATTACH_SNS 成功且 60 秒无狗咬，说明 #91 找到的缺失依赖方向正确，
CREATE_STATIC 不是必要条件。若 ATTACH_SNS 返回错误但 shell 保持稳定，
记录 errno 后再考虑手动 CREATE_STATIC。若 ATTACH_SNS 仍触发系统死机，
则说明任何 sensor PD FastRPC invoke 都会命中更底层固件/内存映射问题，
下一轮应先查 remote heap/context bank 与 ADSP 固件 ABI，而不是 sleepstate。

## 10. #92 实机结果与 #93 SID 修正

### 10.1 #92 实机结果

#92 成功证明 UART shell 可以先出现；约 5 秒后 helper 启动时系统死机。
最后一条输出为：

```text
fastrpc-sensor-attach: /dev/fastrpc-adsp opened (mode=attach-sns
```

该行最后的 `)` 与换行都没有发出，随后没有：

```text
fastrpc-sensor-attach: requesting ATTACH_SNS
fastrpc ... attach request: pd=2 sid=...
```

这次连不分配 remote heap 的 ATTACH_SNS 也触发死机，说明问题在更底层的
FastRPC invoke 地址/ context bank 编码，而不是 CREATE_STATIC 独有行为。

### 10.2 根因复核：mainline reg 不是 live cb 节点序号

#91/#92 犯了一个关键解释错误：

```text
live/downstream 节点名: cb10/cb11/cb12
stream ID:             0x1403/0x1404/0x1405
```

当时把节点名 cb10..cb12 直接映射成主线：

```dts
compute-cb@10 { reg = <10>; iommus = <&apps_smmu 0x1403 ...>; };
```

但主线 `reg` 是 driver 用来合成 consolidated IOVA 的 FastRPC SID，不是
下游平台容器里的 context-bank 节点序号。`drivers/misc/fastrpc.c` 中：

```c
static inline u64 fastrpc_sid_offset(struct fastrpc_channel_ctx *cctx,
                                     struct fastrpc_session_ctx *sctx)
{
        return (u64)sctx->sid << cctx->soc_data->sid_pos;
}
```

默认 `soc_data.sid_pos = 32`，因此 #92 实际把第一个 invoke buffer 地址变成：

```text
SID 10 << 32 = 0x0000000a00000000
```

而 stream ID `0x1403` 对应的 DSP 侧 SID 应为 3：

```text
SID 3 << 32 = 0x0000000300000000
```

ADSP 按 SID 3 解析 context bank 时收到 SID 10 的合成地址，可能在能否打印
timeout/error 之前就触发低级内存/保护故障，这与两次“helper 输出中途死机、
没有 invoke 结果”的现象一致。

主线已有平台也证明了这个映射关系：

```text
SM6350 CDSP: compute-cb@3 -> apps_smmu 0x1403
SM6350 CDSP: compute-cb@4 -> apps_smmu 0x1404
SM6350 CDSP: compute-cb@5 -> apps_smmu 0x1405

SDM845 CDSP: compute-cb@3 -> apps_smmu 0x1403
SDM845 CDSP: compute-cb@4 -> apps_smmu 0x1404
SDM845 CDSP: compute-cb@5 -> apps_smmu 0x1405
```

因此 live 的 cb10..cb12 只说明下游把这些 ADSP bank 放在平台节点的第
10..12 个子节点；转换到主线模型时应使用：

```text
cb10 -> compute-cb@3, reg=3, stream ID 0x1403
cb11 -> compute-cb@4, reg=4, stream ID 0x1404
cb12 -> compute-cb@5, reg=5, stream ID 0x1405, nsessions=5
```

### 10.3 #93 修改

runtime DT 已修正为：

```dts
compute-cb@3 {
    reg = <3>;
    iommus = <&apps_smmu 0x1403 0x0>;
};

compute-cb@4 {
    reg = <4>;
    iommus = <&apps_smmu 0x1404 0x0>;
};

compute-cb@5 {
    reg = <5>;
    iommus = <&apps_smmu 0x1405 0x0>;
    qcom,nsessions = <5>;
};
```

helper 仍保持 #92 的安全策略：

1. shell 先启动；
2. 5 秒后自动 ATTACH_SNS；
3. CREATE_STATIC 只保留手动模式；
4. ioctl 结果和内核 invoke 状态均有日志。

### 10.4 #93 本地验证与产物

- runtime DTB 反编译确认 `compute-cb@3/4/5` 与 `reg=<3/4/5>`；
- stream ID 仍严格来自 live：`0x1403/0x1404/0x1405`；
- no-map reserved region 仍为 22；
- manifest 全部断言通过。

```text
boot_7.3.img SHA256:
92d87e5e8809e4e3d8e507c404e87ae58767ec1e39cf8e823355ae313c006cfb

runtime DTB SHA256:
f0e22f2c149f0f9a0fee779da6136800b8f77c045a12c863135c78b505d2edf1

Image SHA256:
2637d2a65a21ae1fe152896b82ef4734b41ac9d18dd30921b0ed42a67aecb21c

initramfs SHA256:
6111b8b285c042bdc659a6ae9abf6f2eb39c7fed12db4878982fccb332557043
```

### 10.5 #93 实机判据

这次 compute bank 日志必须从 #92 的 sid 10/11/12 变成：

```text
compute bank sid=3 nsessions=1 ... total_sessions=1
compute bank sid=4 nsessions=1 ... total_sessions=2
compute bank sid=5 nsessions=5 ... total_sessions=7
```

5 秒后应看到：

```text
fastrpc-sensor-attach: /dev/fastrpc-adsp opened (mode=attach-sns)
fastrpc-sensor-attach: requesting ATTACH_SNS
fastrpc ... attach request: pd=2 sid=3
fastrpc ... attach complete: pd=2 sid=3
```

成功标准仍是 60 秒内无：

```text
sensor_process ... dog_virtual_user
remoteproc0: crash detected in adsp
```

如果 SID 3 的 ATTACH_SNS 不再立即死机但返回错误，则说明底层地址编码已
修复，剩余问题是 sensor PD ABI/listener 语义；如果仍立即死机，下一轮才
继续查 IOMMU fault mask、secure-domain 标志和 firmware variant。

## 11. #93 实机结果与 #94 SMP2P sleepstate

### 11.1 #93 实机结果

#93 证明 FastRPC 地址编码修复成功：

```text
compute bank sid=3 nsessions=1 ... total_sessions=1
compute bank sid=4 nsessions=1 ... total_sessions=2
compute bank sid=5 nsessions=5 ... total_sessions=7

qcom,fastrpc ... user open: secure=0 sid=3 domain=0
qcom,fastrpc-cb ... attach request: pd=2 sid=3
qcom,fastrpc-cb ... attach complete: pd=2 sid=3
```

shell 保持稳定，7.1 秒 ATTACH_SNS 成功返回，没有再出现 #91/#92 的立即
死机。因此：

- FastRPC glink 通道已通；
- cb3/stream `0x1403` 的 consolidated IOVA 编码正确；
- sensor PD 可以接受 AP 侧 ATTACH_SNS；
- 立即死机根因确认为此前 `reg=10/11/12` 的 SID 编码错误。

但 41.475 秒仍出现：

```text
PDM: service 'sensor_process' crash:
EF:sensor_process:0x1:TMR_CLNT_1:0xfc:dog_virtual_user

USER-PD DOG detects stalled initialization
remoteproc0: crash detected in adsp
```

剩余问题回到 #90 后列出的下一个 live/downstream 差异：SMP2P sleepstate。

### 11.2 live/downstream sleepstate 事实

live ADSP SMP2P edge 除 master/slave kernel 外还声明：

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

并有独立 client：

```dts
qcom,smp2p_sleepstate {
    compatible = "qcom,smp2p-sleepstate";
    qcom,smem-states = <&sleepstate_smp2p_out 0>;
    interrupt-parent = <&sleepstate_smp2p_in>;
    interrupts = <0 0>;
    interrupt-names = "smp2p-sleepstate-in";
};
```

下游 `smp2p_sleepstate.c` 在 probe 时执行：

```c
qcom_smem_state_update_bits(state, BIT(12), BIT(12));
```

也就是立即把 AP awake bit 置 1；suspend 前清零，resume 后恢复。当前
mainline DT 没有这两个 entry，也没有 client，ADSP sensor 固件可能一直看
到 AP 处于非 awake 状态。

ADSP 固件字符串同样包含：

```text
sleepstate
sleepstate_see
Remote_proc_state
```

这与 `sns_client_manager.c` / remote processor state 语义吻合。

### 11.3 #94 修改

新增主线化 driver：

```text
drivers/soc/qcom/qcom-smp2p-sleepstate.c
```

行为：

1. `devm_qcom_smem_state_get()` 获取 `sleepstate`；
2. 读取 DT smem-state 引用参数，但按下游行为固定更新 32-bit entry 的
   bit 12；
3. 获取并注册 `sleepstate_see` rising IRQ；
4. probe 成功后立即置 awake bit；
5. 移除时清零；
6. 当前内核未启用 suspend，因此不移植 PM notifier，保持 boot 期间常 awake。

> 注：live DT 的 `<sleepstate-out 0>` 是 legacy state 引用参数；下游
> `smp2p_sleepstate.c` 不用它选择 awake bit，而是固定执行
> `BIT(PROC_AWAKE_ID)`，其中 `PROC_AWAKE_ID=12`。#94 保留该行为，同时
> 打印 DT 参数和实际更新的 bit 12，便于实机确认。

DT 新增：

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

配置新增：

```text
CONFIG_QCOM_SMP2P_SLEEPSTATE=y
```

`scripts/build-boot.sh` 已加入强制检查。主线 `qcom,smp2p.yaml` binding
增加 `sleepstate` / `sleepstate-see` entry；新增
`qcom,smp2p-sleepstate.yaml` binding。

### 11.4 #94 排查日志

SMP2P core 新增 entry 生命周期日志：

```text
registered outbound SMP2P entry 'master-kernel'
registered inbound SMP2P entry 'slave-kernel'
registered outbound SMP2P entry 'sleepstate'
registered inbound SMP2P entry 'sleepstate_see'

matched early inbound SMP2P entry 'sleepstate_see', initial value 0x...
```

sleepstate client 日志：

```text
qcom-smp2p-sleepstate: ADSP sleepstate ready: DT state bit 0, apps-awake bit 12 asserted
qcom-smp2p-sleepstate: sleepstate_see edge irq=...
```

这些日志可区分：

- sleepstate entry 没有注册：DT/config 未生效；
- outbound entry 已注册但 client defer/probe 失败：smem-state 引用异常；
- inbound `sleepstate_see` 没有被远端匹配：ADSP 固件未创建 entry；
- client 已把 awake bit 置 1 但 sensor 仍狗咬：sleepstate 不是根因；
- 收到 `sleepstate_see` edge：远端状态变化已到达 AP。

### 11.5 #94 本地验证与产物

- sleepstate driver diff checkpatch：0 errors / 0 warnings；
- runtime DTB 反编译确认 sleepstate/sleepstate_see/client 均存在；
- no-map reserved region 仍为 22；
- `CONFIG_QCOM_SMP2P_SLEEPSTATE=y`；
- `strings Image` 确认 sleepstate 日志编入；
- 完整 `scripts/build-boot.sh` 通过，manifest 全部断言为 True。

```text
boot_7.3.img SHA256:
e954c55bd798622544aff449c1f997d4f59a30801eb3c54c86ee52f54620658f

runtime DTB SHA256:
dbe7a698e722a25055fe7aa40cd3d6d00bde0ab7d5373afb1639660d30ad0863

Image SHA256:
9076ef1de14786bc94c167c79e1418f4e3e1c6aeb25ca2cddf533dd1798a92a3

initramfs SHA256:
6111b8b285c042bdc659a6ae9abf6f2eb39c7fed12db4878982fccb332557043
```

### 11.6 #94 实机判据

必须先看到：

```text
registered outbound SMP2P entry 'sleepstate'
registered inbound SMP2P entry 'sleepstate_see'
qcom-smp2p-sleepstate: ADSP sleepstate ready: apps-awake bit ... asserted
```

随后保留 #93 的：

```text
fastrpc ... attach complete: pd=2 sid=3
```

成功标准：60 秒内没有：

```text
sensor_process ... dog_virtual_user
remoteproc0: crash detected in adsp
```

如果 sleepstate client 成功但狗咬不变，下一优先差异是完整复刻
`libadsp_default_listener.so` 的 remote invoke/eventfd 行为，或继续逆向
`sns_user_pd_init.c:init countdown` 等待的具体 signal。
