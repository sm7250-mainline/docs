# 2026-10-16：A620 完整频率、ACD、带宽与热管理支持

状态（2026-10-19 #82 实机修订）：已修正全 bin 频表，并补齐独立 GPU 总线
调频、LM/isense、四 ring preemption 与安全门控 IFPC。#82 实机已恢复 GMU
原生 625MHz 启动、IFPC、显示 fb0 和触摸固件下载。实测本机 speed-bin 为
132，频表 `150/275/400/500/625MHz` 正确；真实 3D 负载下的 DCVS/总线/LM
边界仍待验证（本 bin 不启用 LM）。

## 1. 本轮纠正的核心问题

此前为了让 GPU 从 `gfx3d` 根时钟启动失败中恢复，`a6xx_gpu.c` 会无条件把
RCG 改挂到 XO，因而 GPU 实际长期运行在 19.2MHz。更早的一次修复又把
750MHz OPP 的 `opp-supported-hw` 从 `0x03` 扩成 `0x07`，让本机读到的
speed-bin 115 也选择 750MHz。

这与下游 `lito-gpu.dtsi` 和用户提供的 `live_dts.dts` 冲突：speed-bin
115 的最高档是 540MHz，不是 750MHz。原先 GMU 把 PLL0 编程为约 750MHz
后无法输出，和错误放宽 bin 上限能够互相解释。

## 2. speed-bin 与 OPP

主线 catalog 现在支持 live DTS 中的五种 fuse，并将它们映射到独立的
`opp-supported-hw` bit：

| fuse | supported-hw | 可用频点 |
|---|---:|---|
| 0 | bit 0 | 625/500/400/275/150MHz |
| 132 | bit 1 | 625/500/400/275/150MHz |
| 115 | bit 2 | 540/400/275/150MHz |
| 158 | bit 3 | 750/670/625/500/400/275MHz |
| 165 | bit 4 | 625/500/400/275MHz |

每个 OPP 同步带入 live DTS/下游的：

- `opp-level`：`gfx.lvl` RPMh 电压级；
- `opp-peak-kBps`：原 KGSL bus level 对应的 DDR 峰值带宽；
- `qcom,opp-acd-level`：A620 各频点的 ACD 标定值。

625/500/400MHz 在旧、新 bin 上的 ACD 参数不同，因此使用相同频率、互斥
`opp-supported-hw` 的双 OPP 节点；OPP core 会在加入表之前按 speed-bin
过滤，不会留下同频冲突。

GPU 节点增加 `gfx-mem` interconnect，使 OPP 带宽属性可被解析。MSM GPU
驱动把这些值转为 SH0/MC0/ACV BCM 命令并随 HFI 交给 GMU。

2026-10-18 review 发现原实现只传每个 OPP 的下游 `bus-freq`，没有覆盖
`bus-min..bus-max`。现已加入 live DTS 的完整 13 档 grp3d 表，并在每个 OPP
记录 min/default/max。A620 GBIF 四个读写 beat counter 和两个 DDR wait
counter 按下游 `gpubw_mon` 的 TARGET=80、CAP=75%、WAIT=10% 和 5% hysteresis
生成 fast/slow hint；GMU HFI 的 `freq` index 保持不变，只独立改变 `bw`
index。因此 fuse 115 的 540MHz 档可在 bus 7/8/9/10/11/12 间变化，而不是
永久锁在默认 bus 8。

Picasso 未提供固定 `qcom,bus-accesses`，因此 normalized high watermark 从
350 起步；观察到更高的 `ram_time/total_time` 时抬高水位并立即发出 fast
hint，后续 up/down cutoff 随新水位重算，与下游 floating 模式一致。

DDR wait 固定 counter 0/1 分别选择通道 0/1（第二个 selector 为 `0x100`）。
总线停止、GPU 升频或存在负修正量时清零 `bus_modifier` 并回到新 OPP 默认值；
GPU 降频时允许短暂保留正向余量，复现下游的切档语义。

## 3. ACD 与 AOSS

下游 A620 明确带 `ADRENO_ACD`，并在启动时：

1. 把逐频点 ACD 表发给 GMU；
2. 启用 GMU ACD feature；
3. 通过 AOSS mailbox 通知 `{class: gpu, res: acd, ...}`。

主线现有驱动已经具备这些机制，但原 Picasso DT 没有 ACD 属性和
`qcom,qmp`，配置里也没有 mailbox/AOSS 驱动，因此该路径从未运行。本轮：

- GPU OPP 表改为 `operating-points-v2-adreno` 并加入 ACD 值；
- GMU 增加指向 AOSS QMP 的 `qcom,qmp`；
- 打开 `MAILBOX`、`QCOM_IPCC`、`QCOM_AOSS_QMP`；
- A620 使用下游顺序：先下发 ACD 表，再启用 ACD feature。

## 4. gfx3d 启动与 DCVS

保留已由实机证明必要的 `ROOT_EN → branch 0/1 relatch → UPDATE` 序列，
但不再先把 RCG 改到 XO：

1. 优先在 GMU 根据当前 speed-bin OPP 配置的 PLL0 源上执行补刀；
2. 成功后保持该源，后续 devfreq 通过 HFI 进行正常 DCVS；
3. 只有根时钟仍未启动时才回退到已验证的 XO 路径，并打印明确 warning。

日志明确打印的是软件/HFI 的目标频率，不再冒充硬件实测频率：

```text
A620 gfx3d root enabled, target 540000000 Hz (cfg=...)
```

或：

```text
gfx3d source ... failed, falling back to XO
gfx3d is running from the 19.2 MHz XO fallback
```

`target 540000000` 只能证明目标 OPP 和根时钟状态，不证明 PLL0 实际输出已被
测量。最终判据仍是 `cur_freq/trans_stat`、寄存器状态以及真实 3D 吞吐。

## 5. 其它下游对齐

- `gmu_cgc_mode` 从 `0x00020200` 修正为 A620/A650 下游值
  `0x00020202`；
- A620 HWCG 同步设置 `GMU_WFI_CONFIG=2`；
- 设置 `PC_DBG_ECO_CNTL` bit9，匹配下游 `disable_tseskip`；
- hang detect 周期改为 A620 下游值 `0x3ffff`；
- GPUCC PLL1 补 `cal_l=0x44`，GMU RCG 使用主线 shared/safe-config ops；
- 补齐 live GPUCC 声明对应的六个 reset 映射；
- 打开 TSENS/devfreq cooling，并把两个 GPUSS 传感器的 95°C passive trip
  接到 GPU cooling device。
- 按 live DTS 增加 `gaming_bin`、`isense_slope` nvmem；fuse 0/158（仅这两个
  bin 有 `throttle-pwrlevel`）按下游顺序启用 HFI LM 并发送 LM slope；同步
  配置 XOCLK1/2/3 throttle counter，devfreq busy time 会扣除节流周期。
- A620 默认四 ring preemption 已启用；A6xx IFPC/preemption 的静态恢复表、
  target-specific A620/A650 表和动态 perf-counter selector 恢复均已移植，
  CP_ME_INIT 携带 spinlock register-list 地址。
- IFPC 采用安全门控：首次启动先保持 active idle。只有 GMU 原生点亮 gfx3d
  根时钟时才启用；只要需要 host relatch 或 XO fallback，就保持/切回 active，
  避免 IFPC exit 再次撞上根时钟失活。

## 6. 本地验证与实机判据

使用原 `.o.cmd` 的 AArch64 编译参数、NDK r29 Clang 21 重放以下对象，均
零错误、零警告：

- `a6xx_catalog.o`
- `a6xx_gpu.o`
- `a6xx_gmu.o`
- `a6xx_hfi.o`
- `gpucc-sm7250.o`
- `qcom_aoss.o`（以新配置宏重放）
- `msm_gpu_devfreq.o`

自包含 DTS 经 dtc 编译通过。反编译 DTB 后逐项核对，五个 fuse 得到的频表
分别与上表一致；13 档 BW 表、per-OPP min/max、三个 GPU nvmem、`gfx-mem`、
AOSS QMP、ACD 和 cooling phandle 均存在。修订后 DTB SHA256 为
`6d265cd2bdb50fb1e39e1f000843b79153ef2310198f42c3f2ac59fa3e846f49`。
现有自包含 DTS 的历史数值 phandle 警告仍在，没有新增结构错误。

完整构建仍需回到 GNU make 4+ 的 Linux 环境执行：

```sh
bash scripts/build-boot.sh
```

不能使用 `--skip-kernel`。实机至少验证：

```sh
dmesg | grep -iE "A620 gfx3d|falling back|bandwidth|LM|IFPC|preempt|ACD|GMU|adreno"
cat /sys/class/devfreq/3d00000.gpu/available_frequencies
cat /sys/class/devfreq/3d00000.gpu/cur_freq
cat /sys/class/devfreq/3d00000.gpu/trans_stat
```

#82 实测修正：本机 fuse/speed-bin 是 **132**，不是此前预判的 115。预期频表
为 `150/275/400/500/625MHz`，启动目标 625MHz，空闲应降到 150MHz，且日志
不应出现 XO fallback。该 bin 按 live DTS 不启用 LM，但会启用独立 bus
调节和四 ring preemption；GMU 原生时钟启动成功后已启用 IFPC。还需用真实
3D 负载确认满载升到 625MHz、bus 档位变化、GPU IRQ、抢占、多次 IFPC 进出、
SMMU fault recovery 和热限频。
