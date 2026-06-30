# Redmi K30 5G (Picasso / SM7250 / SDM765G) 硬件信息文档

> 从下游 Linux 4.19 内核设备树 (`picasso-r-oss`) 提取
>
> 生成日期: 2026-07-01

---

## 目录

1. [SoC 总览](#1-soc-总览)
2. [CPU 拓扑](#2-cpu-拓扑)
3. [中断控制器](#3-中断控制器)
4. [时钟控制器](#4-时钟控制器)
5. [保留内存映射](#5-保留内存映射)
6. [电源管理 (PMIC / 调节器 / GDSC)](#6-电源管理)
7. [引脚控制 (TLMM / LPI Pinctrl)](#7-引脚控制)
8. [存储 (UFS / eMMC / SD)](#8-存储)
9. [串行总线 (QUPv3: I²C / SPI / UART / I3C)](#9-串行总线)
10. [USB](#10-usb)
11. [显示子系统 (MDSS / SDE / DSI)](#11-显示子系统)
12. [GPU (Adreno 620)](#12-gpu)
13. [NPU (Hexagon)](#13-npu)
14. [视频编解码 (Venus)](#14-视频编解码)
15. [相机子系统 (CAMSS)](#15-相机子系统)
16. [音频子系统](#16-音频子系统)
17. [互联总线 (NoC / Interconnect)](#17-互联总线)
18. [SMMU / IOMMU](#18-smmu)
19. [热管理 (Thermal)](#19-热管理)
20. [CoreSight 调试追踪](#20-coresight-调试追踪)
21. [IPC 通信 (SMP2P / GLINK / IPCC)](#21-ipc-通信)
22. [网络 (WiFi / Bluetooth / IPA)](#22-网络)
23. [Picasso 专用外设](#23-picasso-专用外设)
24. [GPIO 完整分配表](#24-gpio-完整分配表)
25. [关键寄存器地址速查](#25-关键寄存器地址速查)

---

## 1. SoC 总览

| 属性 | 值 |
|------|-----|
| **芯片型号** | Qualcomm Snapdragon 765G |
| **内部代号** | SM7250 / Lito |
| **制造工艺** | 7nm (N7) Samsung |
| **设备代号** | Picasso |
| **商业名称** | Redmi K30 5G |
| **DTS compatible** | `qcom,lito-mtp`, `qcom,lito`, `qcom,mtp` |
| **MSM ID** | `400` / `440` (0x10000 mask) |
| **Board ID** | `36 0` |

---

## 2. CPU 拓扑

### 2.1 三丛集架构 (1+1+6)

| CPU | 地址 (reg) | 核心类型 | 容量 (DMIPS/MHz) | 功耗系数 | 频率域 | LMH DCVS |
|-----|-----------|----------|-------------------|----------|--------|----------|
| **CPU0** | 0x000 | Kryo 475 Silver (Cortex-A55) | 1024 | 100 | freq-domain0 (level 6) | lmh_dcvs0 |
| **CPU1** | 0x100 | Kryo 475 Silver (Cortex-A55) | 1024 | 100 | freq-domain0 (level 6) | lmh_dcvs0 |
| **CPU2** | 0x200 | Kryo 475 Silver (Cortex-A55) | 1024 | 100 | freq-domain0 (level 6) | lmh_dcvs0 |
| **CPU3** | 0x300 | Kryo 475 Silver (Cortex-A55) | 1024 | 100 | freq-domain0 (level 6) | lmh_dcvs0 |
| **CPU4** | 0x400 | Kryo 475 Silver (Cortex-A55) | 1024 | 100 | freq-domain0 (level 6) | lmh_dcvs0 |
| **CPU5** | 0x500 | Kryo 475 Silver (Cortex-A55) | 1024 | 100 | freq-domain0 (level 6) | lmh_dcvs0 |
| **CPU6** | 0x600 | **Kryo 475 Gold** (Cortex-A76) | **1740** | **341** | freq-domain1 (level 2) | lmh_dcvs1 |
| **CPU7** | 0x700 | **Kryo 475 Prime** (Cortex-A76) | **1740** | **375** | freq-domain2 (level 2) | lmh_dcvs2 |

### 2.2 CPU 集群映射

```
cpu-map {
    cluster0: CPU0, CPU1, CPU2, CPU3, CPU4, CPU5  (6× Silver)
    cluster1: CPU6                                  (1× Gold)
    cluster2: CPU7                                  (1× Prime)
}
```

### 2.3 缓存层次

- **L1 I-Cache**: 每核独立
- **L1 D-Cache**: 每核独立
- **L2 Cache**: 每核独立 (Silver: L2_0~L2_500, Gold: L2_600, Prime: L2_700)
- **L3 Cache (LLCC)**: L3_0 - 共享, 地址 0x9200000

### 2.4 CPU 空闲状态

#### Silver 集群 (CPU0-5)
| 级别 | PSCI 模式 | 进入延迟 | 退出延迟 | 最小驻留 |
|------|-----------|---------|---------|---------|
| WFI | 0x1 | 61 us | 60 us | 121 us |
| PC | 0x3 | 549 us | 901 us | 1774 us |
| Rail-PC | 0x4 | 702 us | 915 us | 4001 us |

#### Gold 集群 (CPU6-7)
| 级别 | PSCI 模式 | 进入延迟 | 退出延迟 | 最小驻留 |
|------|-----------|---------|---------|---------|
| WFI | 0x1 | 55 us | 66 us | 121 us |
| PC | 0x3 | 523 us | 1244 us | 2207 us |
| Rail-PC | 0x4 | 526 us | 1854 us | 5555 us |

#### 集群级别 (L3)
| 级别 | 进入延迟 | 退出延迟 | 说明 |
|------|---------|---------|------|
| WFI | 660 us | 600 us | |
| PC | 2752 us | 3048 us | is-reset |
| CX-Off | 3638 us | 4562 us | notify-rpm |
| LLCC-Off | 3263 us | 6562 us | notify-rpm |

---

## 3. 中断控制器

### 3.1 GIC-v3 (ARM Generic Interrupt Controller)

| 属性 | 值 |
|------|-----|
| **compatible** | `arm,gic-v3` |
| **GICD 基地址** | `0x17a00000` (64KB) |
| **GICR 基地址** | `0x17a60000` (1MB × 8 regions) |
| **redistributor-stride** | 0x20000 |
| **PSCI 方法** | `smc` (PSCI v1.0) |

### 3.2 PDC (Power Domain Controller 中断控制器)

| 属性 | 值 |
|------|-----|
| **compatible** | `qcom,lito-pdc` |
| **基地址** | `0xb220000` (192KB) + `0x17c000f0` (96B) |
| **PDC 范围映射** | `<0 480 42>, <42 612 28>, <70 63 1>, <71 640 15>, <86 522 52>` |

### 3.3 定时器

| 定时器 | 地址 | 中断 |
|--------|------|------|
| ARMv8 Arch Timer | — | PPI 1(sec), 2(non-sec), 3(virt), 0(hyp) |
| ARMv7 Mem Timer | 0x17c20000 | SPI 8-14 (7帧, 仅 frame0 启用) |
| 看门狗 (WDog) | 0x17c10000 | SPI 0 (bark), SPI 1 (bite) |
| 时钟频率 | 19.2 MHz / 32.768 kHz (睡眠) | |

---

## 4. 时钟控制器

### 4.1 时钟控制器总览

| 控制器 | 基地址 | Compatible | 额外寄存器 | 供电 | 功能 |
|--------|--------|-----------|-----------|------|------|
| **GCC** | `0x100000` (1.98MB) | `qcom,gcc-lito` | — | VDD_CX_LEVEL | 全局时钟+复位 |
| **CAMCC** | `0xad00000` (64KB) | `qcom,lito-camcc` | — | VDD_CX_LEVEL + VDD_MX_LEVEL | 相机时钟 |
| **VIDEOCC** | `0xab00000` (64KB) | `qcom,lito-videocc` | — | VDD_CX_LEVEL | 视频时钟(iris efuse) |
| **DISPCC** | `0xaf00000` (128KB) | `qcom,lito-dispcc` | — | VDD_CX_LEVEL | 显示时钟 |
| **GPUCC** | `0x3d90000` (36KB) | `qcom,lito-gpucc` | — | VDD_CX_LEVEL + VDD_MX_LEVEL | GPU时钟 |
| **NPUCC** | `0x9980000` (64KB) | `qcom,lito-npucc` | QDSP6SS: 0x9800000, PLL: 0x9810000 | VDD_CX_LEVEL | NPU时钟(npu efuse) |
| **RPMhCC** | apps_rsc 子节点 | `qcom,lito-rpmh-clk` | apps_rsc: 0x18200000 | — | RPMh 时钟 (XO等) |
| **AOPCC** | AOP QMP | `qcom,aop-qmp-clk` | mboxes: qmp_aop | — | QDSS 调试时钟 |
| **CPUCC** | `0x182a0000` (4B) | `syscon` | — | — | CPU 时钟域参考 |
| **MCCC** | `0x90b0000` (4KB) | `syscon` | — | — | MC 时钟域参考 |

### 4.2 固定时钟

| 时钟名 | 频率 | 用途 |
|--------|------|------|
| `xo_board` | 38.4 MHz | 主晶振 |
| `sleep_clk` | 32.768 kHz | 休眠时钟 |

### 4.3 CPU 频率硬件

| 节点 | 基地址 | 说明 |
|------|--------|------|
| cpufreq_hw freq-domain0 | 0x18323000 | Silver 集群 (CPU0-5) |
| cpufreq_hw freq-domain1 | 0x18325800 | Gold 集群 (CPU6) |
| cpufreq_hw freq-domain2 | 0x18327800 | Prime 集群 (CPU7) |
| L3/LLCC devfreq | 0x18321000 | L3 缓存频率控制 |

---

## 5. 保留内存映射

### 5.1 No-Map 区域（固件/安全独占）

| 区域名 | 物理地址 | 大小 | 用途 |
|--------|---------|------|------|
| hyp_mem | 0x80000000 | 6 MB | Hypervisor |
| xbl_aop_mem | 0x80700000 | 1.375 MB | XBL + AOP 固件 |
| cmd_db | 0x80860000 | 128 KB | RPMh Command DB |
| sec_apps_mem | 0x808ff000 | 4 KB | 安全应用 |
| smem_mem | 0x80900000 | 2 MB | 共享内存 (SMEM) |
| removed_mem | 0x80b00000 | 19 MB | 移除区域 |
| qtee_apps_mem | 0x81e00000 | 66 MB | QTEE 可信执行环境 |
| pil_camera_mem | 0x86000000 | 5 MB | 相机固件 |
| pil_npu_mem | 0x86500000 | 5 MB | NPU 固件 |
| pil_video_mem | 0x86a00000 | 5 MB | 视频固件 |
| pil_cdsp_mem | 0x87400000 | 30 MB | CDSP 固件 |
| pil_adsp_mem | 0x89200000 | 40 MB | ADSP 固件 |
| pil_wlan_fw_mem | 0x8ba00000 | 2 MB | WLAN 固件 |
| pil_ipa_fw_mem | 0x8bc00000 | 64 KB | IPA 固件 |
| pil_ipa_gsi_mem | 0x8bc10000 | 21 KB | IPA GSI |
| pil_gpu_mem | 0x8bc15400 | 8 KB | GPU 固件 |
| reserved_pil | 0x8bc17400 | ~4 MB | PIL 保留 |
| modem_wlan_mem | 0x8c000000 | 248 MB | Modem + WLAN |
| cdsp_sec_mem | 0x9f400000 | 12 MB | CDSP 安全区 |
| cont_splash_memory | 0xA0000000 | 35 MB | 启动画面 |
| dfps_data_memory | 0xA2300000 | 1 MB | DFPS 数据 |
| disp_rdump_memory | 0xA0000000 | 35 MB | 显示崩溃转储 |

### 5.2 共享 DMA 池

| 区域名 | 大小 | 特性 |
|--------|------|------|
| qseecom_mem | 20 MB | 可重用, 4MB 对齐 |
| qseecom_ta_mem | 16 MB | 可信应用 |
| adsp_mem | 12 MB | ADSP 共享 |
| sdsp_mem | 4 MB | 传感器 DSP |
| cdsp_mem | 4 MB | CDSP 共享 |
| secure_display_memory | 160 MB | 安全显示 |
| dump_mem | 40 MB | 崩溃转储 |
| linux,cma | 32 MB | 默认 CMA |

---

## 6. 电源管理

### 6.1 PMIC 设备

SM7250 使用的 PMIC 组合：
- **PM8150** (主 PMIC) - SPMI 地址从 `spmi_bus` (0xc440000)
- **PM8150A** (辅助 PMIC) - 电池/显示供电
- **PM8150L** (LED/显示偏置) - LCD bias, WLED, flash LED
- **PM7250B** (充电/PD) - 充电管理, Type-C PD PHY
- **PM8008** (I²C PMIC) - I²C 地址 0x8 on qupv3_se9_i2c

### 6.2 PM8150 电压轨 (label 后缀 `A`)

#### SMPS 开关电源

| 调节器 | 资源名 | 类型 | 电压范围 | 默认值 |
|--------|--------|------|---------|--------|
| **S1** (VDD_CX) | `cx.lvl` | ARC (level) | Retention–MAX | Retention |
| **S4** | `smpa4` | VRM | 1.80V | 1.80V |
| **S5** | `smpa5` | VRM | 1.664V–2.04V | 1.664V |
| **S6** | `smpa6` | VRM | 0.408V–1.128V | 0.408V |
| **S8** (VDD_MSS) | `mss.lvl` | ARC (level) | Retention–MAX | Retention |

#### LDO 线性调节器

| 调节器 | 电压范围 | 默认值 | 负载电流阈值 | 代理 |
|--------|---------|--------|-------------|------|
| **L1A** | 0.352–0.952V | 0.352V | 0/1 uA | |
| **L2A** | 2.70–3.30V | 2.70V | 0/10mA | |
| **L3A** | 0.312–0.824V | 0.312V | 0/1 uA | |
| **L4A** (VDD_LPI_MX) | ARC level | | | |
| **L5A** | 0.72–1.056V | 0.72V | 0/1 uA | proxy 23.8mA |
| **L6A** | 1.096–1.248V | 1.096V | 0/10mA | |
| **L7A** | 1.648–1.904V | 1.648V | 0/1 uA | |
| **L8A** | 0.72–0.824V | 0.72V | 0/1 uA | |
| **L9A** | 1.152–1.320V | 1.152V | 0/1 uA | proxy 51.8mA |
| **L10A** | 3.00–3.304V | 3.00V | 0/10mA | |
| **L11A** | 0.696–0.880V | 0.696V | 0/1 uA | |
| **L12A** | 1.71–1.89V | 1.71V | 0/1 uA | |
| **L13A** | 3.008–3.304V | 3.008V | 0/1 uA | |
| **L14A** | 1.616–1.984V | 1.616V | 0/10mA | |
| **L15A** | 1.60–1.90V | 1.60V | 0/1 uA | |
| **L16A** | 2.60–3.304V | 2.60V | 0/1 uA | proxy |
| **L17A** | 1.80V | 1.80V | 0/1 uA | |
| **L18A** (VDD_LPI_CX) | ARC level | | | lpi_cx_cdev |

### 6.3 PM8150A 电压轨 (label 后缀 `C`)

| 调节器 | 类型 | 电压范围 | 默认值 |
|--------|------|---------|--------|
| **S2C** | VRM | 0.60–0.80V | 0.60V |
| **S3C** (VDD_EBI) | ARC level | Retention–MAX | |
| **S4C+S5C** (VDD_MX) | ARC level | Retention–MAX | |
| **S6C** (VDD_GFX) | ARC level | Retention–MAX | |
| **S7C** | VRM | 1.128V | 1.128V |
| **S8C** | VRM | 0.696–1.416V | 0.696V |
| **L1C** | VRM | 1.80–1.896V | 1.80V (proxy 62mA) |
| **L2C–L5C** | VRM | 1.14–3.304V | 见 DTS |
| **L6C** | VRM | 1.80–2.96V | 1.80V |
| **L7C** | VRM | 2.704–3.304V | 2.704V |
| **L8C** | VRM | 1.704–1.952V | 1.704V |
| **L9C** | VRM | 2.704–3.304V | 2.704V |
| **L10C–L11C** | VRM | 3.00–3.312V | 3.00V |
| **BOB** | Boost | 3.008–3.96V | PASS mode |

### 6.4 GDSC 电源域

#### GCC 域

| GDSC | 寄存器偏移 | 供电设备 |
|------|-----------|---------|
| `ufs_phy_gdsc` | 0x177004 | UFS PHY |
| `usb30_prim_gdsc` | 0x10f004 | USB3 主控制器 |
| `hlos1_vote_mmnoc_mmu_tbu_hf0_gdsc` | 0x17d050 | 显示 HF TBU0 |
| `hlos1_vote_mmnoc_mmu_tbu_hf1_gdsc` | 0x17d058 | 显示 HF TBU1 |
| `hlos1_vote_mmnoc_mmu_tbu_sf0_gdsc` | 0x17d054 | 相机 SF TBU0 |

#### 相机域 (CAM_CC)

| GDSC | 偏移 | 供电 |
|------|------|------|
| `bps_gdsc` | 0xad07004 | BPS |
| `ipe_0_gdsc` / `ipe_1_gdsc` | 0xad08004 / 0xad09004 | IPE0 / IPE1 |
| `ife_0_gdsc` / `ife_1_gdsc` | 0xad0a004 / 0xad0b004 | IFE0 / IFE1 |
| `titan_top_gdsc` | 0xad0c1c4 | Titan 顶层 |

#### 显示域 (DISP_CC)

| GDSC | 偏移 | 特性 |
|------|------|------|
| `mdss_core_gdsc` | 0xaf03000 | hw-trigger, proxy |

#### GPU 域

| GDSC | 偏移 | 特性 |
|------|------|------|
| `gpu_cx_gdsc` | 0x3d9106c | hw-ctrl @0x3d91540 |
| `gpu_gx_gdsc` | 0x3d9100c | domain-addr @0x3d91508, sw-reset @0x3d91008 |

#### 其他域

| GDSC | 偏移 | 用途 |
|------|------|------|
| `npu_core_gdsc` | 0x9981004 | NPU |
| `mvsc_gdsc` | 0xab00814 | Venus 顶层 |
| `mvs0_gdsc` | 0xab00874 | Venus 核 0 |
| `mvs1_gdsc` | 0xab008b4 | Venus 核 1 |

---

## 7. 引脚控制

### 7.1 TLMM (Top-Level Mode Multiplexer)

| 属性 | 值 |
|------|-----|
| **基地址** | `0x0f000000` (16MB) |
| **compatible** | `qcom,lito-pinctrl` |
| **GPIO 数量** | ≥145 (唤醒能力覆盖 0–144) |
| **中断** | GIC_SPI 208 (LEVEL_HIGH) |
| **wakeup-parent** | PDC |

### 7.2 LPI (Low Power Island) Pinctrl

| 属性 | 值 |
|------|-----|
| **基地址** | `0x33c0000` |
| **Slew 寄存器** | `0x355a000` |
| **GPIO 数量** | 15 (gpio0–gpio14) |
| **支持接口** | MI2S, TDM, AUX, SoundWire (WSA/TX/RX), DMIC |

### 7.3 主要 GPIO 功能组

| 功能名 | 占用 GPIO |
|--------|----------|
| `qup00` ~ `qup15` | QUPv3 串行引擎 (SPI/I²C/UART) |
| `cci_i2c` | 17,18 (CCI0), 19,20 (CCI1), 27,28 (CCI2) |
| `cam_mclk` | 13 (MCLK0), 14 (MCLK1), 15 (MCLK2), 16 (MCLK3), 25 (MCLK4) |
| `mdp_vsync` | 10 (TE0), 11 (TE1) |
| `ibi_i3c` | 42,43 (SE0 I3C), 59,60 (SE6 I3C) |
| `sec_mi2s` | 70 (SCK), 71 (WS), 72 (DATA0), 74 (DATA1) |
| `qdss_cti` | 63 (trace trigger) |

---

## 8. 存储

### 8.1 UFS (Universal Flash Storage)

| 属性 | 值 |
|------|-----|
| **控制器基地址** | `0x1d84000` (12KB) |
| **ICE 基地址** | `0x1d90000` (32KB) |
| **PHY 基地址** | `0x1d87000` (3.5KB) + ICE `0x1d90000` (32KB) |
| **compatible** | `qcom,ufshc` / `qcom,ufs-phy-qmp-v4-lito` |
| **中断** | GIC_SPI 265 |
| **通道数** | 2 lanes |
| **PHY 供电** | vdda-phy: pm8150_l5 (0.72–0.88V, 90.2mA), vdda-pll: pm8150_l9 (1.152–1.2V, 19mA) |
| **控制器供电** | vcc: pm8150a_l7 (2.95V), vccq2: pm8150_s4 (1.8V) |
| **时钟** | 10个时钟（GCC_UFS_PHY_* 系列） |
| **复位** | GCC_UFS_PHY_BCR |

### 8.2 eMMC (SDHC_1)

| 属性 | 值 |
|------|-----|
| **HC 基地址** | `0x7c4000` (4KB) |
| **CQHCI 基地址** | `0x7c5000` (4KB) |
| **ICE 基地址** | `0x7c8000` (32KB) |
| **中断** | GIC_SPI 202 (hc), GIC_SPI 204 (pwr) |
| **总线宽度** | 8-bit |
| **供电** | vdd: pm8150a_l7 (2.95V), vdd-io: pm8150_s4 (1.8V) |
| **compatible** | `qcom,sdhci-msm-v5` + `qcom,sdhci-msm-cqe` |

### 8.3 SD 卡 (SDHC_2) — Picasso 上已禁用

| 属性 | 值 |
|------|-----|
| **HC 基地址** | `0x8804000` (4KB) |
| **中断** | GIC_SPI 207 (hc), GIC_SPI 223 (pwr) |
| **总线宽度** | 4-bit |
| **CD GPIO** | TLMM 69 (ACTIVE_LOW) |
| **状态** | `status = "disabled"` |

---

## 9. 串行总线 (QUPv3)

### 9.1 QUPv3 引擎总览

SM7250 有 2 个 QUPv3 wrapper，共 12 个串行引擎 (SE0-SE11)。

#### Wrapper 0 (North) — 基地址 `0x8c0000`

| 引擎 | 基地址 | 支持模式 | 中断 | DMA (GPI DMA0) |
|------|--------|---------|------|----------------|
| SE0 | 0x880000 | **I3C**, I²C, SPI | 601 | SE0 |
| SE1 | 0x884000 | I²C, SPI | 602 | SE1 |
| SE2 | 0x888000 | **UART console**, I²C, SPI | 603 | SE2 |
| SE4 | 0x890000 | I²C, SPI | 605 | SE4 |
| SE5 | 0x894000 | **UART 4-wire HS**, I²C, SPI | 606 | SE5 |

#### Wrapper 1 (South_1) — 基地址 `0x9c0000`

| 引擎 | 基地址 | 支持模式 | 中断 | DMA (GPI DMA1) |
|------|--------|---------|------|----------------|
| SE6 | 0x980000 | **I3C**, I²C, SPI | 353 | SE0 |
| SE7 | 0x984000 | I²C, SPI | 354 | SE1 |
| SE8 | 0x988000 | UART, I²C, SPI | 355 | SE2 |
| SE9 | 0x98c000 | I²C (shared), SPI | 356 | SE3 |
| SE10 | 0x990000 | I²C, SPI | 357 | SE4 |
| SE11 | 0x994000 | I²C, SPI | 358 | SE5 |

### 9.2 Picasso 上的 I²C 设备

| 总线 | SE# | 设备 | I²C 地址 |
|------|-----|------|---------|
| qupv3_se0_i2c | SE0 | NFC (nq) | 0x28 |
| qupv3_se8_i2c | SE8 | NXP TFA98xx SmartPA | 0x34 |
| qupv3_se9_i2c | SE9 | BQ25970 charge pump, PM8008 | 0x66, 0x8 |

### 9.3 Picasso 上的 SPI 设备

| 总线 | SE# | 设备 | 最大频率 |
|------|-----|------|---------|
| qupv3_se7_spi | SE7 | Novatek NT36672C 触摸屏 | 10 MHz |
| qupv3_se11_spi | SE11 | 红外 LED | 19.2 MHz |

### 9.4 GPI DMA 引擎

| DMA | 基地址 | 中断数 | max-gpii | 流 ID |
|-----|--------|--------|----------|--------|
| gpi_dma0 | 0x800000 (384KB) | 10 (SPI 244–253) | 10 | apps_smmu 0x4f6 |
| gpi_dma1 | 0x900000 (384KB) | 10 (SPI 279–296) | 10 | apps_smmu 0x36 |

---

## 10. USB

### 10.1 USB DWC3 控制器

| 属性 | 值 |
|------|-----|
| **基地址** | `0x0a600000` (1MB) |
| **DWC3 核** | 0x0a600000 + 偏移 (52KB) |
| **中断** | GIC_SPI 240 (LEVEL_HIGH) |
| **compatible** | `qcom,dwc-usb3-msm` / `snps,dwc3` |
| **模式** | DRD (双角色) |
| **最大速度** | SuperSpeed (5Gbps) — Picasso 降为 High-Speed |
| **IOMMU** | apps_smmu 0xE0 (atomic DMA) |
| **GDSC** | usb30_prim_gdsc |
| **PHY** | usb2_phy0 (HS) + usb_qmp_dp_phy (SS+DP) |

### 10.2 USB2 HS PHY

| 属性 | 值 |
|------|-----|
| **基地址** | `0x88e3000` (272B) |
| **compatible** | `qcom,usb-hsphy-snps-femto` |
| **供电** | vdd: pm8150_l5, vdda18: pm8150_l12, vdda33: pm8150_l2 |
| **时钟** | RPMH_CXO_CLK (19.2MHz) |
| **复位** | GCC_QUSB2PHY_PRIM_BCR |
| **Picasso 调优** | `param-override-seq = <0xc8 0x70> <0x24 0x74> <0x64 0x6c>` |

### 10.3 USB3 QMP DP Combo PHY

| 属性 | 值 |
|------|-----|
| **基地址** | `0x88e8000` (12KB) |
| **compatible** | `qcom,usb-ssphy-qmp-dp-combo` |
| **供电** | core: pm8150_l9 (88mA) |
| **时钟** | aux_clk, pipe_clk, ref_clk_src, com_aux_clk |
| **复位** | GCC_USB3_DP_PHY_PRIM_BCR, GCC_USB3_PHY_PRIM_BCR |

### 10.4 USB 中断映射

| 中断 | 来源 | 类型 |
|------|------|------|
| dp_hs_phy_irq | PDC 14 | Edge Rising |
| pwr_event_irq | GIC_SPI 144 | Level High |
| ss_phy_irq | PDC 9 | Level High |
| dm_hs_phy_irq | PDC 15 | Edge Rising |
| hs_phy_irq | PDC 13 | Level High |

---

## 11. 显示子系统 (MDSS / SDE)

### 11.1 顶层

| 属性 | 值 |
|------|-----|
| **MDP 基地址** | `0xae00000` (~528KB) |
| **VBIF 基地址** | `0xaeb0000` (8KB) |
| **中断** | GIC_SPI 83 (IRQ_TYPE_LEVEL_HIGH) |
| **GDSC** | mdss_core_gdsc |

### 11.2 显示管道

| 组件 | 数量 | 偏移 |
|------|------|------|
| CTL (控制器) | 4 | 0x2000, 0x2200, 0x2400, 0x2600 |
| Mixer (图层混合) | 4 | 0x45000–0x48000 (linewidth 2560) |
| DSPP | 2 | 0x55000, 0x57000 |
| SSPP (源管道) | 5 | 2× VIG + 3× DMA |
| INTF (输出接口) | 4 | DP, DSI0, DSI1, DP |
| DSC (压缩) | 2 | 0x81000, 0x81400 |
| WB (回写) | 1 | 0x66000 |
| Dest Scaler | 2 | |
| PP (像素管道) | 4 | |

### 11.3 DSI 控制器与 PHY

#### DSI0

| 属性 | 值 |
|------|-----|
| **控制器基地址** | `0xae94000` (1KB) |
| **PHY 基地址** | `0xae94400` (2KB) |
| **PHY 版本** | v4.1 |
| **PHY VDDA-0.9** | pm8150_l5 (0.88V) |
| **MDSS 中断** | 4 |

#### DSI1

| 属性 | 值 |
|------|-----|
| **控制器基地址** | `0xae96000` (1KB) |
| **PHY 基地址** | `0xae96400` (2KB) |
| **MDSS 中断** | 5 |

### 11.4 DisplayPort

| 属性 | 值 |
|------|-----|
| **基地址** | `0xae90000` |
| **PHY 版本** | 0x420 |
| **最大 PCLK** | 675 MHz |
| **USB PD 检测** | pm7250b_pdphy |
| **CC GPIO** | TLMM 114 |
| **MST** | 支持 |

### 11.5 显示拓扑

```
sde_wb + sde_dsi (主) + sde_dsi1 (副) + sde_dp + sde_rscc
```

### 11.6 Picasso 显示面板

| 面板名称 | 类型 | 分辨率 | 刷新率 | 背光控制 | 接口 |
|---------|------|--------|--------|---------|------|
| Samsung EA8076 F1MP | AMOLED CMD | 1080×2340 | 60Hz | DCS | DSI0 |
| G7A 37 02 0A | LCD DSC Video | 1080×2400 | 30–120Hz (DFPS) | WLED | DSI0 |
| G7A 37 02 0B | LCD DSC Video | 1080×2400 | DFPS | WLED | DSI0 |
| G7A 36 02 0C | LCD DSC Video | 1080×2400 | DFPS | WLED | DSI0 |
| J9 38 0A 0A | LCD Video | 1080×2400 | 60Hz | DCS | DSI0 |

### 11.7 显示供电

| 供电轨 | 电压 | 最大负载 |
|--------|------|---------|
| vddio | 1.8V | 62mA |
| vdd | 3.0V | 857mA |
| lab (AMOLED) | 4.6–6.0V | 100mA |
| ibb (AMOLED) | −4.6 ~ −6.0V | 100mA |

### 11.8 WLED 背光 (Picasso)

| 属性 | 值 |
|------|-----|
| **控制器** | pm8150l_wled |
| **string-cfg** | 7 |
| **OVP** | 28V |
| **FS 电流限制** | 22.5A |
| **CABC** | 启用, cabc-pwm = <2924 4095>, cabcoff-dbv = 200 |

---

## 12. GPU (Adreno 620)

### 12.1 GPU 核心

| 属性 | 值 |
|------|-----|
| **compatible** | `qcom,kgsl-3d0` |
| **芯片 ID** | `0x06020000` (Adreno 620) |
| **基地址** | `0x3d00000` (256KB) |
| **CX_DBGC** | `0x3d61000` (2KB) |
| **RSCC** | `0x3de0000` (4KB) |
| **ISENSE_CNTL** | `0x3d8b000` (8KB) |
| **中断** | GIC_SPI 300 (LEVEL_HIGH) |
| **UBWC** | mode 3, highest bank bit 14, min access 32 |
| **Snapshot** | 2MB |
| **idle timeout** | 80ms |
| **初始性能级别** | 3 (LOW_SVS: 275 MHz) |

### 12.2 GMU (Graphics Management Unit)

| 属性 | 值 |
|------|-----|
| **寄存器** | `0x3d6a000` (192KB) |
| **PDC CFG** | `0xb290000` (64KB) |
| **PDC SEQ** | `0xb490000` (64KB) |
| **中断** | SPI 304 (kgsl_hfi_irq), SPI 305 (kgsl_gmu_irq) |
| **Mailbox** | qmp_aop (mbox "aop") |

### 12.3 GPU OPP 表

| 频率 | 电压级别 |
|------|---------|
| 750 MHz | TURBO |
| 700 MHz | NOM_L1 |
| 670 MHz | NOM_L1 |
| 625 MHz | NOM |
| 540 MHz | NOM |
| 500 MHz | SVS_L1 |
| 400 MHz | SVS |
| 275 MHz | LOW_SVS |
| 0 MHz | OFF |

### 12.4 GPU 速度分级

| Speed Bin | 最高频率 | 说明 |
|-----------|---------|------|
| 0 (default) | 625 MHz | 标准 |
| 132 | 625 MHz | 高性能 |
| 115 | 540 MHz | 低频版本 |

### 12.5 GPU 固件

| 属性 | 值 |
|------|-----|
| **PAS ID** | 13 |
| **固件名** | "a620_zap" |
| **内存区域** | pil_gpu_mem (8KB) |

### 12.6 GPU SMMU (kgsl_smmu)

| 属性 | 值 |
|------|-----|
| **基地址** | `0x3da0000` (64KB) |
| **TCU** | `0x3dc2000` |
| **compatible** | `qcom,qsmmu-v500` |
| **全局中断** | GIC_SPI 673 |
| **CB 中断** | SPI 678–685 |

**GPU TBU:**
| TBU | 基地址 | 流 ID | 中断 |
|-----|--------|-------|------|
| gfx_0_tbu | 0x3dc5000 | 0x0–0x3FF | SPI 687 |
| gfx_1_tbu | 0x3dc9000 | 0x400–0x7FF | SPI 688 |

---

## 13. NPU (Hexagon 神经处理单元)

### 13.1 核心

| 属性 | 值 |
|------|-----|
| **compatible** | `qcom,msm_npu` |
| **TCM** | `0x9900000` (128KB) |
| **核心** | `0x99F0000` (64KB) |
| **CC** | `0x9980000` (64KB) |
| **共享内存** | `0x17c00000` (64KB) |
| **TCSR** | `0x01F40000` (256KB) |
| **QFPROM** | `0x780000` (28KB) |

### 13.2 中断

| 中断 | 号 | 类型 |
|------|-----|------|
| error_irq | GIC_SPI 364 | LEVEL_HIGH |
| wdg_bite_irq | GIC_SPI 366 | EDGE_RISING |
| ipc_irq | GIC_SPI 368 | EDGE_RISING |
| general_irq | GIC_SPI 365 | LEVEL_HIGH |

### 13.3 NPU 供电级别

| 级别 | vreg | DSP Core Freq (Max) |
|------|------|---------------------|
| 0 (SVS) | 1 | 300 MHz |
| 1 | 2 | 400 MHz |
| 2 | 3 | 500 MHz |
| 3 (NOM) | 4 | 660 MHz |
| 4 (TURBO) | 6 | 800 MHz |
| 5 (NOM_TURBO) | 7 | 800 MHz |

### 13.4 NPU 固件

| 属性 | 值 |
|------|-----|
| **PAS ID** | 23 |
| **内存** | pil_npu_mem (5MB) |
| **时钟** | 24 个 NPUCC 时钟 |

---

## 14. 视频编解码 (Venus)

### 14.1 核心

| 属性 | 值 |
|------|-----|
| **基地址** | `0xaa00000` (2MB) |
| **中断** | GIC_SPI 174 (LEVEL_HIGH) |
| **compatible** | `qcom,msm-vidc`, `qcom,lito-vidc` |

### 14.2 供电域

| 供电 | GDSC |
|------|------|
| iris-ctl | mvsc_gdsc |
| vcodec | mvs0_gdsc |
| cvp | mvs1_gdsc |

### 14.3 时钟

| 时钟名 | 来源 | 可伸缩 |
|--------|------|--------|
| mvsc_ctl_axi | VIDEO_CC_MVSC_CTL_AXI_CLK | No |
| core_clk | VIDEO_CC_MVSC_CORE_CLK | **Yes** |
| vcodec_clk | VIDEO_CC_MVS0_CORE_CLK | **Yes** |
| cvp_clk | VIDEO_CC_MVS1_CORE_CLK | **Yes** |

### 14.4 允许的频率

| SKU | 频率 |
|-----|------|
| vidc0 (标准) | 240, 338, 365 MHz |
| vidc1 (低配) | 200 MHz |

### 14.5 Venus IOMMU 上下文银行

| CB 名称 | SMMU SID | 用途 |
|---------|----------|------|
| venus_ns | 0x1380 (mask 0x60), 0x1360 | 非安全 |
| venus_sec_non_pixel | 0x1304 (mask 0xe0), VMID 0xB | 安全非像素 |
| venus_sec_bitstream | 0x1361 (mask 0x4), VMID 0x9 | 安全比特流 |
| venus_sec_pixel | 0x1303 (mask 0xe0), VMID 0xA | 安全像素 |

---

## 15. 相机子系统 (CAMSS)

### 15.1 Picasso 相机配置 (6 摄)

| 索引 | 摄像头 | 像素 | CCI 总线 | CSI PHY | MCLK | MCLK GPIO | 复位 GPIO | 方向 (R/P/Y) |
|------|--------|------|---------|---------|------|-----------|-----------|---------------|
| 0 | **后置主摄 (Wide)** | 64MP | CCI0 M1 | csiphy1 | MCLK1 | GPIO 14 | GPIO 29 | 90/0/180 |
| 1 | **后置超广角 (UltraWide)** | 8MP | CCI1 M0 | csiphy0 | MCLK2 | GPIO 15 | GPIO 30 | 90/0/180 |
| 2 | **前置主摄 (Front)** | 20MP | CCI0 M0 | csiphy2 | MCLK0 | GPIO 13 | GPIO 32 | 270/0/0 |
| 3 | **后置微距 (Macro)** | 5MP | CCI1 M0 | csiphy3 | MCLK4 | GPIO 25 | GPIO 23 | 90/0/180 |
| 4 | **后置景深 (Depth)** | 2MP | CCI0 M0 | csiphy0 | MCLK3 | GPIO 16 | GPIO 21 | 90/0/180 |
| 5 | **前置景深 (Front Depth)** | 2MP | CCI0 M1 | csiphy3 | MCLK2 | GPIO 15 | GPIO 33 | 270/0/0 |

### 15.2 相机 EEPROM

| EEPROM | 关联传感器 | CCI 设备 | CCI Master |
|--------|-----------|---------|------------|
| eeprom_wide | 后置 64M (idx 0) | 1 | 1 |
| eeprom_rear_ultra | 后置 8M (idx 1) | 0 | 0 |
| eeprom_front | 前置 20M (idx 2) | 1 | 0 |
| eeprom_rear_macro | 后置 5M (idx 3) | 0 | 0 |
| eeprom_rear_depth | 后置 2M (idx 4) | 1 | 0 |
| eeprom_front_depth | 前置 2M (idx 5) | 1 | 1 |

### 15.3 相机供电

#### 固定 GPIO LDO

| LDO | 电压 | 使能 GPIO | 输入供电 |
|-----|------|----------|---------|
| camera_rear_wide_vana_ldo | 2.8V | TLMM 127 | BOB |
| camera_rear_wide_vana2_ldo | 1.8V | TLMM 127 | S5A |
| camera_rear_wide_dvdd_ldo | 1.2V | TLMM 45 | S8C |
| camera_rear_ultrawide_vio_ldo | 1.8V | TLMM 68 | S5A |
| camera_rear_ultrawide_vana_ldo | 2.8V | TLMM 122 | BOB |
| camera_rear_ultrawide_vdig_ldo | 1.2V | TLMM 126 | S8C |
| camera_rear_front_vana_ldo | 2.8V | TLMM 125 | BOB |
| camera_rear_front_vdig_ldo | 1.2V | TLMM 125 | S8C |
| camera_rear_macro_vana_ldo | 2.8V | TLMM 56 | BOB |
| camera_rear_macro_vdig_ldo | 1.2V | pm8150l_gpios 12 | S8C |

所有传感器 MCLK 频率: 19.2 MHz, camcc 控制级别为 "turbo"。

### 15.4 闪光灯

| Flash | 闪光源 | 手电筒源 | 开关源 |
|-------|--------|---------|--------|
| 所有 4 个 | pm8150l_flash0 + flash1 | pm8150l_torch0 + torch1 | pm8150l_switch2 |

### 15.5 马达

| 马达 | 关联 | CCI Master | VAF 供电 |
|------|------|-----------|---------|
| actuator_wide | 后置 64M | 1 | L16A (2.8V) |
| actuator_rear_macro | 后置 5M | 0 | L16A (2.8V) |

---

## 16. 音频子系统

### 16.1 音频架构

| 组件 | 值 |
|------|-----|
| **Machine Driver** | `lito-mtp-snd-card` |
| **Codec** | WCD9385 (via SoundWire) |
| **宏** | Bolero (num-macros = 3: TX, RX, VA) |
| **WSA 扬声器** | 已禁用 |
| **MI2S** | 6 个端口均为 Master |

### 16.2 麦克风路由

| 输入 | 偏置 | 物理麦克风 |
|------|------|----------|
| AMIC1 | MIC BIAS1 (2750mV) | 听筒麦克风 |
| AMIC2 | MIC BIAS2 (2750mV) | 耳机麦克风 |
| AMIC5 | MIC BIAS3 (2750mV) | 副麦克风 |

### 16.3 SoundWire

| SWR 总线 | 用途 | 状态 |
|----------|------|------|
| TX SWR | ADC 输出 | 活动 |
| RX SWR | 耳机输出 | 活动 |
| VA SWR | 语音激活 | 活动 |
| WSA SWR | 扬声器 | **已禁用** |

### 16.4 SmartPA (NXP TFA98xx)

| 属性 | 值 |
|------|-----|
| **I²C 总线** | qupv3_se8_i2c |
| **I²C 地址** | 0x34 |
| **compatible** | `nxp,tfa98xx` |
| **复位 GPIO** | TLMM 58 |
| **中断 GPIO** | TLMM 65 |

---

## 17. 互联总线 (NoC / Interconnect)

### 17.1 NoC 基地址

| NoC | 基地址 | 大小 |
|-----|--------|------|
| Aggre1 NOC | 0x016E0000 | 0x15080 |
| Aggre2 NOC | 0x1700000 | 0x23080 |
| Config NOC | 0x1500000 | 0x28000 |
| DC NOC | 0x9160000 | 0x3200 |
| GEM NOC / MC Virt | 0x9680000 | 0x3e200 |
| MMSS NOC | 0x1740000 | 0x1c100 |
| System NOC | 0x1620000 | 0x19200 |
| Compute NOC | 0x1700000 | 0x3d180 |
| NPU NOC | 0x9990000 | 0x1600 |

### 17.2 主要带宽监控器

| 监控器 | 基地址 | 中断 | 监控对象 |
|--------|--------|------|---------|
| cpu_cpu_llcc_bwmon | 0x90B6300 | SPI 581 | CPU → LLCC |
| cpu_llcc_ddr_bwmon | 0x90cd000 | SPI 263 | LLCC → DDR |
| npu_npu_llcc_bwmon | 0x00060400 | SPI 121 | NPU → LLCC |
| npu_llcc_ddr_bwmon | 0x90CE000 | SPI 263 | NPU LLCC → DDR |
| npudsp_npu_ddr_bwmon | 0x00070300 | SPI 205 | NPUDSP → DDR |

---

## 18. SMMU / IOMMU

### 18.1 apps_smmu (系统 SMMU)

| 属性 | 值 |
|------|-----|
| **基地址** | `0x15000000` (1MB) |
| **TCU** | `0x15182000` (32B) |
| **compatible** | `qcom,qsmmu-v500` |
| **全局中断** | GIC_SPI 65 |
| **CB 中断** | SPI 97–118, 181–192, 315–345, 395–409 等 |

#### 系统 TBU

| TBU | 基地址 | 流 ID | 中断 | GDSC |
|-----|--------|-------|------|------|
| anoc_1_tbu | 0x15185000 | 0x0–0x3FF | SPI 423 | — |
| anoc_2_tbu | 0x15189000 | 0x400–0x7FF | SPI 424 | — |
| mnoc_hf_0_tbu | 0x1518D000 | 0x800–0xBFF | SPI 425 | hf0_gdsc |
| mnoc_hf_1_tbu | 0x15191000 | 0xC00–0xFFF | SPI 689 | hf1_gdsc |
| mnoc_sf_0_tbu | 0x15195000 | 0x1000–0x13FF | SPI 690 | sf0_gdsc |
| adsp_tbu | 0x15199000 | 0x1400–0x17FF | SPI 691 | — |
| compute_dsp_0_tbu | 0x1519D000 | 0x1800–0x1BFF | SPI 692 | — |
| compute_dsp_1_tbu | 0x151A1000 | 0x1C00–0x1FFF | SPI 693 | — |

### 18.2 SMMU 特性

- 3 级页表 (qcom,use-3-lvl-tables)
- 跳过初始化 (qcom,skip-init)
- kgsl_smmu: 无动态 ASID (qcom,no-dynamic-asid)

---

## 19. 热管理 (Thermal)

### 19.1 温度传感器 (TSENS)

| 传感器 | 基地址 | TM 基地址 | 中断 |
|--------|--------|----------|------|
| **tsens0** | 0xc222000 (8B SROT) | 0xc263000 (511B TM) | PDC 26/28/20 |
| **tsens1** | 0xc223000 (8B SROT) | 0xc265000 (511B TM) | PDC 27/29/21 |

### 19.2 关键温控区

| 温控区 | 传感器 | 类型 | 触发温度 | 冷却措施 |
|--------|--------|------|---------|---------|
| gpuss-max-step | — | step_wise | 95°C | GPU 降频 |
| cpu-0-0~5-step | tsens0 1–6 | step_wise | 110°C | 单独隔离各 Silver 核 |
| cpu-1-0~3-step | tsens0 9–12 | step_wise | 110°C | 单独隔离各 Gold 核 |
| q6-hvx-step | tsens1 4 | step_wise | 100°C | CDSP rm + GPU + modem + NPU 联合降频 |
| npu-step | tsens1 6 | step_wise | 95°C | NPU 降频 |
| mdm-vec/scl/core-step | tsens1 8–11 | step_wise | 95/105/115°C | Modem 热节流三级 |

### 19.3 LMH (Limit Management Hardware)

| LMH | 基地址 | 中断 | 管理集群 |
|-----|--------|------|---------|
| lmh_dcvs0 | 0x18358800 | SPI 32 | Silver 集群 (CPU0-5) |
| lmh_dcvs1 | 0x18350800 | SPI 33 | Gold 集群 (CPU6-7) |

---

## 20. CoreSight 调试追踪

### 20.1 关键组件基地址

| 组件 | 基地址 |
|------|--------|
| STM (System Trace) | 0x6002000 (stimulus: 0x16280000) |
| TMC ETR | 0x6048000 (BAM: 0x6064000), 4MB buffer |
| TMC ETF SWAO | 0x6b05000 |
| Main Replicator | 0x6046000 |
| SWAO Replicator | 0x6b06000 |
| APSS Funnel | 0x7800000 |
| ETM0–ETM7 | 0x7040000–0x7740000 (每核 1MB stride) |
| CTI CPU0–CPU7 | 0x7020000–0x7720000 (每核 1MB stride) |
| Debug APB 时钟 | aopcc QDSS_CLK |

### 20.2 追踪数据流

```
ETM0–7 (CPU)
  → funnel_apss (0x7800000)
    → funnel_apss_merg (0x7810000)
      → funnel_in1 (0x6042000)
        → funnel_merg (0x6045000)
          → funnel_swao (0x6b04000)
            → tmc_etf_swao (0x6b05000)
              → replicator_swao (0x6b06000)
                → replicator_qdss (0x6046000)
                  → tmc_etr (0x6048000) → USB/BAM 输出
```

---

## 21. IPC 通信

### 21.1 IPCC

| 属性 | 值 |
|------|-----|
| **基地址** | `0x408000` (4KB) |
| **中断** | GIC_SPI 229 |
| **客户端** | AOP, MPSS, LPASS, CDSP, NPU |

### 21.2 SMP2P (4 个通道)

| 通道 | 远程处理器 | 本地 PID | 远程 PID | IPCC 客户端 |
|------|-----------|---------|---------|------------|
| mpss | Modem | 0 | 1 | IPCC_CLIENT_MPSS |
| adsp | ADSP/LPASS | 0 | 2 | IPCC_CLIENT_LPASS |
| cdsp | CDSP | 0 | 5 | IPCC_CLIENT_CDSP |
| npu | NPU | 0 | 10 | IPCC_CLIENT_NPU |

### 21.3 GLINK (4 个通道)

| 通道 | 远程处理器 | PID | IPCC 客户端 |
|------|-----------|-----|------------|
| modem | MPSS | 1 | IPCC_CLIENT_MPSS |
| adsp | LPASS | 2 | IPCC_CLIENT_LPASS |
| cdsp | CDSP | 5 | IPCC_CLIENT_CDSP |
| npu | NPU | 10 | IPCC_CLIENT_NPU |

### 21.4 QMP AOP

| 属性 | 值 |
|------|-----|
| **基地址** | `0xc300000` (4KB + msg ram) |
| **中断** | IPCC_CLIENT_AOP (EDGE_RISING) |
| **mbox-cells** | 1 |

### 21.5 PIL (外设镜像加载器)

| 子系统 | 基地址 | PAS ID | 内存 | 中断 | 供电 |
|--------|--------|--------|------|------|------|
| Modem (MSS) | 0x4080000 | 4 | modem_wlan_mem | SPI 266 | VDD_CX + VDD_MSS |
| ADSP/LPASS | 0x3000000 | 1 | pil_adsp_mem | PDC 16 | — |
| CDSP/Turing | 0x8300000 | 18 | pil_cdsp_mem | SPI 578 | VDD_CX |
| Venus | 0xaae0000 | 9 | pil_video_mem | — | mvsc_gdsc |
| GPU | — | 13 | pil_gpu_mem | — | — |
| NPU | 0x9800000 | 23 | pil_npu_mem | — | — |
| IPA | — | 0xf | pil_ipa_fw_mem | — | — |

---

## 22. 网络

### 22.1 WiFi (ICNSS)

| 属性 | 值 |
|------|-----|
| **compatible** | `qcom,icnss` |
| **基地址** | `0x18800000` (8MB) / `0xb0000000` (64KB IPA) |
| **中断 (CE0-CE11)** | SPI 132, 81, 415, 416, 417, 86, 94, 96, 82, 119, 120, 87 |
| **IOMMU** | apps_smmu 0xC0 (mask 1) |
| **内存** | pil_wlan_fw_mem (2MB) |
| **供电** | vdd-cx-mx, vdd-1.8-xo, vdd-1.3-rfa, vdd-3.3-ch0/ch1 |

### 22.2 蓝牙

| 属性 | 值 |
|------|-----|
| **compatible** | `qca,wcn3990` |
| **BT-SW-CTRL GPIO** | TLMM 73 |

### 22.3 IPA (IP Accelerator)

| 属性 | 值 |
|------|-----|
| **compatible** | `qcom,ipa` (hw v18 = IPA v4.7) |
| **基地址** | `0x1e00000` (528KB IPA) + `0x1e04000` (140KB GSI) |
| **中断** | GIC_SPI 311 (ipa), GIC_SPI 432 (gsi) |
| **IOMMU** | AP: 0x0520, WLAN: 0x0521, UC: 0x0522 |

---

## 23. Picasso 专用外设

### 23.1 触摸屏 (Novatek NT36672C)

| 属性 | 值 |
|------|-----|
| **接口** | SPI (qupv3_se7_spi) |
| **compatible** | `novatek,NVT-ts-spi` |
| **最大频率** | 5 MHz (SPI) / 10 MHz (spi-for-tp) |
| **复位 GPIO** | TLMM 4 (+ PM8150L GPIO 3 for TDDI) |
| **中断 GPIO** | TLMM 5 (2001 flags) |
| **CS GPIO** | TLMM 9 |
| **面板配置数** | 3 (cfg_0: vendor 0x41+maker 0x37, cfg_1: 0x41+0x37+glass 0x01, cfg_2: 0x46+0x36) |
| **分辨率** | 18×36 (X_Channel=18, Y_Channel=36) |

### 23.2 指纹传感器

#### Goodix

| 属性 | 值 |
|------|-----|
| **compatible** | `goodix,fingerprint` |
| **复位 GPIO** | TLMM 63 |
| **中断 GPIO** | TLMM 121 |
| **供电** | pm8150_l13 (2.8V) |

#### FPC1020

| 属性 | 值 |
|------|-----|
| **compatible** | `fpc,fpc1020` |
| **复位 GPIO** | TLMM 63 |
| **中断 GPIO** | TLMM 121 |

### 23.3 充电系统

#### BQ25970 Charge Pump (I²C)

| 属性 | 值 |
|------|-----|
| **I²C 总线** | qupv3_se9_i2c |
| **I²C 地址** | 0x66 |
| **compatible** | `ti,bq2597x-standalone` |
| **中断 GPIO** | TLMM 64 |
| **电池 OVP 阈值** | 4480mV |

#### PM7250B 充电器

| 属性 | 值 |
|------|-----|
| **最大充电电流** | 6000mA (FCC) |
| **末端电压** | 4450mV |
| **USB 输入限流** | 3000mA |
| **步进充电** | 启用 (6 steps: 3.01V–4.45V) |
| **JEITA** | 启用 (0–50°C: 880mA, …, 451–580°C: 2200mA) |
| **FFC** | 启用 (Flash Fast Charge) |
| **VBUS 禁用 GPIO** | TLMM 67 |
| **PD PHY** | pm7250b_pdphy |

#### 小米充电协议

| 节点 | compatible | 最大 BAT 电压 | 最大 BAT 电流 | 最大 BUS 电压 |
|------|-----------|-------------|-------------|-------------|
| cp_qc30 | `xiaomi,cp-qc30` | 4450mV | 3600mA | 12000mV |
| usbpd_pm | `xiaomi,usbpd-pm` | 4450mV | 6000mA | 12000mV |

### 23.4 电池 (4500mAh)

| 属性 | 值 |
|------|-----|
| **标称容量** | 4500mAh |
| **最大电压** | 4.40V (FFC: 4.45V) |
| **快充电流** | 6000mA |
| **Batt ID 电阻** | 100kΩ |
| **电池类型** | "Nonstandard" |
| **电量计** | PM7250B QG (带 SOC 小数率, 线性化, TCSS) |
| **VBATT 切断** | 3400mV |
| **VBATT 低电** | 3500mV |
| **VBATT 空电** | 3100mV (冷: 3000mV) |

### 23.5 其他外设

| 外设 | compatible | 接口 | GPIO |
|------|-----------|------|------|
| **红外 LED** | `ir-spi` | SPI (qupv3_se11_spi, 19.2MHz) | — |
| **SIM 卡检测** | `xiaomi,simtray-status` | GPIO | TLMM 82 |
| **单总线 (DS28E16)** | `maxim,ds28e16` / `xiaomi,onewire_gpio` | GPIO | TLMM 66 |
| **按键** | `gpio-keys` | GPIO | PM8150L GPIO 5 (VOL_UP, ACTIVE_LOW) |
| **NFC** | `nq-nci` | I²C (0x28) | IRQ=34, VEN=12, FIRM=35, CLKREQ=31 |

---

## 24. GPIO 完整分配表

### 24.1 TLMM GPIO 快查表

| GPIO | 功能 | 用途 |
|------|------|------|
| 0–3 | qup01 | SE1 I²C/SPI |
| 4 | gpio | 触摸屏复位 (Novatek) |
| 5 | gpio | 触摸屏中断 (Novatek) |
| 6–9 | qup11 | SE7 SPI (触摸屏 CS=9) |
| 10–11 | mdp_vsync | 显示 TE0/TE1 |
| 12 | gpio | NFC VEN |
| 13 | cam_mclk | MCLK0 (前置 20M + 前置景深) |
| 14 | cam_mclk | MCLK1 (后置 64M) |
| 15 | cam_mclk | MCLK2 (超广角 8M + 前置景深) |
| 16 | cam_mclk | MCLK3 (后置景深 2M) |
| 17–18 | cci_i2c | CCI0 I²C |
| 19–20 | cci_i2c | CCI1 I²C |
| 21 | gpio | 相机复位 (后置超广角/景深) |
| 23 | gpio | 相机复位 (微距) |
| 25 | cam_mclk | MCLK4 (微距 5M) |
| 27–28 | cci_i2c | CCI2 I²C |
| 29 | gpio | 相机复位 (后置 64M) |
| 30 | gpio | 相机复位 (后置超广角 8M) |
| 31 | gpio | NFC CLKREQ |
| 32 | gpio | 相机复位 (前置 20M) |
| 33 | gpio | 相机复位 (前置景深) |
| 34–35 | gpio/qup02 | NFC IRQ+I²C |
| 36–37 | qup02 | SE2 UART |
| 38–41 | qup05 | SE5 UART 4-wire HS |
| 42–45 | qup00 | SE0 I²C/SPI/I3C |
| 45 | gpio | 64M Wide DVDD LDO 使能 |
| 46–48 | qup13 | SE9 I²C/SPI |
| 49–52 | qup12 | SE8 UART/I²C/SPI |
| 53–56 | qup14 | SE10 I²C/SPI |
| 56 | gpio | 微距 VANA LDO 使能 |
| 57 | gpio | WCD938x 复位 |
| 58 | gpio | WSA Spkr1 SD / SmartPA 使能 |
| 59–62 | qup10 | SE6 I²C/SPI/I3C |
| 63 | gpio | 指纹复位 |
| 64 | gpio | BQ2597x 中断 |
| 65 | gpio | SmartPA 中断 / WSA Spkr2 SD |
| 66 | gpio | OneWire |
| 67 | gpio | USB VBUS 禁用 |
| 68 | gpio | 相机 VIO LDO 使能 |
| 69 | gpio | SD 卡检测 |
| 70,71,72,74 | sec_mi2s | 辅助 I²S |
| 73 | gpio | 蓝牙 SW-CTRL |
| 82 | gpio | SIM 卡检测 |
| 108–109,112–113 | qup15 | SE11 I²C/SPI (红外 SPI) |
| 114 | gpio | DP USB Plug CC |
| 121 | gpio | 指纹中断 |
| 122 | gpio | 超广角 VANA LDO 使能 |
| 125 | gpio | 前置 VANA/VDIG LDO 使能 |
| 126 | gpio | 超广角 VDIG LDO 使能 |
| 127 | gpio | 64M Wide VANA LDO 使能 |

---

## 25. 关键寄存器地址速查

| 地址 | 大小 | 模块 |
|------|------|------|
| `0x00100000` | 1.98MB | GCC 时钟控制器 |
| `0x00408000` | 4KB | IPCC |
| `0x00780000` | 28KB | QFPROM (熔丝) |
| `0x007c4000` | 4KB | SDHC1 (eMMC) |
| `0x00880000` | 384KB | GPI DMA0 (QUPv3 North) |
| `0x00880000` | — | QUPv3 SE0–SE5 |
| `0x00900000` | 384KB | GPI DMA1 (QUPv3 South) |
| `0x00940000` | — | QUPv3 SE6–SE11 |
| `0x00a600000` | 1MB | USB3 DWC3 控制器 |
| `0x00aa00000` | 2MB | Venus 视频编解码 |
| `0x00ab00000` | 64KB | VIDEOCC 时钟控制器 |
| `0x00ad00000` | 64KB | CAMCC 时钟控制器 |
| `0x00ae00000` | ~528KB | MDSS MDP (显示) |
| `0x00ae90000` | — | DisplayPort |
| `0x00ae94000` | 1KB | DSI0 控制器 |
| `0x00ae96000` | 1KB | DSI1 控制器 |
| `0x00af00000` | 128KB | DISPCC 时钟控制器 |
| `0x00b220000` | 192KB | PDC 中断控制器 |
| `0x00c440000` | — | SPMI PMIC Arbiter |
| `0x00f000000` | 16MB | TLMM Pinctrl |
| `0x00ff1000` | 132B | Refgen 调节器 |
| `0x016E0000` | — | Aggre1 NOC |
| `0x01700000` | — | Aggre2 NOC / Compute NOC |
| `0x01500000` | — | Config NOC |
| `0x015000000` | 1MB | apps_smmu (系统 SMMU) |
| `0x01740000` | — | MMSS NOC |
| `0x01620000` | — | System NOC |
| `0x017a00000` | 64KB | GIC-v3 GICD |
| `0x017a60000` | 8MB | GIC-v3 GICR |
| `0x017c10000` | 4KB | 看门狗 |
| `0x017c20000` | 4KB | ARMv7 内存定时器 |
| `0x018200000` | — | RPMh RSC (apps_rsc) |
| `0x018323000` | 4KB | cpufreq_hw freq-domain0 |
| `0x018325800` | 4KB | cpufreq_hw freq-domain1 |
| `0x018327800` | 4KB | cpufreq_hw freq-domain2 |
| `0x018800000` | 8MB | ICNSS (WLAN) |
| `0x01d84000` | 12KB | UFS 控制器 |
| `0x01de0000` | 128KB | 加密引擎 (QCEDEV/QCRYPTO) |
| `0x01e00000` | 528KB | IPA |
| `0x03000000` | 256B | ADSP PIL |
| `0x033c0000` | — | LPI Pinctrl |
| `0x03d00000` | 256KB | GPU 核心 |
| `0x03d6a000` | 192KB | GPU GMU |
| `0x03d90000` | 36KB | GPUCC 时钟控制器 |
| `0x03da0000` | 64KB | kgsl_smmu (GPU SMMU) |
| `0x06000000` | — | CoreSight 组件 |
| `0x07040000–0x07740000` | — | ETM0–ETM7 |
| `0x07800000` | — | CoreSight APSS Funnel |
| `0x08300000` | 1MB | CDSP/Turing PIL |
| `0x08804000` | 4KB | SDHC2 (SD卡) |
| `0x088e0000` | 8KB | EUD (USB 调试) |
| `0x088e3000` | 272B | USB2 HS PHY |
| `0x088e8000` | 12KB | USB3 QMP DP PHY |
| `0x09160000` | — | DC NOC |
| `0x09200000` | 832KB | LLCC (L3 缓存控制器) |
| `0x09680000` | — | GEM NOC / MC Virt |
| `0x09800000` | — | NPU PIL |
| `0x09900000` | 128KB | NPU TCM |
| `0x09980000` | 64KB | NPUCC 时钟控制器 |
| `0x09990000` | — | NPU NOC |

---

> **文档结束** — 从 `picasso-r-oss` 下游设备树 (~19,500 行 DTS/DTSI 代码) 中提取
