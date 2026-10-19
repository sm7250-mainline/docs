# picasso (SM7250) 主线移植——文件级修改清单

> 生成于 2026-10-11（显示链打通之日）。基于 `linux-6.17.0.zip` 原始包与
> `sdm765_sm7250_linux-6.17.0/` 工作树的**全树 diff**（非文档回溯），
> 供移植到干净的新版内核（如 7.3）使用。
>
> 每个文件标注：**【正式】**= 功能性修改，必须移植；**【debug】**= 排障
> 打点，移植时可剥离；**【退役】**= 历史产物，不再参与构建，可跳过。

---

## A. 新增文件（内核树内手写源码）

### A1. 设备树

| 文件 | 性质 | 说明 |
|---|---|---|
| `arch/arm64/boot/dts/qcom/sm7250-xiaomi-picasso.dts` | 【正式】 | **唯一 DT 源**（迭代 24 起自包含、显式 phandle，dtc 直编）。含全部实机验证状态：memory 三段、22 条 no-map、显示/GPU/USB/串口/pinctrl/qfprom 等。注意其中 dispcc 的 sleep_clk 引用 `<0x29>`（0-cell，单 u32——带 arg 会造成条目错位，迭代 25-fix6 根因，勿"顺手补全"） |
| `arch/arm64/boot/dts/qcom/sm7250.dtsi` | 【退役】 | 迭代 0-23 的 dtsi 组织结构，已被上面自包含 dts 取代，**不被引用**。移植可跳过（如需 dtsi 组织参考，备份在 `sm7250-xiaomi-picasso.dts.dtsi-based.bak`） |

### A2. 时钟控制器（下游 R 内核移植，全部经 gcc-lito.c/dispcc-lito.c/gpucc-lito.c 对照审计）

| 文件 | 说明 |
|---|---|
| `drivers/clk/qcom/gcc-sm7250.c` | GCC（已做 vs gcc-lito.c 全量审计：59 个 kona 幽灵偏移 + 36 处投票位/hwcg 修正 + gpll7 幽灵移除 + USB reset 表 3 处修正） |
| `drivers/clk/qcom/dispcc-sm7250.c` | dispcc（45 时钟；byte/pixel parent 用 **.index** 引用——fw_name 版本在 6.17 实测解析错位，见 §B6） |
| `drivers/clk/qcom/gpucc-sm7250.c` | gpucc（13 时钟 + LUCID PLL1 + CX/GX GDSC + probe 的 CX_GMU WAKE/SLEEP=0xF，均对下游逐字审计） |
| `include/dt-bindings/clock/qcom,gcc-sm7250.h` | GCC 时钟 id（含 gcc_gpu_gpll0/161/162 等） |
| `include/dt-bindings/clock/qcom,dispcc-sm7250.h` | dispcc 时钟 id |
| `include/dt-bindings/clock/qcom,gpucc-sm7250.h` | gpucc 时钟 id + GDSC id |

### A3. 显示

| 文件 | 说明 |
|---|---|
| `drivers/gpu/drm/msm/disp/dpu1/catalog/dpu_6_1_sm7250.h` | DPU 6.1 catalog（偏移=live 实测-0x1000；**merge_3d=0x83000/0x83100**——原始移植曾误 0x3000/0x3100，迭代 25 审计修正） |
| `drivers/gpu/drm/panel/panel-novatek-nt36672c.c` | **在用的 G7A 面板驱动**（Makefile 编译 CONFIG_DRM_PANEL_NOVATEK_NT36672C；文件名 nt36672c 实为 G7A DSC video 驱动，含 DBRG 打点） |
| `drivers/gpu/drm/panel/panel-simple-g7a-37-02-0a-dsc.c` | 【退役】Makefile 无引用（早期产物） |
| `drivers/gpu/drm/panel/panel-xiaomi-g7a.c` | 【退役】同上 |

### A4. 其它驱动

| 文件 | 说明 |
|---|---|
| `drivers/pinctrl/qcom/pinctrl-sm7250.c` | TLMM（4-tile 偏移并入寄存器算式=下游 4.19 同款；函数表从 pinctrl-lito.c 整体重生；**no_get_direction + 72 脚白名单**——6.17 gpiolib 逐脚 get_direction 会撞无响应槽位） |
| `drivers/regulator/qcom-lcdb-regulator.c` | PM8150L LCD bias（avdd/avee，SPMI 孙节点 DT 直读 reg，裁剪自下游 qpnp-lcdb-regulator.c） |
| `drivers/interconnect/qcom/sm7250.c` + `sm7250.h` | qnoc（**当前 DT 全部 interconnect 节点禁用**——缺 BCM 会冲 RPMh 复位；UFS/USB 阶段修表后再启用） |
| `include/dt-bindings/interconnect/qcom,sm7250.h` | qnoc 资源 id |

### A5. 配置

| 文件 | 说明 |
|---|---|
| `.config` | 配置基准（含 §D 全部 CONFIG；移植到新内核时以此为参照 diff） |

---

## B. 修改文件（相对 linux-6.17.0 原始包，共 33 项有效 + 1 构建镜像）

### B1. 构建/注册（【正式】，每项一行改动）

| 文件 | 改动 |
|---|---|
| `arch/arm64/boot/dts/qcom/Makefile` | `+dtb-$(CONFIG_ARCH_QCOM) += sm7250-xiaomi-picasso.dtb` |
| `drivers/clk/qcom/Kconfig` | +SM_GCC_7250 / SM_DISPCC_7250 / SM_GPUCC_7250 条目 |
| `drivers/clk/qcom/Makefile` | +三个 cc 驱动 obj 行 |
| `drivers/interconnect/qcom/Kconfig` + `Makefile` | +INTERCONNECT_QCOM_SM7250 |
| `drivers/pinctrl/qcom/Kconfig.msm` + `Makefile` | +PINCTRL_SM7250 |
| `drivers/regulator/Kconfig` + `Makefile` | +REGULATOR_QCOM_LCDB |
| `drivers/gpu/drm/panel/Kconfig` + `Makefile` | +NOVATEK_NT36672C（G7A） |
| `drivers/gpu/drm/msm/disp/dpu1/dpu_hw_catalog.c` + `.h` | +`#include "catalog/dpu_6_1_sm7250.h"` / `+extern dpu_sm7250_cfg` |
| `drivers/gpu/drm/msm/msm_mdss.c` | +sm7250-mdss 匹配条目 |

### B2. 驱动功能修改（【正式】）

| 文件 | 改动 |
|---|---|
| `drivers/clk/qcom/clk-rpmh.c` | +`clk_rpmh_sm7250` 表（仅 bi_tcxo/bi_tcxo_ao——lito cmd-db 精简，缺资源是整体 -ENODEV）+ match 项 |
| `drivers/gpu/drm/msm/adreno/a6xx_catalog.c` | +`0x06020002` 条目（Adreno 620；fw=a650_sqe.fw/a650_gmu.bin、zapfw=a620_zap.mdt、speedbins {0,132,115}；与上游 diff 仅此一条） |
| `drivers/pinctrl/qcom/pinctrl-msm.c` | +`soc->init_valid_mask` 挂载（gpio 层掩码，非 girq 层——两套独立开关） |
| `drivers/pinctrl/qcom/pinctrl-msm.h` | +`init_valid_mask`/`no_get_direction` soc_data 字段（6.17 gpiolib 逐脚 get_direction 行为适配） |
| `drivers/soc/qcom/ubwc_config.c` | +`qcom,sm7250` 条目（ubwc 3.0/swizzle 6/hbb 15/macrotile——无条目 DPU probe -EINVAL） |
| `drivers/watchdog/qcom-wdt.c` | ① match 表 +`qcom,apss-wdt-sm7250/sm8250 → apcs_tmr 布局`（kpss 回退布局错误）；② probe 末尾无条件停狗（**bring-up 用**，量产前应恢复接管喂狗） |
| `drivers/gpu/drm/msm/dsi/dsi_host.c` | **DSI 显式 reparent 修复（正式部分）**：`dsi_clk_init_6g_v2()` 可选 get byte_src/dsi_pll_byte/pixel_src/dsi_pll_pixel；`dsi_link_clk_set_rate_6g()` power_on 时 reparent byte/pixel src → DSI PLL（上游 v2.9 方案下沉 v2.4；assigned-clock-parents 在 probe 时机不可靠）。⚠️ 该文件混有排障 debug，见 B4 |

### B3. 带 debug 打点的修改（【debug】——移植时建议剥离打点保留功能，或直接丢弃）

| 文件 | 内容 |
|---|---|
| `drivers/clk/clk.c` | `clk_core_fill_parent_index` 的 DBRG-CLK 条件打印（0-cell 错位排障用，**已结案可删**） |
| `drivers/gpio/gpiolib.c` | GDBG 三处打点（valid_mask 排障，fix15 遗留，可删） |
| `kernel/dma/pool.c` | atomic_pool_expand 打点（迭代 6，QTEE 保留区排障，可删） |
| `arch/arm64/mm/dma-mapping.c` | arch_dma_prep_coherent 打点（同上，可删） |
| `drivers/gpu/drm/msm/dsi/dsi_host.c` | （与 B2 同文件）reparent 失败时的 parent 表 dump —— 排障用可删 |
| `drivers/gpu/drm/msm/dsi/dsi_manager.c` | dsi bridge power_on DBRG（5 处） |
| `drivers/gpu/drm/msm/dsi/phy/dsi_phy.c` | phy enable DBRG（3 处） |
| `drivers/gpu/drm/msm/disp/dpu1/dpu_crtc.c` / `dpu_encoder.c` / `dpu_kms.c` | modeset 链 DBRG 打点 |
| `drivers/gpu/drm/msm/msm_drv.c` / `msm_kms.c` / `msm_fbdev.c` | msm_drm_init/kms_post_init/fbdev DBRG（fix20 盲区打点） |
| `drivers/gpu/drm/panel/panel-novatek-nt36672c.c`（新增文件内） | panel prepare 各阶段 DBRG —— 移植时可保留（无害）或剥离 |

### B4. 排除项（非修改）

- `scripts/dtc/include-prefixes/arm64/qcom/Makefile`：构建期展开的镜像（同 B1 的 dts Makefile），不是独立修改。

---

## C. CONFIG 清单（.config 相对 stock 的功能性开项）

```
# 串口（QCOM_GENI_SE 是硬依赖，缺它 olddefconfig 静默丢串口四件套）
CONFIG_QCOM_GENI_SE=y
CONFIG_SERIAL_QCOM_GENI=y
CONFIG_SERIAL_QCOM_GENI_CONSOLE=y
CONFIG_SERIAL_EARLYCON=y
CONFIG_CMDLINE_FORCE=y
CONFIG_CMDLINE="earlycon=qcom_geni,0x888000 console=ttyMSM0,115200 rw ignore_loglevel initcall_debug clk_ignore_unused pd_ignore_unused regulator_ignore_unused"

# RPMh / 时钟
CONFIG_QCOM_CLK_RPMH=y
CONFIG_SM_GCC_7250=y
CONFIG_SM_DISPCC_7250=y
CONFIG_SM_GPUCC_7250=y

# pinctrl / SPMI 链（PDC 必开——spmi/pinctrl/tsens 的 wakeup 域）
CONFIG_PINCTRL_SM7250=y
CONFIG_QCOM_PDC=y
CONFIG_SPMI=y
CONFIG_SPMI_MSM_PMIC_ARB=y
CONFIG_MFD_SPMI_PMIC=y
CONFIG_PINCTRL_QCOM_SPMI_PMIC=y

# 显示
CONFIG_DRM_MSM=y
CONFIG_DRM_PANEL_NOVATEK_NT36672C=y      # G7A 面板
CONFIG_REGULATOR_QCOM_LCDB=y             # PM8150L LCD bias
CONFIG_BACKLIGHT_QCOM_WLED=y
CONFIG_REGULATOR_QCOM_REFGEN=y           # DSI refgen-supply 非可选
CONFIG_NVMEM_QCOM_QFPROM=y               # fw_devlink：gpu speedbin supplier
CONFIG_INTERCONNECT_QCOM_SM7250=y        # 驱动在编（DT 节点暂禁用）

# USB gadget console（CDC-ACM → /dev/ttyGS0 → 电脑 /dev/ttyACM0）
CONFIG_USB_G_SERIAL=y
CONFIG_PHY_QCOM_USB_SNPS_FEMTO_V2=y
# CONFIG_USB_ETH 不能开（g_ether 抢 UDC）

# initramfs（内嵌；文件名必须 .cpio.gz 结尾）
CONFIG_INITRAMFS_SOURCE="<工作目录>/builds/copydown/initramfs.cpio.gz"

# 基础
CONFIG_QCOM_SCM=y  CONFIG_QCOM_SMEM=y(⚠ 曾被静默丢弃，需复核)
CONFIG_HW_RANDOM=y CONFIG_HW_RANDOM_QCOM=y(⚠ 同上)
CONFIG_POWER_RESET_MSM=y  CONFIG_HWSPINLOCK=y  CONFIG_HWSPINLOCK_QCOM=y
CONFIG_KEYBOARD_GPIO=y
```

⚠️ **已知 config 静默回归**（迭代 25 发现）：`QCOM_SMEM`、`HW_RANDOM_QCOM`
曾两度被丢——新内核移植后逐一 grep 确认。

---

## D. 内核树之外的配套文件（工作目录，非内核移植对象但构建依赖）

| 文件 | 作用 |
|---|---|
| `scripts/build-boot.sh` | 一键构建（内核+DTB+shim+打包+校验） |
| `scripts/copydown-shim.S.in` | copydown shim 汇编模板（EL2 降级/停狗/UART 探针/拷贝） |
| `scripts/mkboot-linux-copydown-picasso.sh` | shim 构建 + mkbootimg + manifest |
| `scripts/make-initramfs.sh` | initramfs 生成（busybox/musl/**GPU 固件**双路径打包） |
| `scripts/initramfs/` | busybox + musl loader |
| `scripts/initramfs/firmware/qcom/` | GPU 固件（a650_gmu.bin/a650_sqe.fw/a620_zap.mdt+b00-b02，与 YAAP ROM 逐字一致） |
| `scripts/runtime-dtb-picasso.dts` | 【退役参考】迭代 24 前的 DT 源 |
| `docs/copydown-shim-port.md` | 迭代日志（0-25，全部技术细节） |
| `docs/2026-10-11.md` | 显示链打通当日总结 |

---

## E. 移植操作建议（顺序）

1. **先 DT**：A1 的 dts + B1 的 dts Makefile 行（新内核 dts 语法/包含关系可能有
   变化，自包含结构正好降低冲突面）。
2. **时钟三件套**：A2 + B1 注册 + B2 的 clk-rpmh 表（新版内核若已有
   sm7250 支持，逐文件 diff 取舍）。
3. **pinctrl**：A4 + B2 的 pinctrl-msm.c/.h 机制改动（**注意**：新内核
   gpiolib 若已改逐脚 get_direction 行为，`no_get_direction`/白名单可能
   可以退役——先看新版 gpiolib 的注册路径）。
4. **显示链**：A3 + B1/B2 + DPU catalog + panel 驱动；dsi_host 的显式
   reparent 检查新内核是否已自带（v2.9 方案可能已普及到 v2.4）。
5. **收尾**：watchdog 停狗（bring-up 期）、LCDB、ubwc、qfprom config。
6. **debug 剥离**：B3 全部打点删除（或最后保留 panel DBRG 到稳定）。
7. **dt-binding 头**：A2/A4 的 dt-bindings 头文件随驱动一起搬。

> 移植后首轮验证基线：UART `SCK` → earlycon → `Run /init` → ttyMSM0 shell
> →（显示）`Initialized msm 1.13.0` → `Console: switching to colour frame
> buffer` → 亮屏。遗留项见 docs/2026-10-11.md §8（sqe fw -2 等）。


---

## F. 已完成：同步到 linux-7.3-rc6（2026-10-11）

7.3 同步已完成并通过构建（Image 44.8MB + dtb + boot.img manifest 全过；
`builds/boot-7.3-rc6-picasso.img`，sha256 `0d6006f59dfc494970946e4945205da94b58690c79c4874bef5a4ea9915aa231`）。
上游 7.3 无任何 sm7250 支持（预检零命中），全部修改按下列适配重放：

### API 演进适配（6.17 → 7.3 结构差异）

| 文件 | 7.3 变化 | 适配 |
|---|---|---|
| a6xx_catalog.c | `.init` → `.funcs = &a6xx_gpu_funcs`；新增 `.gbif_cx` 字段 | 0x06020002 条目按新结构重写（gbif 照同家族 A621 用 a640_gbif） |
| dpu_6_1_sm7250.h | `vbif_count/vbif[]` → `.vbif = &sdm845_vbif` 单指针；`VBIF_RT`/`vbif_idx` 移除 | 删 wb 的 .vbif_idx、cfg 尾部改单指针 |
| ubwc_config.c | 结构重构为预设组合（`ubwc_3_0_hbb15` 等），swizzle/macrotile 按版本推导 | 用 `ubwc_3_0_hbb15`（推导结果 swizzle=LVL2|LVL3=6、hbb15 与下游 lito 实测精确等价；macrotile 上游推导 3.0=4ch，下游 dtsi 写 8ch——当前 linear fb 不依赖，已在案） |
| interconnect/qcom | `qcom_icc_node` 大改：无 `.id`、`.links{id}` → `.link_nodes{&var}`；node 数组改顺序式（per-provider id 直接索引） | 自动转换器重写 sm7250.c（132 node、多行 links、[MACRO] 索引数组→排序顺序数组）；本地 sm7250.h 弃用（dt-binding 头即够） |
| dsi_host.c | v2_9 显式 reparent 机制仍在（未下沉 v2.4） | 显式 reparent + 可选 clk 四件套按 7.3 函数体重放（无 debug dump） |
| pinctrl-msm.c/.h | 结构兼容（init_valid_mask 锚点仍在） | no_get_direction + init_valid_mask 机制原样重放（不带 PDBG 打点） |
| 其它 Kconfig/Makefile | 全部有漂移 | 逐一找锚点插入（同 6.17 内容） |

### .config 生成方式

`make defconfig` + scripts/config 批量开 §C 清单 + olddefconfig。
注意链式归 m 陷阱：`DRM`/`DRM_PANEL`/`BACKLIGHT_CLASS_DEVICE` 需先扳 y，
`DRM_MSM` 才会 =y（且自动带出 DSI/7nm PHY 全子项）。构建命令：
`PATH=/tmp/lld-extract/usr/lib/llvm-21/bin:$PATH make ARCH=arm64 LLVM=1 -j16 Image dtbs`
（7.3 强制 `-fuse-ld=lld`，GNU ld 回退不可用——必须提取 lld）。

### 与 6.17 树的差异（有意为之）

- **debug 打点全部未移植**（B3 清单 12 项：DBRG/GDBG/PDBG/clk.c dump 等）——
  7.3 保持干净；panel 驱动文件内自带的 DBRG 除外（原样复制的文件）；
- `sm7250.dtsi`/未编译的两个 G7A panel 退役文件未复制；
- watchdog 停狗注释按 7.3 语义重写（内容等价）。

### 状态与验证

- ✅ 构建：Image + dtbs 零错误；boot.img manifest 全部 True
- ⏳ **实机验证未做**——预期首轮暴露 7.3 binding/行为差异（参考 6.17 经验：
  fw_devlink、dts 细节、SMEM 等），按迭代日志方法排障
- 6.17 基线：`boot-linux-copydown-picasso.img` 已恢复（sha `0856bf87…`，
  lld 链接版；与亮屏版 `0c645147…`（GNU ld）功能等价，差异见手册 §3.2）


### 补记：7.3 首刷实机验证 ✅ + config 精简（同日）

**实机（0d6006f5 全量版）**：一次点亮——shell/屏幕 fbcon/USB ttyACM0 全部
正常，`Initialized msm 1.13.0`，全部曾 -517 设备最终 probe 0（含 7.3 上游
**已支持 sm7250 的 ARM_QCOM_CPUFREQ_HW**）。启动 /init 用时 27.3s。

**config 精简**（以 6.17 实机验证集为基底迁移 + 下游 4.19 config.gz 交叉
核对）：Image **44.8MB → 15.1MB（-66%）**；同时修复两处 6.17 遗留静默回归
（QCOM_SMEM/HW_RANDOM_QCOM）、开 cpufreq-hw/AOSS/GENI I2C（触摸预备）；
去掉 cmdline 的 initcall_debug/ignore_loglevel（UART 输出是启动耗时大头）。
产物 `builds/boot-7.3-rc6-slim-picasso.img`（4d29800d…）。

**7.3 日志遗留问题清单**：
1. `a650_sqe.fw` 加载 -2（6.17 同款谜题未解，GPU 渲染路径缺微码；显示
   DPU 软件路径不受影响）——待实机 shell `ls /lib/firmware/qcom/` 定案；
2. psci `failed to set PC mode: -3` Firmware Bug——无害（TZ 拒绝，常见）；
3. WLED IRQ short/ovp -ENXIO——已知无害（无中断接线的警告）；
4. cpufreq-dt 抢占报错已随 CPUFREQ_DT=n 消失（精简版应干净）。


### 补记 2：PRINTK_TIME + CPU OPP 表修复（同日晚）

**CPU OPP "Voltage update failed"×23 根因**：cpufreq-hw 平台（EPSS/LUT）在
DT 提供 opp 表时走 icc_scaling 路径——把硬件 LUT 档位逐个
`dev_pm_opp_adjust_voltage` 写回 opp 表，**DT 表外的档位全部失败且被丢弃**
（我们每簇仅列 7 档，LUT 有 20+）。**修复 = 删除 DT 的三个 cpu opp 表 +
cpu 节点引用**（上游 sm8250 同款做法）：走纯 LUT 路径 → 全档位可用 +
`fast_switch` 打开。CPU 节点的 `qcom,freq-domain` 保留（cpufreq-hw 域绑定）。

**PRINTK_TIME=y**（精简迁移时基底为 n，日志无时间戳无法量化启动耗时）。

**构建环境**：`/tmp/lld-extract` 再被系统清理（同一坑两次）→ lld-21 固化
`tools/lld-21`，build-boot.sh 的 LLD_DIR 指向持久目录；**7.x 下 GNU ld 回退
删除**（HOSTCC 也强制 lld，回退无效）。期间发现 Ubuntu clang 找不到
ld.lld 时报误导性文案 "invalid linker name in argument '-fuse-ld=lld'"
（实为找不到 ld.lld 可执行文件）。

产物：`builds/boot-7.3-rc6-slim-picasso.img`（475ec6b5…）。


### 补记 3：工具链规则修正（用户指示，最终版）

**规则：我方控制的编译环节一律用 ARM 官方裸机工具链（aarch64-none-elf-*，
含 aarch64-none-elf-ld 做目标端内核链接）；Linux 内核自身的默认编译选项
（如 LLVM=1 下 HOSTCC 强制 -fuse-ld=lld）不对抗。** 脚本实现：
`KERNEL_LD_OVERRIDE="LD=$TC/aarch64-none-elf-ld"` 固定；tools/lld-21 的
ld.lld 存在时仅 export 进 PATH 服务内核默认 HOSTCC。验证：7.3 以该组合
构建零错误（Image 17.2MB，GNU ld RELR 取消较 lld 版 +2MB 属预期）。
产物更新：`builds/boot-7.3-rc6-slim-picasso.img`（799f3df3…，含
PRINTK_TIME + CPU opp 表移除 + GNU ld）。


### 补记 4：GNU ld 的 RELR 恢复（修正历史误判）

迭代 11 的"GNU ld 无 RELR 打包、+1.2MB 属预期"是**探测脚本假象**：
`scripts/tools-support-relr.sh` 的 `$CC -c -x c` 裸调在 LLVM=1（CC=clang，
默认 x86 target）+ GNU aarch64 LD 混合链下产出 x86 .o → 目标 ld 拒绝 →
TOOLS_SUPPORT_RELR=n。实际 GNU ld 2.45 支持 `-z pack-relative-relocs`
（.relr.dyn 实测生成）。**两树已修探测脚本**（.o 加
`--target=aarch64-linux-gnu`）→ RELR=y 恢复。成果：GNU ld 版与 lld 版
Image 持平（7.3: 15,478,792 vs 15,085,576；6.17: 14,516,232 vs 14,503,944），
"大 2MB"不复存在。产物：`builds/boot-7.3-rc6-slim-picasso.img`（38f378e5…）。
