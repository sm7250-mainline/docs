# 2026-10-18：A620 完整支持 review 修复

状态（2026-10-19 00:35）：review 指出的源码问题均已修复。#81 镜像实机
暴露 ACD/AOSS 配置缺口，导致 GPU、显示与触摸固件链连锁失败；#82 打开
mailbox/IPCC/AOSS QMP 并补上触摸 ESD workqueue 的 `WQ_UNBOUND` 后，
GPU/显示/触摸启动链实机恢复。实测本机 GPU speed-bin 为 132，频表
`150/275/400/500/625MHz` 正确；真实 3D 负载下的 DCVS/带宽/IFPC 仍待验证。
#83 另补 `CONFIG_DEVFREQ_THERMAL=y` 以消除 #82 中最后的 GPU cooling 注册
错误，已构建待实机回归。

## 1. speed-bin 与 ACD

`live_dts.dts` 和下游 `lito-v2.dtsi` 均表明 speed-bin 158/165 不包含
150MHz。150MHz 的 `opp-supported-hw` 已从 `0x1f` 改为 `0x07`：

| fuse | 可用频点（MHz） |
|---:|---|
| 0 | 625/500/400/275/150 |
| 132 | 625/500/400/275/150 |
| 115 | 540/400/275/150 |
| 158 | 750/670/625/500/400/275 |
| 165 | 625/500/400/275 |

驱动对 A620 增加六档 ACD 上限检查，防止 DT 再次向 GMU 2.0 固件发送未标定
的第七档。

## 2. 独立 GPU 总线调频

GPU 节点现在保存 live DTS 完整的 13 档 grp3d 表（0..12）。每个 OPP 除
默认 `opp-peak-kBps` 外，还记录 `qcom,opp-bw-min-kBps` 与
`qcom,opp-bw-max-kBps`。

驱动初始化四个 GBIF read/write beat counter 和两个 DDR wait counter，并按
下游 `gpubw_mon` 的阈值计算 fast/slow hint：

- TARGET=80；
- GPU CAP=75%；
- wait threshold=10%；
- hysteresis=5%；
- normalized high watermark 初值为 350；Picasso 未固定 `qcom,bus-accesses`，
  因此按下游逻辑动态抬高水位并触发 fast hint；
- 带宽只能在当前 OPP 的 min..max 内移动。

两个固定 DDR wait counter 的 selector 与下游一致：counter 0 选择通道 0，
counter 1 通过 `GBIF_PERF_PWR_CNT_SEL=0x100` 选择通道 1，避免重复统计通道 0。
总线停止时清零 `bus_modifier`；GPU 升频或当前修正量为负时也先回到新 OPP
默认带宽，GPU 降频时则允许暂时保留正向带宽余量，与下游
`kgsl_pwrctrl_pwrlevel_change()` 的状态转换一致。

调节时 HFI 的 GPU perf index 不变，只改变 bandwidth index。IFPC 期间读取
GX 计数器前使用 PERFCOUNTER OOB 唤醒，并处理计数器因 GX collapse 清零的
情况。

## 3. LM/isense

按 live DTS 补入 `gaming_bin@212` 和 `gpu_lm_efuse@45c8`。驱动读取
`gaming_bin`、`isense_slope`；仅 fuse 0/158 带下游 `throttle-pwrlevel=0`，
因此仅这两个 bin 启用 LM：

1. HFI feature 7，throttle mask 指向最高 perf level；
2. HFI SET_VALUE(115) 发送 isense slope；
3. XOCLK1/2/3 选择 0x10/0x16/0x0f；
4. devfreq busy counter 按下游公式扣除 LM throttle cycles。

fuse 115 不启用 LM，这是 live DTS 的明确行为，不是缺失。

## 4. preemption 与 IFPC

A620 catalog 已启用四 ring preemption 和 IFPC capability。移植了下游 A6xx
IFPC-only、preemption 共用、A620/A650 target-specific power-up register
list，并补齐动态 perf-counter selector 的 pair-list 恢复。A620 CP_ME_INIT
使用下游 11-dword ordinal 布局和 spinlock register-list 地址。

IFPC 不盲目开启：首次启动保持 active idle；若 GPU_SET 后根时钟已由 GMU
原生点亮，完成 CP/register-list 初始化后才启用 IFPC。若启动需要 host
relatch 或 XO fallback，则保持/切回 active，防止 IFPC exit 再次卡在失活
的 gfx3d 根时钟。

## 5. 日志与验证

启动日志改为 `target N Hz`，明确这是 HFI/软件目标，不是硬件测频结果。
实机必须同时检查：

```sh
dmesg | grep -iE "A620 gfx3d|bandwidth|LM|IFPC|preempt|falling back"
cat /sys/class/devfreq/3d00000.gpu/available_frequencies
cat /sys/class/devfreq/3d00000.gpu/cur_freq
cat /sys/class/devfreq/3d00000.gpu/trans_stat
```

还需用真实 3D 负载验证吞吐、四 ring 抢占、bus 7..12 变化以及多次 IFPC
进入/退出。DTB SHA256：
`6d265cd2bdb50fb1e39e1f000843b79153ef2310198f42c3f2ac59fa3e846f49`。

## 6. 当前工作树修改清单与检查结果

| 文件 | 同步内容 |
|---|---|
| `sm7250-xiaomi-picasso.dts` | 五个 speed-bin OPP、13 档 BW、逐 OPP min/default/max、三个 GPU nvmem、`gfx-mem`、AOSS QMP 与 cooling |
| `gpu.yaml` / `opp-v2-qcom-adreno.yaml` | A620 三段 reg/三个 nvmem、完整 BW 表、逐 OPP BW 范围 binding |
| `a6xx_catalog.c` | A620 IFPC/preemption capability 和 A6xx/A620 恢复表 |
| `a6xx_gmu.c` / `a6xx_gmu.h` | 独立 bus governor、floating watermark、bus modifier 生命周期、LM、六档 ACD 保护、运行时 IFPC |
| `a6xx_hfi.c` / `a6xx_hfi.h` | LM feature/SET_VALUE、A620 ACD 顺序、IFPC feature control |
| `a6xx_gpu.c` | 双通道 GBIF/DDR wait counter、LM busy 修正、pair register-list、CP_ME_INIT、时钟/IFPC 门控与目标频率日志 |
| `msm_gpu.h` / `msm_gpu_devfreq.c` | devfreq 到 GPU bus governor 的采样回调 |

最终本地检查：

- NDK r29 Clang 21 重放 5 个直接受影响对象，全部为 AArch64 ELF，零错误、
  零警告；
- DTB 编译成功，`fdtget` 反查确认 13 档 BW、三个 nvmem 名称、逐 OPP
  min/default/max，以及 150MHz mask=`0x07`；
- 目标文件 `git diff --check` 通过；GPU C/header 与 binding 分开运行
  `checkpatch` 无告警；
- 合并检查为 0 error；一个 binding 拆分提示和十个 DTS/UFS 历史格式 warning
  不属于本轮 GPU 行；
- 当时 macOS 环境只能完成单对象检查；完整构建与 #81 实机边界见 §7。

## 7. #81 实机回归与 #82 修复（2026-10-19）

`uart_early.log` 对应内核 #81。UFS 与 `/init` 正常，但 GPU 在
`Dynamic GPU bandwidth enabled with 13 GMU levels` 后报：

```text
a6xx_gmu_acd_probe: Unable to send ACD state to AOSS
msm_dpu: failed to load adreno gpu
msm_dpu: failed to bind 3d00000.gpu: -22
panel-g7a: Failed to attach to DSI host
```

错误回滚路径还触发 `Unable to drop a managed device link reference`
WARNING。显示 aggregate 未成立，后续 NT36672C 固件下载读到 `0xFF` 并触发
F2C ESD 恢复失败，应先按显示/GPU 连锁问题处理，不能把该现象单独定性为
触摸 SPI 回归。

根因是文档虽写了打开 mailbox/IPCC/AOSS QMP，但 #81 的 `.config` 实际为
`# CONFIG_MAILBOX is not set`，因此 `QCOM_AOSS_QMP` 未编入，`qmp_get()`
使用 stub 返回 `-ENODEV`；A620 ACD 需要该 handle，最终返回 `-EINVAL`。

已修复：

- `CONFIG_MAILBOX=y`
- `CONFIG_QCOM_IPCC=y`
- `CONFIG_QCOM_AOSS_QMP=y`
- `nvt_esd_check_wq` 增加 `WQ_UNBOUND`，消除 7.3 workqueue WARN

完整 `scripts/build-boot.sh` 通过。#82 `boot_7.3.img` SHA256：
`12119f90541e9bdd0c3fdb8584100e20864721ef01a4dce14cbe5dab9b495b40`。

## 8. #82 实机结果（2026-10-19）

启动日志显示：

- `Dynamic GPU bandwidth enabled with 13 GMU levels`;
- `A620 gfx3d root enabled by GMU, target 625000000 Hz`;
- `A620 IFPC enabled after native GMU clock start`;
- `Initialized msm 1.13.0`;
- `fb0: msmdrmfb frame buffer device`;
- NT36672C 固件 174ms 下载成功，`fw_ver=20`、`x_num=18`、`y_num=36`；
- `nvt_esd_check_wq` 的 workqueue WARN 已消失。

用户提供的 devfreq 状态：

```text
available_frequencies:
150000000 275000000 400000000 500000000 625000000

trans_stat:
625000000 -> 150000000，共 1 次转换
```

这不是 540MHz 表缺失的回归，而是此前文档对本机 fuse 的预判有误。反推依据：

1. 只有 fuse 0/132 的 live DTS 频表包含 `625/500/400/275/150`；
2. fuse 0 会启用 LM 并打印 `A620 LM enabled`，本机没有；
3. 因此本机实际是 **fuse/speed-bin 132**，不是 115。

live DTS 中 speed-bin 132（`0x84`）正是上述五档；speed-bin 115（`0x73`）
才是 `540/400/275/150`。当前 625→150 的转换符合启动后进入空闲的预期。
尚未运行真实 3D 负载，不能用 `trans_stat` 证明满载升频、bus 7..12 动态
调节或多次 IFPC 进出。

#82 中仍有一处可确定错误：

```text
Couldn't register GPU cooling device
```

原因是 `CONFIG_DEVFREQ_THERMAL` 未开，`of_devfreq_cooling_register()` stub
固定返回 `-EINVAL`。已打开该配置并完整构建 #83；#83 `boot_7.3.img`
SHA256：`ffe2f1bd9e6b329bdc919cf04dd1e238943c61716760c745f8f102d0688935fe`。
回归时除上述 GPU/显示/触摸判据外，还应确认该 cooling 错误消失。
