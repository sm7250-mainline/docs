# picasso live_dts UFS 全量梳理——逐属性对照下游 4.19 溯源

> **2026-10-16 纠错**：已更正 ref/TX 频率错位解释、QPHY/GDSC 等价误判和
> “DT 已无缺口”结论。P41 补齐供电模式与 QPHY 投票，待完整构建和实机。
> 详见 [P41 记录](2026-10-16-ufs-power-fix.md)。

> 2026-10-14/15 整理（迭代 33 期间）。数据源等级：`live_dts.dts`（用户
> 提供的运行时权威）为值基准；**下游 4.19 内核源码**
> （`Xiaomi_Kernel_OpenSource-picasso-r-oss/`）+ **工厂 dtsi 源**
> （`kernel_devicetree-picasso-r-oss/factory_downstream(android)_4.19_dts/`，
> 带原始注释）为语义权威。每个属性标注：live 实值 → 消费代码（文件:行）
> → 作用 → 主线 7.3 移植状态。
>
> 配套：bring-up 史与逐轮实判见 [ufs-bringup.md](ufs-bringup.md)；
> 迭代日志见 [copydown-shim-port.md](copydown-shim-port.md) 迭代 29+。

---

## 目录

1. [UFS 相关节点总览](#1-ufs-相关节点总览)
2. [ufshc@1d84000 逐属性](#2-ufshc1d84000-逐属性)
3. [ufsphy_mem@1d87000 逐属性](#3-ufsphy_mem1d87000-逐属性)
4. [供电链全图](#4-供电链全图)
5. [时钟链全表](#5-时钟链全表)
6. [复位链](#6-复位链)
7. [带宽投票：msm-bus 22 档向量与 ad-hoc-bus 拓扑](#7-带宽投票msm-bus-22-档向量与-ad-hoc-bus-拓扑)
8. [PM 与 QoS 属性](#8-pm-与-qos-属性)
9. [alias / symbols / cmdline 关联](#9-alias--symbols--cmdline-关联)
10. [live 与工厂 dtsi 差异](#10-live-与工厂-dtsi-差异)
11. [主线移植对照总表](#11-主线移植对照总表)

---

## 1. UFS 相关节点总览

live_dts 中与 UFS 相关的全部节点（不止 ufshc 本体）：

| 节点 | live 位置 | 角色 |
|---|---|---|
| `ufshc@1d84000` | /soc | UFS 主机控制器（DT 主体，§2） |
| `ufsphy_mem@1d87000` | /soc/rsc@18200000 内 | QMP v4-lito PHY（老式独立框架驱动，§3） |
| `qcom,gdsc@177004`（ufs_phy_gdsc） | /soc | UFS PHY GDSC 电源域，以 regulator 形式被 vdd-hba 消费 |
| `ufs_dev_reset_assert/deassert` | /soc/pinctrl@f000000 | 设备复位脚（TLMM 特殊组 ufs_reset）两态 |
| `mas-xm-ufs-mem` | /soc/ad-hoc-bus | UFS 主端口的 NoC 拓扑 + QoS 节点（aggre1 noc） |
| `slv-qhs-ufs-mem-cfg` | /soc/ad-hoc-bus | UFS 配置空间从端口（config noc，CN0 BCM） |
| alias `ufshc1`、symbol `ufshc_mem`/`ufsphy_mem` | __aliases__/__symbols__ | 多 host 索引 / 符号导出 |
| cmdline `androidboot.bootdevice`、`swiotlb=2048` | /chosen | probe 保护判据 / 32 位 DMA 弹回池 |

## 2. ufshc@1d84000 逐属性

### 2.1 身份与基本资源

| 属性 | live 值 | 消费代码 | 作用 |
|---|---|---|---|
| `compatible` | `"qcom,ufshc"` | `ufs-qcom.c` of_match 表 | 绑定 qcom variant ops（`ufs_hba_qcom_variant`） |
| `reg` | `<0x1d84000 0x3000>` + `<0x1d90000 0x8000>` | `ufshcd-pltfrm.c` 平台资源 | `ufs_mem`=HCI 寄存器 12KB；`ufs_ice`=内联加密引擎 32KB（本机无 crypto，仍映射） |
| `reg-names` | `"ufs_mem","ufs_ice"` | 同上 | 按名取资源 |
| `interrupts` | GIC_SPI 265 LEVEL_HIGH | `ufshcd.c` | UTP/错误/UIC 完成总中断 |
| `phys` / `phy-names` | `<&ufsphy_mem>` / `"ufsphy"` | `devm_phy_get` | PHY 通道；calibrate/power_on 经 phy 框架转发 |
| `status` | `"ok"` | — | 工厂 dtsi 默认 disabled，板级 `xiaomi-sm7250-common.dtsi` 开启 |

### 2.2 协议参数

| 属性 | live 值 | 消费代码 | 作用 |
|---|---|---|---|
| `lanes-per-direction` | `<2>` | `ufshcd-pltfrm.c:447` → `hba->lanes_per_direction` | ① lane 时钟获取数量（`ufs_qcom_init_lane_clks`：>1 才取 rx_lane1/tx_lane1）② 功率协商上限 |
| `dev-ref-clk-freq` | `<0>`（19.2MHz） | `ufshcd-pltfrm.c:380` → `ufshcd.c:8934` `ufshcd_set_dev_ref_clk_freq` | **写设备侧属性 bRefClkFreq**（UPIU query WRITE_ATTR `REF_CLK_FREQ`）——告知设备 host ref clk 频率，设备据此调整内部计时（gear 切换相关性）；枚举：0=19.2M/1=26M/2=38.4M/3=52M，默认 26M |
| `spm-level` | `<3>` | `ufshcd-pltfrm.c:299` → `hba->spm_lvl` | 系统挂起（suspend-to-RAM）目标档 = `UFS_PM_LVL_3`（SLEEP 功率模式 + HIBERN8 链路）。见 §8 档位表 |
| `non-removable` | 存在 | `ufs-qcom.c:2773` probe | **多存储保护**：若 `androidboot.bootdevice != 1d84000.ufshc` 则本 probe 直接 -ENODEV——防止非启动盘的 ufshc 关掉别的存储的供电（bootloader 已开的电） |

### 2.3 时钟（详见 §5）

`clock-names` ×10 + `clocks` ×10 + `freq-table-hz` 10 对。消费：
`ufshcd-pltfrm.c:60 ufshcd_parse_clock_info`——**全部** 10 个时钟进
`hba->clk_list_head`（含 0-0 对，无跳过）；`ufshcd_setup_clocks` 对全部
`clk_prepare_enable`；devfreq scaling 只对**非零**频率对调
`clk_set_rate`（min/max），且按名字跳过 `core_clk_ice_hw_ctl`。
downstream `ufshcd_set_clk_freq`（`ufshcd.c:1487`）rate 失败时
`dev_err + break`（与主线 7.3 同型）。

### 2.4 带宽投票（详见 §7）

`qcom,msm-bus,name = "ufshc_mem"`、`num-cases = <22>`、`num-paths = <2>`、
`vectors-KBps`（22 档 × 2 路径）、`qcom,bus-vector-names`（22 个档名）。
消费：`ufs-qcom.c:1174 ufs_qcom_bus_register` → `msm_bus_cl_get_pdata` +
`msm_bus_scale_register_client`；`pwr_change POST_CHANGE` 时按当前
gear/lane/rate 组名（如 `HS_RB_G3_L2`）在 vector-names 里查索引 →
`msm_bus_scale_client_update_request` 切档。

### 2.5 PM QoS（详见 §8）

`qcom,pm-qos-cpu-groups = <0x3f 0xc0>`（CPU0-5 一组、CPU6-7 一组）、
`qcom,pm-qos-cpu-group-latency-us = <67 67>`、`qcom,pm-qos-default-cpu = <0>`。
消费：`ufs-qcom.c:1820 ufs_qcom_pm_qos_init`。

### 2.6 设备复位 pinctrl（详见 §6）

`pinctrl-names = "dev-reset-assert","dev-reset-deassert"` + 两态引用。
消费：**下游 ufshcd core**（`ufshcd.c:565 ufshcd_device_reset_ctrl`），
仅在 EH/恢复路径（`ufshcd_reset_device`：assert→10µs→deassert→10µs）。

### 2.7 复位与时钟门控（详见 §6）

`resets = <&gcc GCC_UFS_PHY_BCR>` + `reset-names = "core_reset"`。
消费：`ufshcd-pltfrm.c:49` → `hba->core_reset` →
`ufs_qcom_assert/deassert_reset`（PHY 复位脉冲，power_up_sequence 用）。

### 2.8 供电（详见 §4）

| 属性 | live 值 | 消费代码 | 作用 |
|---|---|---|---|
| `vdd-hba-supply` | `<&ufs_phy_gdsc>` | `ufshcd-pltfrm.c:274` populate_vreg | 主机控制器供电——本机接到 **GDSC**（域形式）而非 LDO |
| `vdd-hba-fixed-regulator` | 存在 | `ufshcd-pltfrm.c:186` | 固定电声明：**跳过** max-microamp/电压解析（ regulator_get 仍做，enable 仍会拉 GDSC） |
| `vcc-supply` / `vcc-voltage-level` / `vcc-max-microamp` | pm8150a_l7 / `<2950000 2960000>` / 800000 | `ufshcd-pltfrm.c:203-223` | 闪存 VCC（2.95V/800mA），probe/runtime 按链路态开合 |
| `vccq2-supply` / `vccq2-max-microamp` | pm8150_s4 / 800000 | 同上 | 控制器 IO VCCQ2（s4 固定 1.8V） |
| `qcom,vddp-ref-clk-supply` / `…-max-microamp` | pm8150_l9 / 100 | `ufs-qcom.c:2185 ufs_qcom_parse_reg_info`（缺省电压 1.2V） | **ref clk 引脚供电**：probe 期 `phy_init` 后 enable；仅 link 完全 off（spm_lvl ≥ 4）时 disable——深睡路径专用 |
| `vcc-supply-1p8` / `vccq-pwr-collapse-sup` 等 | 不存在 | — | 可选变体，本机未用（1.8V VCC 设备/挂起断电特性） |

## 3. ufsphy_mem@1d87000 逐属性

| 属性 | live 值 | 消费代码 | 作用 |
|---|---|---|---|
| `compatible` | `"qcom,ufs-phy-qmp-v4-lito"` | `phy-qcom-ufs-qmp-v4-lito.c` | v4-lito 专属 phy_spec_ops：calibrate（写三张表：rate_A 84 项 / 2nd_lane 45 项=lane0 镜像 / rate_B 1 项）、start_serdes、power_control（POWER_DOWN_CONTROL + TX pull-down）、set_tx_lane_enable（**空函数**，v4 无此寄存器） |
| `reg` | `<0x1d87000 0xe00>` + `<0x1d90000 0x8000>` | `phy-qcom-ufs.c:100`（按名 `phy_mem`） | QMP 寄存器 3.5KB；`ufs_ice` 段 v4 不使用（老平台兼容） |
| `#phy-cells` | `<0>` | phy 框架 | 无参数 phy |
| `lanes-per-direction` | `<2>` | `phy-qcom-ufs.c:144` | 2nd lane 初始化（calibrate 的 2nd_lane 表是否写） |
| `qcom,rpmh-resource-name` | `"qphy.lvl"` | `phy-qcom-ufs.c:555` | PHY **模拟轨 RPMh 投票**：power on/off 时 `rpmh_write_async(qphy.lvl 地址, on/off)`（cmd-db 按名查址） |
| `clocks`/`clock-names` | ref_clk_src=rpmh CXO；ref_clk=`GCC_UFS_1X_CLKREF_CLK`(101)；ref_aux_clk=`GCC_UFS_PHY_PHY_AUX_CLK`(107) | `phy-qcom-ufs.c:205 ufs_qcom_phy_init_clks` | ref_clk_src=源；ref_clk=送 PHY 的 19.2M 参考（1x clkref 分支）；ref_aux=PHY 辅助时钟 |
| `vdda-phy-supply`/`…-max-microamp` | pm8150_l5 / 90192µA | `phy-qcom-ufs.c:304` vreg 框架 | PHY 模拟供电 0.88V |
| `vdda-pll-supply`/`…-max-microamp` | pm8150_l9 / 19000µA | 同上 | PLL 供电 1.2V（与 vddp-ref-clk 同一颗 l9） |
| PHY 校准数据 | **无 DT 属性** | 驱动内建三张表 | v4 框架：全部寄存器序列硬编码在驱动（主线 qmp-ufs 同思路，我们已按下游值移植） |

## 4. 供电链全图

```
ufs_phy_gdsc (GDSCR @0x177004, qcom,gdsc 框架)
  └─ vdd-hba（fixed 声明）─────── HCI 主机域：regulator_enable 时 GDSC 拉起
pm8150a_l7 (2.95V) ── vcc ─────── 闪存 VCC（800mA 上限）
pm8150_s4 (1.8V)  ── vccq2 ────── 控制器 IO（800mA 上限）
pm8150_l5 (0.88V) ── vdda-phy ──── PHY 模拟（90.2mA）
pm8150_l9 (1.2V)  ── vdda-pll ──── PHY PLL（19mA）
        └───────── vddp-ref-clk ── ref clk 引脚供电（100µA；深睡才关）
qphy.lvl (RPMh ARC 资源) ───────── PHY 模拟轨开关投票（phy power on/off 时）
```

要点：**l9 一供三**（PLL + ref clk 引脚）；vdd-hba 走 GDSC 域而非 LDO
（`vdd-hba-fixed-regulator` 跳过电压/电流解析）；全部 probe 期开，
深睡（spm_lvl≥4）才分层关。

## 5. 时钟链全表

GCC 编号 = 下游 `include/dt-bindings/clock/qcom,gcc-lito.h`。

### 5.1 ufshc 的 10 个时钟

| clock-name | GCC id | 名称 | freq-table（live） | 角色 |
|---|---|---|---|---|
| core_clk | 103 | UFS_PHY_AXI | 50M–200M | 主机 AXI/核心；**scaling 主时钟** |
| bus_aggr_clk | 129 | AGGRE_UFS_PHY_AXI | 0–0 | aggre noc 聚合时钟（仅 enable） |
| iface_clk | 102 | UFS_PHY_AHB | 0–0 | AHB 配置总线（仅 enable） |
| core_clk_unipro | 112 | UFS_PHY_UNIPRO_CORE | 37.5M–150M | UniPro 核（scaling；`set_core_clk_ctrl` 的 cycles 计算用） |
| core_clk_ice | 105 | UFS_PHY_ICE_CORE | 75M–300M | ICE 时钟；主线于 P38 补齐 RCG 与使能 |
| core_clk_ice_hw_ctl | 138 | ICE_CORE_HW_CTL | 75M–300M | **按名字跳过 rate**（set_clk_freq 里 strcmp 排除） |
| ref_clk | rpmh CXO | bi_tcxo | 0–0 | 送设备的 19.2M 参考；`keep_link_active` 语义（关时钟保链路时它最后关） |
| tx_lane0_sync_clk | 111 | UFS_PHY_TX_SYMBOL_0 | 0–0 | TX 符号时钟 0（PHY 输出馈入的分支） |
| rx_lane0_sync_clk | 109 | UFS_PHY_RX_SYMBOL_0 | 0–0 | RX 符号时钟 0 |
| rx_lane1_sync_clk | 110 | UFS_PHY_RX_SYMBOL_1 | 0–0 | RX 符号时钟 1（2-lane） |

注：下游的 enable 管理**双轨**——core/iface 等由 ufshcd core 的 clk_list
统一 `clk_prepare_enable`；tx/rx lane sync 在
`ufs_qcom_enable_lane_clks`（HCE enable 后按 lanes-per-direction 开）
**再 enable 一次**（PHY PLL 起来后才有效，注释明示该顺序）。

### 5.2 ufsphy 的 3 个时钟

ref_clk_src（rpmh cxo）/ ref_clk（GCC_UFS_1X_CLKREF_CLK=101）/
ref_aux_clk（GCC_UFS_PHY_PHY_AUX_CLK=107）——`phy-qcom-ufs.c` 在
calibrate/power_on 时 enable。

## 6. 复位链

| 复位 | DT | 消费 | 时机 |
|---|---|---|---|
| **PHY 复位**（GCC_UFS_PHY_BCR） | `resets`+`reset-names="core_reset"` | `hba->core_reset` → `ufs_qcom_assert/deassert_reset` | 每次 power_up_sequence：assert → 1ms → PHY calibrate → deassert → 1ms → start_serdes |
| **设备复位**（ufs_reset pad） | pinctrl 两态（assert=output-low / deassert=output-high，均 bias-pull-down + drive 8） | 下游 ufshcd core `ufshcd_reset_device`（assert→10µs→deassert→10µs） | 仅 EH/恢复路径（`ufshcd_detect_device`/link recovery）；**probe 不做**（迭代 32 实证：probe 期脉冲会杀死 link startup） |
| （对照）主机复位 | 无 DT（驱动写 HCE） | `ufshcd_hba_stop/enable` | HCE 寄存器位 |

## 7. 带宽投票：msm-bus 22 档向量与 ad-hoc-bus 拓扑

### 7.1 22 档向量（live 实值 = 工厂 lito.dtsi，带原始注释语义）

每档两路：`<123 512 AB IB>`（路径 1：mas-xm-ufs-mem→…DDR）+
`<1 757 AB IB>`（路径 2：CNOC 侧）。工厂注释：**HS G3 投 4GBps 是为了
把 BIMC 推到 nominal 角、200MBps 推 CNOC 到 nominal；L1 时减半；MAX 档
投得足够高以推 turbo**；HS_G3_RB_L2 的 IB=409600 是实测需要（瞬时带宽
才能拉起 CNOC 时钟）。

| 档 | 路径1 AB/IB | 路径2 AB/IB |
|---|---|---|
| MIN | 0/0 | 0/0 |
| PWM_G1..G4_L1 | 922/1844/3688/7376 | 1000 |
| PWM_G1..G4_L2 | 1844..14752 | 1000 |
| HS_RA_G1/G2/G3_L1 | 127796/255591/2097152 | 1000/1000/102400 |
| HS_RA_G1/G2/G3_L2 | 255591/511181/4194304 | 1000/1000/204800 |
| HS_RB_G1/G2/G3_L1 | 149422/298189/2097152 | 1000/1000/102400 |
| HS_RB_G1/G2_L2 | 298189/596378 | 1000 |
| **HS_RB_G3_L2（本机档）** | **4194304/0** | **204800/409600** |
| MAX | 7643136/0 | 307200/0 |

机制：`pwr_change POST_CHANGE` → `ufs_qcom_get_speed_mode`（组名拼接）
→ `qcom,bus-vector-names` 查索引 → `msm_bus_scale_client_update_request`。
（另有 sysfs `max_bus_bw` 可强制 MAX 档。）

### 7.2 ad-hoc-bus 拓扑节点（UFS 相关两个）

- `mas-xm-ufs-mem`：cell-id 123、buswidth 8、qport 5、prio 2、
  `qcom,connections → slv-qns-a1noc-snoc`、bus-dev=fab-aggre1_noc、
  ap-owned；**QoS 时钟** `clk-aggre-ufs-phy-axi-no-rate`（=bus_aggr_clk）。
  作用：UFS 主端口在 aggre1 noc 的 QoS 寄存器（qport/prio）编程数据 +
  数据路由起点（a1noc→snoc→DDR）。
- `slv-qhs-ufs-mem-cfg`：cell-id 757、bus-dev=fab-config_noc、bcms=CN0。
  作用：UFS 配置寄存器在 config noc 的从端口；路径 2 投票经 CN0 BCM。

（主线对应物：qnoc 的 `xm_ufs_mem`/`qhs_ufs_mem_cfg` 节点 + CN0/SN9 等
BCM——迭代 30 已接两条 interconnect 路径。）

## 8. PM 与 QoS 属性

### 8.1 功率档位（spm-level=3 的语义）

```
LVL 0 = ACTIVE  + LINK_ACTIVE     （全速）
LVL 1 = ACTIVE  + HIBERN8         （自动休眠链路）
LVL 2 = SLEEP   + LINK_ACTIVE     （设备睡眠模式）
LVL 3 = SLEEP   + HIBERN8         ← live spm-level（挂起目标）
LVL 4 = POWERDOWN + HIBERN8       （vddp-ref-clk 会关）
LVL 5 = POWERDOWN + LINK_OFF      （PHY 下电 + qphy.lvl off）
```

spm_level>3 才触碰 vddp-ref-clk / vccq-parent 等深睡资源——**本机配置
（3）下它们全程不动**。

### 8.2 pm-qos

`qcom,pm-qos-cpu-groups=<0x3f 0xc0>`（银核簇+金核簇两组）+ 每组 67µs
latency + 默认 CPU0：UFS 活动（IRQ/命令处理）期间对两组 CPU 提
`cpu_latency_qos` 约束（防簇深度 idle 增大完成延迟）；建独立
`ufs_pm_qos_N` 单线程工作队列做异步 update。纯性能特性，无功能依赖。

### 8.3 ref clk 的三层管理（易混，特此拆分）

1. **频率告知**：`dev-ref-clk-freq` → 设备属性 bRefClkFreq（协议层）；
2. **输出门控**：`ufs_qcom_dev_ref_clk_ctrl` 写 HCI `REG_UFS_CFG1` bit26
   （hw≥2 路径，主机侧 REF_CLK_EN，进 PWM/深睡前关）；
3. **引脚供电**：`qcom,vddp-ref-clk-supply`（l9，深睡才关）。

## 9. alias / symbols / cmdline 关联

| 项 | 值 | 消费 | 作用 |
|---|---|---|---|
| alias `ufshc1` | `/soc/ufshc@1d84000` | `ufs-qcom.c:2082` `of_alias_get_id` | 多 host 索引（`ufs_qcom_hosts[]`，debugfs/print 用） |
| symbols `ufshc_mem`/`ufsphy_mem` | 节点路径 | 仅 /sys/firmware 引用 | 工厂 dtsi 的 label 残留 |
| cmdline `androidboot.bootdevice` | `1d84000.ufshc` | `ufs-qcom.c:2773` | non-removable 的启动盘判据 |
| cmdline `swiotlb=2048` | — | 内核 DMA | 32 位 DMA 弹回池显式尺寸（UFS 控制器 32 位寻址限制的配套） |

## 10. live 与工厂 dtsi 差异

| 项 | 工厂 lito.dtsi | live_dts（运行时） | 说明 |
|---|---|---|---|
| freq-table 第 7/8 对（ref_clk / tx_lane0） | 0–0 | 0–0 | 无差异；旧文把第 5/6 对 ICE 的 75M–300M 错位记到 ref/TX，已纠正 |
| 其余 | 一致 | 一致 | vdd-hba/vcc/vccq2/vddp-ref-clk 在 `xiaomi-sm7250-common.dtsi` 板级追加 |

## 11. 主线移植对照总表

| live 属性/节点 | 主线 7.3 移植状态 |
|---|---|
| compatible/reg/interrupts/phys/lanes-per-direction | ✅ 等价（`qcom,sm7250-ufshc` 前缀 + 主线驱动） |
| dev-ref-clk-freq=0 | ✅ 已带（bRefClkFreq 写设备，主线同机制） |
| spm-level=3 | ➖ 主线无此 DT 属性（`ufs_qcom_init` 仅在 drvdata 带 `no_phy_retention` 时设 `spm_lvl=UFS_PM_LVL_5`，否则用 core 默认档——挂起档语义有差异；bring-up 无挂起，无影响） |
| 10 时钟 + freq-table 10 对 | ✅ P38 已补齐 ICE 两对及 RCG，频率与 live 逐对一致；ref/TX/RX 均为 0–0 |
| non-removable | ➖ 主线无此保护（单存储无影响） |
| msm-bus 22 档向量 | ✅ 换形：qnoc 两条 interconnect 路径（`ufs-ddr`/`cpu-ufs`，ufs-qcom.c bw 表投票，迭代 30）；22 档语义≈bw_table 的 gear 索引 |
| pm-qos 三属性 | ➖ 未移植（纯性能） |
| dev-reset pinctrl 两态 | ✅ 迭代 32 补齐；33 清 REINIT quirk 不足以避免 init 期复位，36 的 link_started 门修复该回归 |
| core_reset（PHY BCR） | ✅ 等价（主线 `devm_reset_control_get_optional(dev,"rst")`（ufs-qcom.c:1619），我们 DT `resets=<&gcc BCR>+reset-names="rst"`；power_up_sequence 的 assert→calibrate→deassert 脉冲同型） |
| vdd-hba=GDSC+fixed | ✅ 换形：主线 `power-domains = <&gcc UFS_PHY_GDSC>`（genpd），语义同（拉 GDSCR） |
| vcc/vccq2/vddp-ref-clk | ◐ vcc/vccq2 已带；**vddp-ref-clk（l9@1.2V/100µA）未带**——深睡路径资源，spm≤3 全程不关，bring-up 无影响；上游化时补 |
| UFS/PHY regulator 模式与负载投票 | ◐ P41 修复 L7C/L5A/L9A 的初始 LPM 和 DRMS 权限缺失，bring-up 固定 HPM；待实机。下游模式阈值与主线硬编码阈值不同，恢复动态模式需另行验证 |
| ufsphy 校准及供电 | ✅ 校准表已移植；P41 三条相关电源固定 HPM，原负载值由驱动表提供 |
| qcom,rpmh-resource-name=qphy.lvl | ◐ P41 通过 RPMH_QLINK_CLK_A 补齐独立 1/0 投票；原先只有 GCC GDSC，二者不等价。PHY 四时钟获取/启停接入既有 bulk 生命周期，待实机 |
| alias `ufshc1` | ➖ 未带（单 host 无消费） |
| cmdline swiotlb | ➖ 主线默认 swiotlb 池（UFS 32 位 mask 已强制，fix7） |

### 11.1 对 READ(10) 战线的提示（P41 纠错）

此前“DT 属性方向穷尽”的结论不成立：只对 ufshc/PHY 的直接属性做对照，
漏掉了 regulator provider 的模式权限、负载阈值，以及 QPHY 的独立投票。
P41 已落码修复，验证回到正常 SCSI 分区扫描。P39 的无数据 CHECK
CONDITION 响应只能证明命令/响应通路可用，不能排除主机 DATA-IN/DMA。

---

*数据快照：live_dts.dts（2026-07-05 导出）；下游源
Xiaomi_Kernel_OpenSource-picasso-r-oss（R 树）；工厂 dtsi
factory_downstream(android)_4.19_dts。行号以下游树为准。*
