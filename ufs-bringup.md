# picasso（SM7250/lito）UFS bring-up 全记录（进行中）

> **2026-10-16 最新状态：P41 实机验证成功，UFS READ(10) 打通。**
> L7C/L5A/L9A 固定 HPM、qphy.lvl 独立投票后，6 个 LUN 的 4KB READ、
> 分区扫描和数据读回全部恢复；`Run /init` 提前到 2.18s。HPM 与 QPHY
> 的单独贡献尚未回退定位；P41 及迭代 29 调试/测试代码已清理并完成
> 本地完整构建，清理版镜像待实机回归。实机证据见
> [P41 记录](2026-10-16-ufs-power-fix.md)。
>
> 以下为历史排查记录；其中已被 P41 纠正的推断不作为当前结论。

> 2026-10-13 起步，迭代 29。当前状态（2026-10-14 深夜）：**dev cmd / 设备识别 /
> 块设备枚举全部打通（sda 62.8GB + 5 个小 LUN），数据 READ(10) ≥4KB 楔死未解**。
> fix11~fix19 的假说判别把并发/干扰/doorbell/数据地址/I 位/PHY 全部证伪（§4/§5），
> 期间的自测脚手架经全面审计后于 **fix20（620077f3）彻底清除**——现为干净基线
> （主线 7.3-rc6 + 平台移植 + 两个实证修复），权威源逐属性对齐确认完毕。
> 本文记录完整技术路径：驱动移植、逐轮刷机、真根因与伪线索的分辨过程。
> 配套逐轮实录见 `copydown-shim-port.md` 迭代 29；显示/GPU/触摸另见
> [display-bringup.md](display-bringup.md) / [touch-bringup.md](touch-bringup.md)。

---

## 目录

1. [硬件与软件架构](#1-硬件与软件架构)
2. [移植内容（代码与 DT）](#2-移植内容)
3. [排查历程：四个真根因](#3-排查历程四个真根因)
4. [伪线索清单（浪费过轮次的）](#4-伪线索清单)
5. [当前卡点：READ(10) 数据读挂死](#5-当前卡点)
6. [方法论与教训](#6-方法论与教训)
7. [产物 sha256](#7-产物-sha256)
8. [2026-10-15 攻坚日志：READ(10) 判别框架收敛（37→39）](#8-2026-10-15-攻坚日志)

---

## 1. 硬件与软件架构

```
ufshc@1d84000（UFSHCI，GIC 265 level）──────── SKhynix H9HQ53AECMMDAR
  ├ 时钟: gcc[60-70]：AXI/AGGRE/AHB/UNIPRO/ICE/TX_SYMBOL0/RX_SYMBOL0,1/
  │        1X_CLKREF(0x8c000)/PHY_AUX —— 下游 gcc-lito 偏移逐字核对一致
  ├ GDSC: ufs_phy_gdsc(0x77004)，BCR: GCC_UFS_PHY_BCR(0x77000)
  ├ 复位: phy 侧经 ufshc 的 reset-controller（#reset-cells）
  └ DMA:  **不接 apps_smmu**（下游与 live_dts 均无 iommus；32 位寻址）

phy@1d87000（QMP UFS PHY v4-lito 变体）
  ├ 布局: COM=0x000 / TX(n)=0x400+0x400n / RX(n)=0x600+0x400n / PCS=0xC00
  ├ 电源: vdda-phy=PM8150 l5a 0.88V，vdda-pll=l9a 1.2V
  └ 时钟: ref_clk_src=rpmhcc bi_tcxo，ref_clk=gcc68，ref_aux=gcc69
```

**lito PHY 的两个关键差异**（对照 sm8150/sm8250）：
- TX 尾部：`TRAN_DRVR_EMP_EN`@0xC0（非 0xB8）、PWM 分频器@0x168+（非 0xD8+）
- RX 无 `RX_PWM_*_TIMEOUT` 段：`RX_MODE_00` 起 0x15c（比 sm8150 低 0x14），
  `DCC_CTRL1`=0x1A8（非 0x1BC）
- serdes 24 项与 sm8150 **逐值相同**；rate-B 表（VCO_TUNE_MAP=0x06）亦同

主线 7.3 的 UFS 栈在 `drivers/ufs/{core,host}/`，ufshcd-qcom 绑 `qcom,ufshc`；
dev cmd 走**伪设备保留请求**（`pseudo_sdev` id=max_id、lun=U64_MAX、
CDB 全零、tag=0）——排查时勿被这个形态吓到，它是正常的。

---

## 2. 移植内容

| 文件 | 改动 |
|---|---|
| `drivers/phy/qualcomm/phy-qcom-qmp-ufs.c` | +`sm7250_ufsphy_cfg`（lanes=2、v4 布局、max gear=3）；+`sm7250_ufsphy_{tx,rx,pcs}` 表（下游 v4-lito 值，lito 专属偏移用 `SM7250_*` 本地宏）；serdes/hs_b 复用 sm8150（逐值同）；+`sm7250_ufsphy_vreg_l`（90192/19000µA live 实值）；+of_match `qcom,sm7250-qmp-ufs-phy` |
| `drivers/phy/qualcomm/phy-qcom-qmp-pcs-ufs-v4.h` | +`QPHY_V4_PCS_UFS_RX_SIGDET_CTRL1 0x154` |
| `drivers/clk/qcom/gcc-sm7250.c` + 头文件 | +`gcc_ufs_phy_phy_aux_clk_src` RCG（0x770a0，父 bi_tcxo，19.2MHz 单频表，本树编号 111）+ aux 分支挂父（原先无父、靠 ABL 残留） |
| `arch/.../sm7250-xiaomi-picasso.dts` | ufshc/phy 节点启用；freq-table-hz 8 对（下游 dtsi 实值）；`dev-ref-clk-freq=<0>`；`limit-hs-gear=<3>`；**无 iommus、无 reset-gpios**（见 §3） |
| `.config` | 零新增（UFSHCD/UFS_QCOM/QMP_UFS/SD 已全 =y） |

---

## 3. 排查历程：四个真根因

按发现顺序。每个都经过"插桩实锤 → 对照下游/权威源 → 修复"闭环。

### 根因 ①：`freq-table-hz` 缺失 → 驱动零时钟管理（92ab3382 → 954841b9）

首刷 PHY 链一次通过（PCS_READY、UIC 全通），但
`UNIPRO clk freq 0 MHz not supported` → LINK_STARTUP(-110) ×3。
读码：`ufshcd_parse_clock_info` 在无 `freq-table-hz` 时**直接 goto out，
一条时钟都不进 clk_list**——驱动从未 devm_clk_get/启用任何时钟，
此前能跑全靠 XBL/ABL 残留态。live dts 本带此表，组 DT 时漏了。
**修复**：补 8 对（core 50-200M、unipro 37.5-150M、余 0；ICE 两对删）。

### 根因 ②：`reset-gpios = <tlmm 46>` 错误引脚（fix3 对照审计发现）

下游工厂 dtsi 的 UFS 设备复位走 TLMM **专用 `ufs_reset` pin**
（下游编号 153@0x5bc000 = 本树 146@0xbc000），**gpio46 是 qup13/qdss_cti
普通引脚**。MIUI dump 把它写成 reset-gpios，主线 gpiod 从 boot 起驱动
无关信号。**修复**：删除该属性（上游 sm8250 同样不带；真复位 pad 维持
ABL 去断言态）。教训：**runtime dump 的属性 ≠ 事实**。

### 根因 ③：`iommus` 属性整个不该有（fix6，breakthrough 前夜）

对照发现：**下游 dtsi 与用户提供的 live_dts.dts 里 ufshc 都没有 iommus**
（Android 以 SMMU 旁路跑 UFS）。带 iommus 时 `dmam_alloc_coherent` 给出
IOVA（0xffff9d000），而 sm7250 的 UFS 抓取端口不经 apps_smmu 翻译 →
描述符永远取不到。三个 sid 值（0xe0/0x4e0/0x80）现象完全相同的原理：
翻译从未介入。**修复**：删除 iommus，走 dma-direct/物理地址。

### 根因 ④：DMA 掩码谎报 64 位 → 分配落 4GB+（fix7，dev cmd 打通）

`rq-post: DB=0x1`（doorbell 锁存）但零完成中断；`utrdl_dma=0x100fa1000`
在 4GB 以上。CAP 的 64AS 位不实——控制器抓取只支持 32 位物理寻址。
Android 从未踩到：其分配从不落 4GB+。**修复**：`ufs_qcom_set_dma_mask`
强制 `DMA_BIT_MASK(32)`（vops `.set_dma_mask`，本钩子仅 renesas 用过）。

**打通时刻**（78be31bb）：
```
utrdl_dma=0x8715e000 → DB=0x1 → irq status=0x1（首次 UTP 完成）
→ devcmd done ocs=0 → scsi 0:0:0:49488 Well-known LUN SKhynix...
→ Run /init（2.19s）→ shell
```

### 根因 ⑤（部分）：时钟门控唤醒失步（fix9）

WLU 加完后空闲 → `UFSHCD_CAP_HIBERN8_WITH_CLK_GATING` 生效（h8+关时钟；
旧轮次 clk_list 为空门控无牙，根因①修复后第一次真的关时钟）→
唤醒时 `HW_H8_ENTER=2/EXIT=1` 失步 → 此后全部 `ocs=0xf`。
**修复**：bring-up 移除 CLK_GATING/HIBERN8_WITH_CLK_GATING 两个 cap
（副作用：LUN0 扫描恢复，sda+5 LUN 进 /proc/partitions）。

---

## 4. 伪线索清单

排查中投入过轮次、最终排除的方向——记录以免重蹈：

| 伪线索 | 排除依据 |
|---|---|
| 中断位图错位（0x400≠UIC_COMPL） | 本树与下游位图**逐字一致**（qcom 变体布局） |
| 寄存器偏移错位 | doorbell 0x58 / AGG 0x4c / IE 0x24 与下游逐字一致 |
| 中断聚合扣住完成 | 两边 caps 都无 `UFSHCD_CAP_INTR_AGGR`（禁用） |
| `setup_xfer_req` 钩子 | exynos 专属，qcom 无 |
| PMC 状态位卡 1 = 切换失败 | `PWR_LOCAL` 恰为 1——UPMCRS 检查实际存在且通过 |
| HS/PHY 表错（PWM 强制实验） | PWM 下 READ 同挂——HS 无罪 |
| sid 0x4e0/0x80 猜测链 | 见根因③——sid 无关紧要，属性本不该有 |
| dev cmd "神秘命令"（op=0/id=1/lun=-1） | 主线伪设备保留请求的正常形态 |
| `scsi_mod.scan=manual` 拦截失败 | WLU 注册走 `__scsi_add_device` 绕过 scan_type（但该参数确实救活过 boot） |
| 多命令并发在途（fix11 前） | outstanding=0x2 单命令在途仍挂 |
| 早期显示/GPU bring-up 风暴干扰 | probe 延迟 5s 后 READ 仍挂（fix13） |
| doorbell 写丢失 | 重写对已锁存位是 no-op，且新 tag 复刻命令照常完成（fix14~16） |
| 数据地址/bank 筛选（swiotlb 槽、bank1/bank2） | 重定向 bank1 仍挂 + bank2 探针无数据（fix19） |
| XBL 陈旧 SMMU 翻译 | apps_smmu 全量复位 "preserved 0"（fix13 log） |
| SYS1CLK/hw_ver 分支未执行 | vendor dump 实证已编程且 hw4（fix13） |
| TRD I 位（intr_cmd） | I=1 的同型自测命令仍零数据（fix17 崩溃前 T0） |
| "irq 打印消失=中断停了" | 插桩计数器共用截断（26+14=40 恰上限；fix13 起拆分） |

---

## 5. 当前卡点：READ(10) 数据读挂死

**现象**（c5ba9476 之后稳定复现）：
- 扫描期命令（INQUIRY/ModeSense/READ CAPACITY，串行、小数据）**全通**
  ——1.78-1.86s 六个 LUN 完整识别
- **块层 READ(10)（分区表读取）从第一 sector 起全部 30s 超时** →
  EH → 6 盘 offline；fix11 轮起 EH 后 tag0 abort 风暴卡死
  async_synchronize_full → **连 shell 都到不了**
- 挂死现场（fix11 = 41e94969，dump 齐全轮）：
  **outstanding=0x2——在途的只有这一条孤零零的 READ(10)**；
  TRD/UPIU/PRDT 逐字段解码**全部正确**（CT=UFS_STORAGE、DD=DEV_TO_HOST、
  无 crypto、exp_data_len=4096、CDB=READ LBA0 len1、PRDT 1 条 →
  phys 0xfd83a000 = swiotlb 窗口内弹回槽）；**OCS=0xF（驱动初值，
  控制器从未改写）** + 响应 UPIU 全零 + 30s 零完成中断
  = **控制器压根没执行这条命令**
- 事后：TM abort 超时 ×2、host reset 后 link startup 连败 5 次
  （控制器或设备深度楔死）

**初步证伪**（详见下方证伪总表）：多命令并发（fix11 单命令在途仍挂，
can_queue 实验失效）、HS/PHY（PWM 同挂）、时钟门控/h8、UTRD/PRDT 构造
（逐字段正确）、doorbell 接受（锁存 ✔）。另破一处观察假象："irq 打印
消失=中断停了"实为**插桩计数器共用截断**（qc-enter 与 dev cmd 的 rq
打印共用 40 上限计数器，26+14 恰打满；READ CAPACITY/MODE SENSE 其实
都正常走了 queuecommand）。

**证伪总表（fix11~fix19，2026-10-14 全天判别）**：

| 假说 | 判别实验 | 结论 |
|---|---|---|
| 多命令并发在途 | fix11：outstanding=0x2 单命令仍挂 | ✗ 证伪 |
| 早期显示/GPU 风暴干扰（WA） | fix13：probe 延迟 5s 仍挂 | ✗ 证伪 |
| doorbell 写丢失（H4） | fix13 救援无效——但 doorbell 重写对已锁存位是 no-op，判别本身有洞 | ✗ 存疑作废（实验无效） |
| 数据地址毒性（H2） | fix19：重定向 PRD 到 bank1（0x8a000000）的块层 READ 仍楔死 | ✗ 证伪 |
| bank1/bank2 数据筛选 | fix19：bank2 确定性探针（0xf0000000）同样无数据 | ✗ 证伪 |
| XBL 陈旧 SMR/CB 翻译毒化 | fix13：apps_smmu 完整复位（preserved 0） | ✗ 证伪 |
| SYS1CLK_1US 未编程 | fix13：vendor dump=200 已编程；HW_VERSION=0x4002_0000（hw4，主线分支实际执行） | ✗ 证伪 |
| TRD I 位差异 | fix17 崩溃前 T0（bank1+I=1+512B）仍零数据 | ✗ 证伪（数据缺失维度） |
| tag1/slot0 特异性 | fix14~16 自测 tag1 与新 tag 同型完成 | ✗ 证伪（自测语境内） |
| HS/PHY、时钟门控/h8、UTRD/PRDT 构造、swiotlb | 见 §4 与 fix11 dump（逐字段正确） | ✗ 此前已证伪 |

**三源对照审计（fix13，用户指示：下游 4.19 + live_dts + sm8250/sm6350/sm7225）**：

| 项 | 下游 4.19/live | 上游参考平台 | 我们 | 判读 |
|---|---|---|---|---|
| pwr_change POST_CHANGE | cfg_timers + **phy_set_tx_lane_enable** + bus vote | — | 无 tx_lane_enable、无 bus vote | tx_lane_enable 候补（host TX 侧，READ 挂在 RX 方向，嫌疑低） |
| 带宽投票 | msm-bus 22 档（含 CNOC IB 说明注释） | **interconnects 两条路径（ufs-ddr/cpu-ufs）** | 无（qnoc 迭代 19 禁） | **唯一系统性缺口**（上游三平台全带） |
| opp 表 | 无（clk scaling 走 freq-table） | operating-points-v2 | 无（freq-table） | 等价，非差异 |
| SYS1CLK/定时器 | qunipro 下仅 SYS1CLK（aggr 允许时写） | 同 | hw4 → 写了 | ✅ 等价（vendor dump 实证 200） |
| iommus | 无（live/工厂 dtsi） | sm8250/sm6350 有 | 无（fix6 实证旁路） | ✅ 正确（die 差异） |
| caps | CLK_GATING+H8_WITH_GATING+CLK_SCALING 等 | — | 前两个已禁（fix9） | 禁特性不应引入挂死，低嫌疑 |
| dev-reset pinctrl | live 有 assert/deassert 两态 | — | 无 | 已知缺口（恢复路径，不影响首挂） |
| RTT/adapt | 无特殊 | 无 | 无 | ✅ 等价 |

**自测实验始末（fix14~fix19，全部结论作废但教训保留）**：
- 手工构造 UTRD/UPIU/PRDT 的受控自测（关 IE、独占 tag、纯轮询）经历了
  4 个自伤 bug（偏移 dword 单位、钩子位置覆盖在途命令、ioremap 对齐 Oops、
  PWM 强制遗留）后才跑通形态——最终发现**基线本身污染**：自测位于
  `ufshcd_add_lus()` 之前，LU 未激活，"完成但无数据"不能说明任何问题；
- 唯一 surviving 的实验事实：**重定向到 bank1 的块层 READ 仍楔死**（PRD
  目标地址不是楔死条件）+ **bank2 探针也无数据**（bank 筛选不存在）；
- 完整 bug 总账与教训见 §6。

**fix20（= 620077f3）实判（2026-10-14 深夜）**：READ 楔死原样复现（多条
tag 并发在途全 30s 超时→abort/TM 风暴→offline；36.8s 慢速到 init，EH
未死锁）；另见 `ufshcd_wl_runtime_suspend failed: -16`（WLU 挂起失败，
楔死后的次生现象）。**原生 bug 确认，基线干净。**

**fix20 构成——用户指令：彻底清除全部
推测/测试代码，按下游内核与 live_dts 重新确认**：

清除（git 基线逐块分类）：
- `drivers/ufs/core/ufshcd.c`（相对初始提交的 99 行 = 纯插桩零真修复）、
  `drivers/scsi/scsi_error.c`（sto 打印）、`drivers/iommu/.../arm-smmu.c`
  （SMR 转储打印）**整体还原初始提交**——后两处藏在 HEAD 提交里；
- 探针 DT 区（0x8a000000/0xf0000000）、双探针分配、redirect、自测、
  SND/irq 时间线全部移除；build-boot.sh no-map 护栏还原 22。

保留（ufs-qcom.c 相对初始的全部 18 行，有实证）：
1. `ufs_qcom_set_dma_mask` 强制 32 位（CAP 64AS 谎报，fix7）；
2. CLK_GATING/HIBERN8_WITH_CLK_GATING caps 移除（唤醒失步，fix9）。

**权威源对齐确认（下游工厂 lito.dtsi + live_dts 逐属性）**：

| 属性 | live_dts | 下游 | 我们 | 判定 |
|---|---|---|---|---|
| reg/interrupts/lanes/dev-ref-clk-freq | 一致 | 一致 | 一致 | ✓ |
| freq-table 8 对(去 ICE)+时钟表同步 | 10 对含 ICE | 10 对 | 8 对 | ✓ 等价（无 crypto） |
| vdd-hba | fixed supply | 同 | GDSC power-domains | ✓ 主线式等价 |
| iommus/reset-gpios | 均无 | 均无 | 均无 | ✓（fix6/fix3 实证） |
| limit-hs-gear=3 / vcc 800mA | — | — | ✓ | ✓ |
| PHY v4-lito 偏移抽查（TX 0xC0/RX_MODE 0x15c/DCC 0x1A8/值 0x0C） | — | 逐值 | 一致 | ✓ |
| gcc UFS 时钟 60-70 偏移 | — | gcc-lito | 一致 | ✓（迭代 20 全量审计） |
| dev-reset pinctrl | 有 | 有 | **无** | 已知缺口（恢复路径） |
| msm-bus 向量 | 有 | 有 | **无**（qnoc 禁） | 已知缺口（上游=interconnects） |

**迭代 30 实判（50effd35，2026-10-14 凌晨实机）**：**启动到 1.779398s
整机死机重启**——UART 字符流中途损坏（fabric 断供指纹），死在第 6 个 LUN
attach 中途、**分区扫描之前，READ(10) 判据本轮未测到**。根因不在 UFS：
`icc_sync_state` 全局屏障在 11 个 provider 全部 sync 后把所有节点的
init(INT_MAX) 票清零，非 ufshc 路径的 BCM（keepalive=false 居多）被
valid=0 显式禁用 → MDSS 扫描/gemnoc/snoc 瞬间断供 → CPU 卡死在下一个
总线访问 → 硬看门狗复位。上游设备不死是因为每个 provider 都有大量真实
消费者保底，我们只有 ufshc 一个。完整证据链见 copydown-shim-port.md
迭代 30 实判节。

**迭代 31 实判（1f44d304，2026-10-14 晚）**：✅ 死机消失、存活到 init
（36.77s）——sync_state 屏障因果确认（无 barrier 打印 = 修复指纹）；
ICCD 90 行实证 ufshc 投票链健康（注册票 init 钳制）。**READ(10) 楔死
原样复现**：6 LUN 全枚举（sdf 28MB LUN5 完整现身），分区 READ 全 30s
超时→EH→offline。**ICC 带宽假说正式证伪**（票到位仍挂）。三源对照的
"唯一系统性缺口"补齐后问题依旧——楔死根因在更深处。

**迭代 32（fd2aca15，待实机）：下游审计收口 + dev-reset pinctrl**。

审计排除清单更新（全案底）：
- **tx_lane_enable**：下游 v4-lito PHY 是**空函数**（"v4 PHY does not
  have TX_LANE_ENABLE register"）——最后候补出局；
- **ctrl_rx_linecfg**：仅 unipro 1.41，本机 2.x 不适用；
- **sec_cfg**：下游 `return 0` 桩；
- **probe 期 device reset**：下游也不做（detect_device 只在恢复路径；
  Android 证明 XBL 残留态可正常工作）；
- **LCC/hw_clk_gating/cfg_timers/QUNIPRO_SEL**：主线 7.3 均有等价实现；
- **freq-table ref/tx0**：~~live 75M-300M vs 我们 0-0 = 有意偏差~~
  **误读更正（迭代 37/38）**：live 的 ref/tx0 本就是 0-0——曾把 ICE 两对
  （live 第 5、6 对，75M-300M）错位记到 ref/tx 头上。自迭代 38 补回
  ICE 时钟对（含 gcc RCG 源移植）后，我方 freq-table 与 live **逐对
  完全一致**，无任何偏差。

补齐：`ufs-qcom.c/.h` device reset 增加 **pinctrl 两态回退**（gpiod 缺席
时按名取 "dev-reset-assert"/"deassert"；ufs_reset 特殊组不在 gpio 范围，
gpiod 够不着——下游 4.19 core 正是 pinctrl 语义）；DT 加两态（0x6f/0x70，
live 实值）。**pinctrl-msm 零改动**（7.3 的 output-high/low 已映射
PIN_CONFIG_LEVEL，原生支持；曾误判需补框架、编译打脸后核实）。

**迭代 32 实判（fd2aca15）**：❌ **UFS 全灭回归**——probe 死于
`link startup failed 1`（DME 连败 ~450ms）→ -EIO。根因：hw4 被
`ufs_qcom_advertise_quirks` 打上 `UFSHCD_QUIRK_REINIT_AFTER_MAX_GEAR_SWITCH`
→ `ufshcd_probe_hba` 在链路训练前 vops 脉冲 device reset——≤31 无 gpiod
时该 vops = 空操作，32 接通 pinctrl 后变真 ufs_reset 脉冲 → 设备冷复位、
固件启动超窗口。下游从不在 probe 期复位（只在 EH）——Android 未踩过。
**意外收获：两态物理生效实证**（select 零报错、设备实际被复位）。

**迭代 33（968fc6c6，未单刷）**：`ufs_qcom_init` 清 REINIT quirk（照
上游 `ufs_qcom_fw_managed_init` 既有清除模式）——probe 期不再脉冲，
回到 ≤31 行为；dev-reset pinctrl 接线保留给 EH/恢复路径。

**迭代 34 实判（260380a6，2026-10-15）**：❌ 仍死于 `link startup failed 1`
（2.142s，与 32 签名一致；33 从未单刷）。本轮排除性排查：
- linecfg 偏移**写对了**（下游 `PHY_OFF=0xC00+x`，主线 pcs 基址同为
  +0xC00 → 绝对 0xD48；且主线头文件本就有
  QPHY_V4_PCS_UFS_LINECFG_DISABLE=0x148——自建宏已清理）；
- quirk 清除确认在 `ufs_qcom_init` 生效位；主线 `ufshcd_link_startup`
  的 notify PRE 返回值**被忽略**（5348 行）→ rmw 失败也不会中止训练，
  失败来自 DME_LINK_STARTUP 本身；
- pinctrl core 全链核读（bind_pins/create_pinctrl）无任何自动应用路径；
- **sm7225/sm6350 上游参考**（sm7225.dtsi 即 include sm6350）：该代 hw4
  走通用 `qcom,ufshc` 绑定（无 drvdata）上游可工作 → 我们的绑定无缺；
  差异均为 die 级已处理项（OPP 表↔freq-table 等价、QLINK ref、clk_virt、
  sid 0x80）——**上游表亲无可移植缺口**。

**迭代 35 实判（933a694b）**：仍 link startup 失败 → **34 两补齐无罪、
32 的 pinctrl 接线实锤**。追凶收官：**真凶 = 主线 `ufshcd_init` 的
无条件设备复位**（ufshcd.c:11308，在 `ufshcd_hba_enable` 之前调用
`ufshcd_device_reset(hba)`）——≤31 该 vops 是 -EOPNOTSUPP 空操作无害；
32 接通 pinctrl 后变成真脉冲，设备固件未及重启 → DME 连败。此前排查
漏掉此调用点的教训：**grep 调用点时 `head` 截断**（列表第 6 项被切掉），
错怪了 probe_hba 的 REINIT 块（33 的 quirk 清除因此成为对症但无效的
修复）。冷启动无效的原因：每次 probe 都重新脉冲。

**迭代 36 实判（46206aff，2026-10-15）**：✅ **回归完全修复**——6 LUN
全枚举、sda 容量正常、36.9s 到 init（与 31 形态一致）。`link_started`
门生效（probe 期复位回到 ≤31 行为），34 两补齐（linecfg +
saveconfigtime）同时在场且无罪。❌ **READ(10) 楔死原样**（tag26 0x28
超时风暴 → EH → 六盘 offline）——**启动序列 + DT 属性两方向正式穷尽**。

EH 细节（新数据）：READ 楔死后 EH 走 abort（-110，"cmd pending in the
device"）→ 控制器对挂死 tag 报 **OCS invalid** → `reset_and_restore`
循环 ×5（每轮 device reset 脉冲[gate 已放行] + host reset）→ **link
startup 全败** → HBA state error → 离线。**楔死深到设备复位+主机复位
连环都救不回**——READ 根因的候选空间进一步收窄到"控制器在位/命令
通路被首条 READ 卡死后自锁"。

**下一阶段（READ 攻坚，既定方向）**：运行态仪器化——首条挂死 READ 与
同窗口扫描命令的 UTRD/UPIU/PRDT 逐 dword 对比 + host_regs 在途位反解
（0x60=UTRLDBR=1、0x64、0x94=0x15710000 等待解）+ OCS invalid 事后
报告的机理。

**迭代 37 桌面判读（2026-10-15，先于刷机完成的分析）**：36 的 EH dump
离线解码完成，结论改变了战场设定：

1. **挂死命令完全自洽且形态 = 分区扫描 4KB 读**。CDB 字节序纠错后：
   `28 00…[7]=00 [8]=01 [9]=00` = READ(10) LBA0 **×1 块**（4KB），
   EDTL=0x1000、PRDT 1 项（0xfe807000/4096B）、UTRD DD=DEVICE_TO_HOST
   （byte3=0x15 = interrupt:1+dd:2+ct:1）、响应 UPIU 全零。**软件提交
   面零瑕疵**——此前"1MB READ/PRDT=1 项异常"是本人解码错误（CDB BE16
   读反 + hexdump group 字节序两处），已在本次修正。
2. **host_regs 全量反解**（ufshci.h 映射 + LE 校准）：CAP=0x1587031f
   （nutrs=32、auto-hib 支持、**64AS=1、CRYPTO=1**）、HCS=0x10f 各 list
   ready、HCE=1、UTRLRSR/UTMRLRSR=1、**全部 UIC 错误码=0、IS=0**、
   UTRLDBR=0xa5401000（恰为 6 个挂死 tag）、AHIT=0xc96（15s，H8 计数
   0=从未触发）、**UTRLBA=0x8715e000/UTMRLBA=0x8715f000 有效**。
   旧判读修正：**0x60=UTRLRSR（运行位）不是 doorbell**（doorbell=0x58）；
   0x94=UIC_COMMAND_ARG_1 残留。**控制器"健康"，只是无视 6 个
   doorbell。**
3. **TRLUT/DESC_RAM/EDTL_RAM/PRDT_RAM/UAWM/UARM/TXUC/RXUC/DFC 全零**
   （主线 `ufs_qcom_dump_dbg_regs` 输出，UTP_DBG_RAMS_EN 门控开）——
   **控制器内部任务表为空 = 6 条命令从未被取走**。挂死不在"设备持有
   命令"（abort 的 pending 报告是表象），而在**取指引擎停摆**。
4. **vendor 寄存器新事实**：HW_VERSION(0xE4)=0x40020000 = **hw v4.2**
   （本机不是 3.x；与 kona 同代，主线 sm8250 路径即同代参考）；
   PARAM0(0xD0)=0x32（max HS gear=3，与 DT limit-hs-gear=3 一致）；
   SYS1CLK_1US=0xc8（200MHz ✓）；
   **CFG1(0xDC)=0x1c00052c → QUNIPRO_SEL(bit0)=0**。下游 4.19 对
   hw≥2 无条件设 QUNIPRO（`ufs_qcom_set_caps`），主线
   `ufs_qcom_select_unipro_mode` 也无条件置 1（power_up_sequence 尾）
   ——**挂死时读到 0 = 要么启动后被清、要么从未生效**。首命令探针将
   打印 CFG1 正常值作对照。
5. **DMA 缓冲段归属**：挂死缓冲 0xfe807000 在 bank2（0xc0000000-
   0xffffffff），描述符/扫描命令缓冲全在 bank1（0x871xxxxx）——但
   **fix19 的 PRDT 重定向实验已证死地址理论**（重定向 bank1 仍楔死），
   bank 归属只是伴随现象非根因。Android 无 iommus 也 dma_direct 全 bank
   工作 = 硬件可达性无虞。

**sm7225/sm8250 对照结论**（本轮既定任务）：sm7225.dtsi（Fairphone
FP4，lito 家族、主线可用）与 kona 的 READ 路径与我方**共用同一
ufshcd-core + ufs-qcom 驱动**，compose/PRDT/doorbell 代码无平台分叉；
下游 4.19 的 READ 数据路径是主线祖先（device_reset 三调用点也全部
对应）。**不存在可移植的"下游 READ 专属步骤"**——分歧只可能在运行态
状态（本轮探针取证）。

**迭代 37 镜像（9bbaa55e）三件套**：
1. **提交/完成轨迹**（ufshcd.c `ufshcd_send_command`/`compl_one_cqe`）：
   前 160 条命令逐条记 tag/op/blen/seg/sg0/sgN 物理
   址/prd 条数；首条附 gear/pwr/hs/lane/CFG1/CFG2；
2. **首 3 条 READ(10) doorbell 时间线**：+1/5/10…50ms 采样 DBR 位 +
   OCS——区分"从未取走"与"取走后卡死"；
3. **LU 激活后受控 READ 矩阵**（`picasso_read_matrix`，挂首个普通
   LUN 的 sdev_configure）：克隆 RPMB 路径用 devman 保留 tag 提交真
   READ(10) LBA0×1 块，低段(<0xc0000000)/高段(≥0xc0000000) 双缓冲
   对照，报 err/ocs/响应 UPIU 头 4 字节/data[0..7]/MBR 尾 0x55AA。
   **补上 fix14~16 自测的空洞**（当时跑在 LU 激活前，"完成"但 data
   全零=设备空操作，真实 4KB flash 读从未在受控路径测过）。三判据：
   都成功有数据→块层提交路径问题；都挂→真实数据相位全局死（PHY/
   链路）；一好一坏→地址理论复活。

**迭代 34（260380a6，待实机）：启动序列对照下游全量 diff 收口**。
以 ufs-live-dts-attr-map.md 的梳理为准逐环节 diff（完整对照表见迭代
日志迭代 34 节）——结果：PHY 上电/lane 时钟/cfg_timers/core clk ctrl
（含 hw4 的 40NS=6）/LCC/hw_clk_gating/pwr_change 全部等价，仅两处
缺失已补：
1. **RX LineCfg 使能**（下游 link startup 前清 PCS+0x148 bit1）——
   `phy-qcom-qmp-ufs.c` 新增 `.linecfg_disable_off` cfg 字段 +
   calibrate 尾部幂等 RMW；
2. **SAVECONFIGTIME_MODE**（下游 hw≥3 平台 link startup 前设
   PA_VS_CONFIG_REG1(0x9000) bit13-14=0x6000）——`ufs-qcom.c`
   link_startup PRE 补 `ufshcd_dme_rmw`。

诚实预期：两者是厂商时序旋钮，READ 楔死未必因此消失；若仍挂，**启动
序列/DT 属性方向正式穷尽**，下一步转入运行态仪器化判别（首挂 READ 的
UTRD/UPIU/PRDT 与同窗口扫描命令逐 dword 对比 + host_regs 在途位反解）。

**下一步**：READ 首挂的全新仪器化判别——扫描期命令（INQUIRY/READ
CAPACITY，小数据）与块层 READ(10)（4KB）在同一通路上的差异只剩
**数据长度**与**触发上下文**；下一轮把首条挂死 READ 的 UTRD/UPIU/PRDT
与同窗口完成的扫描命令做逐 dword 对比（含 CQHCI… 无关；含 dd 字段），
或从 host_regs dump（0x60=UTRLDBR、0x64 等）反解在途状态。

---

## 6. 方法论与教训

1. **数据源等级**（用户 2026-10-13 指令，已入迭代日志+记忆）：
   下游 4.19 内核/dtsi = 第一权威；运行时事实以**用户提供的
   live_dts.dts 为唯一权威**；不擅自 adb 实机取数。本轮两个真根因
   （②③）恰恰是历史 dump 引入、而 live_dts 本来就没有的错误属性。
2. **"属性可选"的陷阱**：ufshcd 的 clock-names 给了名字不代表驱动会
   去拿——freq-table-hz 才是建 clk_list 的开关；上游 qcom dtsi 从不
   暴露此坑因为它们全带着表。
3. **CAP 位不等于能力**：64AS 谎报 64 位 DMA 是本轮最深的坑——
   寄存器自述要与实测行为交叉验证（doorbell 锁存+零完成+地址 4GB+
   三点合围才定案）。
4. **插桩要点**：打印要放在**函数入口**（状态机拒绝路径）与
   **写后回读**（doorbell）；一次插桩覆盖多个互斥假说，比逐个猜省
   刷机轮次。
5. **门控类功能在 bring-up 后期才"长牙"**：修好时钟列表后 clk-gating
   才第一次真正关时钟并立即翻车——每修一个根因要重估"此前被它掩盖
   的下一个根因"。
6. **can_queue 之类参数有多个赋值点**：nutrs 更新路径会覆盖，钳制
   要两处同改（本轮实验白刷一次的教训）。
7. **实验代码与产品代码同一纪律**（2026-10-14 审计总结，11 条 bug 全表
   见 §5 修复史）：自测脚手架也要"设计→反查调用链→打印/反汇编验证"；
   钩子位置要证明零干扰（fix15 的 T4 覆盖在途命令、自测跑在 add_lus 前
   都是位置污染）；硬件字段单位/内存类型要落到手册级确认（UTRD 偏移
   qcom=dword 单位、ioremap=Device 内存 memset 即 Oops）。
8. **一次性实验用完即删**：fix10 的 PWM 强制遗留污染了之后 8 轮的判读
   环境（所有"HS"叙述实为 PWM）；每个实验改动必须带 TEMP-DEBUG 标记
   与撤销计划，收工前 git diff 逐块分类清零。
9. **doorbell 位由 HW 在完成时清除，重写已置位=no-op**——任何"重踢"
   类救援实验必须换新 tag 才有效。
10. **自测基线要先证明等价**：手工构造的命令要在"已知good"场景（如
    LU 激活后的扫描期）复现真实行为，才能用于"坏"场景的判别。
11. **二进制 dump 解码必须双锚点校准**（2026-10-15，三度自摆乌龙后
    立）：hexdump 的组串字节序、CDB 的 BE 字段位置，都要先用**已知
    值**（print 出的 DMA 物理地址、struct 已知字段）反推约定再解读
    其余字节——单靠"看着像"解出的 1MB/PRDT 异常全是海市蜃楼。
12. **"与 live 的偏差"叙事要定期对原文复核**：迭代 32 因不知道 ICE
    两对的存在把 75M-300M 错位记到 tx/ref 头上，造出"tx 有意偏差"
    化石叙事存活 6 个迭代——每次 live_dts 重读都是纠错机会。
13. **防护性默认（0-0/enable-only）升级为 live 实频前，先核实驱动侧
    可 rate 能力**：分支时钟无 set_rate 且无父源时，probe 期的
    set_clk_freq 就是 -EINVAL 击穿点；"对齐"必须连同硬件支撑（RCG
    移植）一起落地，否则 0-0 不是保守而是唯一可行。
14. **判别 case 要自检"与默认相同=无效实验"**（39-D 初版写 RTT=2
    撞设备默认值）；自制命令实验要先证明派发路径不重写（保留请求
    走 queue_reserved_command 的机制闭环）。

---

## 7. 产物 sha256

| 版本 | sha256 | 状态 |
|---|---|---|
| 29 首版（PHY 表+节点启用） | 92ab3382… | PHY✅，link -110（缺 freq-table） |
| 29-fix1 freq-table | 954841b9… | 时钟✅，dev cmd 挂（DMA 路死） |
| 29-fix2~5 插桩系列 | 83594ae9/334ba3e6/450af441/e22217f1… | 伪案清理+sid/gpios 修正 |
| 29-fix6 删 iommus | 29688118… | doorbell 锁存仍无完成 |
| 29-fix7 DMA mask 32 | 78be31bb… | **dev cmd/识别/shell 打通**（36s） |
| 29-fix8（含 78be 系）+ 关门控 | c5ba9476… | **sda+5 LUN 枚举**；READ(10) 挂 |
| 29-fix9 pmc 插桩 | d135d7c1… | PMC 切换 ret=0（真完成） |
| 29-fix10 PWM 强制 | 06c97558… | PWM 同挂 → HS 无罪 |
| 29-fix11 can_queue=1（未生效） | 41e94969… | 实判：**并发假说证伪**（outstanding=0x2 单命令在途仍挂；EH 风暴卡死 async → 无 shell） |
| 29-fix12 can_queue=1（双点钳制） | 未单刷（被 fix11 判读吸收，钳制保留） | — |
| 29-fix13 三假说单刷判别（救援+延迟+时间线+插桩拆分） | a9718cf6… | **实判：WA 排除（延迟仍挂）；救援实验设计有洞（doorbell 重写=no-op）**；SMMU 复位/SYS1CLK/hw4 实证排除三假说 |
| 29-fix14 受控自测 T0-T3（关 IE、独占 tag、纯轮询：512B 对照/4KB 无闪存/4KB+闪存/换 LU） | 9badc1bc… | **实判：四 case 全 1ms DONE——命令通路+设备闪存响应无罪（READ 在新 tag 瞬完成）；但自测 PRDT 偏移单位 bug（qcom 无 BYTE_GRAN quirk，字段=dword 单位）→ 数据相位未真正测到** |
| 29-fix15 自测修正（偏移×4 字节+快照模板）+ 新增 T4（tag1+4KB=复刻块层挂死命令） | 80939795… | **实判：T2/T4 READ 真实完成（响应 UPIU 有效）——READ 命令本体无罪；但所有 case data0=0（含 512B）→ 零数据之谜待解；挂点移动实为自测污染（T4 覆盖了已备好的首条 INQUIRY）**；shell 慢启动成功（EH 未死锁） |
| 29-fix16 自测移 probe_hba 尾部（消除污染）+ 响应 UPIU 前 16B 打印（status/residual）+ T5（GFP_DMA32 bank2 目标） | f25df2c2… | **实判：干净自测六 case 全完成（响应有效）但 data0 全零；块层 READ 原样楔死（SND #164）——发现 I 位混淆（块层 0x15/自测 0x14）与 GFP_DMA32 不落 bank2** |
| 29-fix17 去混淆三件套：自测补 I 位 + probe2 改 DT 预留 bank2（0xf0000000）+ 块层 ≥4KB PRDT 重定向实验 | 21c095ac… | **实判：ioremap(Device) 上 memset 对齐错误 Oops 击落自测（黑屏无 shell）；崩溃前 T0(bank1+I=1+512B) 数据仍零——I 位排除，目标地址是数据缺失的判别变量** |
| 29-fix18 probe2 映射改 memremap(WC)，矩阵原样重跑 | 4032e7ca… | 未单刷（被 fix19 审计吸收） |
| 29-fix19 全面审计+清除：回滚 fix10 PWM 强制遗留、redirect 三重守卫（方向/单段/尺寸）、双探针 DT 确定化（bank1 0x8a000000）、删除全部失效插桩与 can_queue 钳制 | 55ab51df… | **实判：T0b(bank2 探针)也无数据+重定向 bank1 的块层 READ 仍楔死——两条地址理论全灭；自测基线污染确认（跑在 add_lus 前，LU 未激活）** |
| 29-fix20 **干净基线**：ufshcd.c/scsi_error.c/arm-smmu.c 整体还原初始提交（99 行纯插桩零真修复）；保留 ufs-qcom.c 两实证修复（32 位 DMA/门控禁用）；DT 探针区删除；权威源逐属性对齐确认（表见 §5 尾） | 620077f3… | **实判：READ 楔死原样复现（多条 tag 并发 30s 超时→EH→offline；36.8s 到 init）——楔死=原生 bug，基线干净** |
| 30 qnoc 裁表启用 interconnects + ufshc 补两条带宽路径（上游 sm8250 同款） | 50effd3522183731e5cc1809b3a77d8209ba80d23e423e08b7f448b501fb0855 | **实判：1.78s 整机死机重启（sync_state 屏障清票断供 fabric，早于分区扫描；见 §5）** |
| 31 sm7250.c 删 .sync_state（init 票恒 INT_MAX）+ ICCD 探针（屏障见证 + 逐 BCM AMC 提交打印） | 1f44d3041ce3ac410dd050ffb9c258e889de06c3020d3d4b14d045cb75a21eef | ✅ 死机消失、存活到 init（屏障因果确认）；READ 楔死原样——**ICC 带宽假说证伪** |
| 32 ICCD 清场 + dev-reset pinctrl（ufs-qcom 两态回退 + DT 0x6f/0x70）+ 审计收口（tx_lane_enable 出局等，见 §5） | fd2aca1540212382575c8c6cb512db2db19518244091d409a19bdcf7cedb4e5f | ❌ UFS 全灭回归：hw4 REINIT quirk 使 probe 期真脉冲 ufs_reset → 设备冷复位后 link startup 连败（≤31 该 vops 是空操作）。收获：两态物理生效实证 |
| 33 清 REINIT_AFTER_MAX_GEAR_SWITCH（fw_managed_init 同款模式；probe 不脉冲，dev-reset 留 EH） | 968fc6c6bc35ba6e2a9cf58e5ce31629a6401fdf5fb23cca5216092b4cf2be3f | 未单刷（被 34 累积包含） |
| 34 启动序列对齐补齐两缺失：RX LineCfg 使能（qmp-ufs cfg 门控 + calibrate 尾 RMW）+ SAVECONFIGTIME_MODE rmw（ufs-qcom link_startup PRE，hw≥3） | 260380a69f7b0a2b29dfb1a811413b89d223fd5e68d5add8b55a542a90895735 / 30c9ce13…（宏清理重建） | **实判：仍 link startup 失败（同 32 签名）；冷启动复验仍挂→设备楔死假说证伪，回归实锤在 32 改动内** |
| 35 二分轮：关闭 34 两补齐（linecfg off + saveconfigtime #if 0 门）=精确 33 状态 | 933a694b6ad2e6f1183d5643875d5816680d16ef9e42a350cccbc013cd9a6cae | **实判：仍挂 → 34 补齐无罪、32 pinctrl 接线实锤 → 追凶到 ufshcd_init:11308 的无条件设备复位（真凶）** |
| 36 真修复：device_reset_ctrl 加 link_started 门（首次链路训练成功前不脉冲，POST_CHANGE 置位）+ 恢复 34 两补齐 + 修正 33 注释归因 | 46206affec43a6046059f5d71684af83d7f88e6ae0529109c80deb1799d456d8 | ✅ **实判：回归完全修复**（6 LUN + 36.9s init，31 形态）；READ 楔死原样——启动序列/DT 方向穷尽，转运行态仪器化；EH 新数据：OCS invalid 事后报告 + 设备/主机复位连环救不回 |
| 37 READ 仪器化三件套：提交/完成轨迹（160 条）+ 首 3 READ doorbell/OCS 时间线（采样后单行汇报）+ LU 激活后受控 READ 矩阵（低/高段双缓冲，sdev_configure 挂钩；TEMP-DEBUG 标记，判读后整体删除） | 4315a6a84a9eb87a3c76d3bf…（第二轮 review+下游对照修正版；e5aeb9e1/9bbaa55e 未刷） | 待实机。桌面判读先行：挂死命令形态=分区扫描 4KB 读完全自洽；TRLUT/DESC_RAM 全零=命令从未被取走；hw v4.2 实锤；CFG1 QUNIPRO_SEL=0 存疑（探针对照）；sm7225/sm8250=同驱动无 READ 分叉。R1 修五处（done 轨迹 is_scsi 门/sg 弃 sg_set_buf/watch 采样制/级联标注/ocs 掩码）；R2+下游对照修两处：**timeout 单位**（rq->timeout=jiffies，裸 3000 在 HZ=250=12s → msecs_to_jiffies，下游 wait_for_dev_cmd 同款）、**超时清理补 outstanding_reqs 清位**（下游 3993 同款，幽灵位污染 completed_reqs/EH 判定）。R2 新闭环验证：保留请求经 `.queue_reserved_command` 派发（scsi_lib.c:1921）不重 compose——自制 UPIU/UTRD 原样到 doorbell ✓；7.3 RPMB 超时路径无 clear_cmd（我方比主线稳）；pseudo_sdev 在 sdev_configure 前早退 ✓；自制 UPIU 与下游 prepare_utp_scsi_cmd_upiu 逐字段等价。**实判（2026-10-15）**：三大假设全灭+真凶锁定——①CFG1/QUNIPRO_SEL=0 从首命令即是常态（工作态=挂死态，排除）；②扫描命令缓冲全在 bank2 且全部秒完成（地址理论再证死）；③**矩阵 low（bank1 coherent、完美自制 READ(10)、保留 tag）同样楔死**（err=-5/ocs=0x0f/响应零/数据零）→ 块层路径无关、命令构造无关。**新事实**：矩阵楔死期间 ~50 条后续命令照常完成（逐命令楔死，非控制器死）；abort TM <1ms 秒回（设备响应正常）；DATA-IN ≥32B（RC16）实证通、4KB 数据单元必挂 → **根因=ICE 核心时钟从未使能**（我方 DT 自 29 起 8 时钟形态漏了 live 的 core_clk_ice/core_clk_ice_hw_ctl 两对；ICE 在数据通路按数据单元处理，时钟关=4KB 穿时钟域永久停滞、小控制传输旁路）。历史误读一并修正：live 的 75M-300M 属 ICE 两对，tx/ref 实为 0-0 |
| 38 ICE 时钟根因修复（用户指示按 live 实频）：①gcc-sm7250 照下游 gcc-lito 移植 `gcc_ufs_phy_ice_core_clk_src` RCG（0x7706c、GPLL0 600M：EVEN/8=37.5M EVEN/4=75M MAIN/4=150M MAIN/2=300M、map_0、下游 vdd_class 省略）+ `gcc_ufs_phy_ice_core_clk` 分支接线 parent_hws+CLK_SET_RATE_PARENT（sm6350 同款；hw_ctl 分支原本就挂 ice_core ✓）+ clocks[]/头文件 id 112（本树自有编号惯例）②ufshc DT 补 core_clk_ice(GCC 63)+core_clk_ice_hw_ctl(GCC 64)（live 同位插入）③freq-table **live 实频 75M-300M 两对**——probe 期 `ufshcd_scale_clks(ULONG_MAX, true)`（ufshcd.c:8471）会实打实 rate 到 300M，没有 RCG 就 -EINVAL 击穿 probe（0-0 方案作废）；devfreq 降频链 core→50M/unipro→37.5M/ice→75M 与 Android 行为一致；P37 探针保留一轮验证。中间版 7c05fd3e（0-0 方案）作废未刷 | f0ff4f43…（全量重建；中间版 7c05fd3e 作废未刷） | ❌ **实判：ICE 时钟假设证伪**——probe 零 rate 错（RCG 工作正常，ICE 确已使能+300M），但矩阵 READ 原样楔死（ocs=0x0f/mbr=0000）、分区表照旧不可读、40.9s init 同形。时钟轴正式穷尽（PWM✓fix10、ICE on/off✓38、降频从未发生✓36）。附带收获：freq-table 与 live 逐对一致（tx 偏差系迭代 32 误读，已更正） |
| 39 到达性判别矩阵 v2（P39，review 后终版 2590c6fc；中间版 d5107e0e 未刷）：A 正常读（基线）/ **B 越界 LBA 读**（EDTL/PRDT 与 A 同形，预期秒回 CHECK CONDITION rsp[7]=0x02、零数据相位）/ C 未配置 LUN6 / D **写 bMaxNumOfRTT=8+回读**后重试；另读 bMaxNumOfRTT+bMaxDataInSize 上下文。判别逻辑：B/C 完成=命令到达设备+描述符路径全好→楔死在数据相位/NAND/尺寸；B/C 也楔死=命令没离开主机（取指引擎拒绝），设备侧理论全灭 | 2590c6fc…（kernel 改，全量重建） | ✅ **实判（2026-10-15 深夜）：分叉裁决——命令通路全好，楔死锁定真实数据相位**。A 楔死（基线✓）；**B 越界读 0.4ms 秒回**（err=0/ocs=0x00/rsp=21 00 00 00 01 02=RESPONSE+CHECK CONDITION）；**C LUN6 0.37ms 秒回**（rsp 精确回显 lun=6）——自制命令确实到达设备、描述符被取走、响应写回完整；36 的"TRLUT 全零"确证为调试 RAM 假象。D：RTT 写 8 被 -22 拒（读回仍 2）→ RTT≠因。**决定性新线索：bMaxDataInSize=64**——与实测边界（≤64B 数据相位全通/4KB 必挂）严丝合缝；该属性可写且**两树内核都不写**（仅 sysfs 暴露），Android 侧疑由用户态/引导链设置 |
| 40 mdi 判决实验（P40）：设备描述符 dump（bDeviceRTTCap@0x1C 主线权威偏移 + 0x18-0x37 原始窗）+ A 基线 + **E：写 bMaxDataInSize=0x1000→回读→重试 READ**——若 mdi 单位=字节且写后 READ 通（mbr=55aa）即根因+修复双落 | 2c8ad3a7…（kernel 改，全量重建） | 待实机。判据：E 成功=mdi=64 即真凶（boot 链残留默认值，主线缺一步 host 初始化），修复=probe 期按能力写 mdi；E 仍挂=mdi=64 为单位歧义（64×对齐=256KB 正常值），战场转设备 NAND/数据流深处 |

*成文于迭代 29 排查中途（2026-10-13），2026-10-14 全天判别与清场后重整
（fix11~fix20 实录详见 copydown-shim-port.md 迭代 29 行表）。数据 READ 打通后补全最终形态。*

---

## 8. 2026-10-15 攻坚日志：READ(10) 判别框架收敛（37→39）

> 当日主题：回归修复收官（36 交付）→ 仪器化三件套（37）→ ICE 时钟
> 假设的提出与证伪（38）→ 到达性分叉实验（39 待实机）。三次实机、
> 三轮 review、十一个假设生死判决——卡点虽未破，**判别框架收敛为
> 一次分叉实验**（39-B/C），战场从"大海捞针"收敛为"二选一"。

### 8.1 当日时间线

| 时段 | 迭代/动作 | sha | 判决 |
|---|---|---|---|
| 凌晨 | 36 判读交付（link_started 门修复 32~35 回归确认） | 46206aff | ✅ 回归修复；启动序列/DT 方向穷尽 |
| 白天 | 37 桌面判读（36 EH dump 离线解码）+ 仪器化三件套构建（R1/R2 两轮 review+下游对照，共修 7 处） | — | 三处自我纠错 + 三新事实（见 8.2） |
| 晚 | 37 实判 | 4315a6a8 | ✅ 三假设灭（块层/构造/地址）+ 新事实四件（见 8.3）；锁定 ICE 线索 |
| 晚 | 38 构建（0-0 废版 → 用户指示 live 实频 → gcc RCG 移植） | 7c05fd3e→f0ff4f43 | — |
| 深夜 | 38 实判 | f0ff4f43 | ❌ ICE 假设证伪；**时钟轴穷尽**；确定遗产（freq-table 与 live 逐对一致） |
| 深夜 | 39 构建（到达性矩阵 v2 + R3 review） | 2590c6fc | ⏳ 待实机（分叉判据见 8.4） |

### 8.2 桌面判读的自我纠错（离线分析也有坑）

1. **CDB 字节序误读**：READ(10) 传输量在 bytes[7..8]（BE16），初读把
   bytes[8..9] 当 BE16 → "1MB 大读"实为 **1 块=4KB 分区扫描读**；
2. **寄存器名误记**：0x60 是 UTRLRSR（运行位）不是 doorbell（0x58），
   0x94 只是 UIC arg 残留——"在途位反解"叙事随之作废；
3. **hexdump 字节序约定**：print_hex_dump groupsize=4 的组串 = LE u32
   值（内存字节为组串反序）——本日三度自摆乌龙后以"已知 DMA 物理地址
   + 已知 struct 字段"双锚点三次校准锁定。教训入 §6-11。

新事实三件：**本机 UFS 控制器 hw v4.2**（0xE4=0x40020000，kona 同代）；
**TRLUT/DESC_RAM/EDTL_RAM/PRDT_RAM/UAWM/UARM/TXUC/RXUC/DFC 全零**
（36 dump——当时判"命令从未被取走"，**39-B/C 将复核**：调试 RAM 窗口
在 hw4.2 上是否可读本身未证）；CFG1 QUNIPRO_SEL=0（后被 37 证明为
工作常态，非异常）。

### 8.3 假设生死表（截至 39 刷机前，按轴归并）

| 轴 | 假设 | 判决 | 决定性证据 |
|---|---|---|---|
| 提交路径 | 块层 vs 扫描路径差异 | ❌ 死 | 37：矩阵在 devman 保留 tag 同样楔死 |
| 提交路径 | 命令构造（UTRD/UPIU/PRDT）缺陷 | ❌ 死 | 37：逐字节完美的自制 READ 同挂；扫描命令同构造全通 |
| 地址 | 缓冲物理段（bank1/2、4GB+、重定向） | ❌ 死 | 29-fix19 重定向仍挂；37 扫描缓冲全在 bank2 秒完成 |
| 并发 | 多命令在途互扰 | ❌ 死 | 29-fix11 单命令（outstanding=0x2）仍挂 |
| 链路 | HS gear/PHY 速率 | ❌ 死 | 29-fix10 强制 PWM 同挂 |
| 链路 | hibern8/auto-hib | ❌ 死 | HW/SW H8 计数恒 0 |
| 链路 | devfreq 降频窗口 | ❌ 死 | 36：EH 时刻 core 仍 200M/unipro 150M |
| 链路 | QUNIPRO_SEL=0 | ❌ 死 | 37：首命令（工作态）即 0=常态 |
| 时钟 | ICE 核心时钟未使能 | ❌ 死 | 38：RCG 移植+300M 实使能，仍挂 |
| quirk | PA_TACTIVATE/HIBERN8TIME 调优缺失 | ❌ 死 | 39 审：仅 UniPro<1.6 生效，本机 1.8+ 两树均空转 |
| quirk | VS_DEBUGSAVECONFIGTIME 等 quirk 差异 | ❌ 死 | 39 审：两树都只有 hisi 处理；位义不同但无数据通路项 |
| ICC | 带宽投票不足 | ❌ 死 | 31 实判 |
| 流控 | **RTT 信用（bMaxNumOfRTT）** | ⏳ | 39-D：写 8+回读+重试 |
| 到达性 | **数据相位/NAND（设备侧）vs 取指拒绝（主机侧）** | ⏳ | **39-B/C 分叉**（见 8.4） |

**行为边界（实机数据点）**：数据相位 ≥64B 实证通（INQUIRY 36B/RC16
32B/MODESENSE 页 64B）、4KB 数据单元必挂；abort TM 早期 <1ms 秒回
（设备命令处理器活着）；楔死期间 ~50 条并发命令照常完成（**逐命令
楔死**，控制器取指引擎对其他命令仍工作）；零 UIC/DL 错误、零中断。

### 8.4 39 的分叉判据（当前判别框架）

**B=越界 LBA 读**（CDB 仅 LBA 与基线不同、EDTL/PRDT 全同形；设备应
立即回 CHECK CONDITION，零数据相位）；C=未配置 LUN6（同判别力）：

- **B/C 秒回** → 命令确实到达设备、描述符/取指路径全好 → 楔死锁定在
  **真实数据相位**（设备 NAND 读挂起 or DATA-IN 流控）→ 战场转设备侧
  （RTT-D 结果并入此支判读）；
- **B/C 也楔死** → 命令**根本没离开主机**（取指引擎对含数据相位的
  命令选择性拒绝；36 的"TRLUT 全零"复核为真）→ 主机控制器侧战场。

### 8.5 当日确定遗产（无论 39 结果均保留）

1. **gcc-sm7250 的 `gcc_ufs_phy_ice_core_clk_src` RCG 移植**（0x7706c，
   照下游 gcc-lito；主线原本只有孤儿分支）+ ufshc 10 时钟 DT 形态 +
   freq-table 与 live **逐对一致**——正确的对齐，非实验；
2. 自制命令的**派发机制闭环**：保留请求经 `.queue_reserved_command`
   （scsi_lib.c:1921）派发、不重 compose——克隆 RPMB 路径构造任意
   UPIU 的方法论（PRDT 只设 dma 字段、超时 msecs_to_jiffies、清槽+
   清 outstanding 位）；
3. review 三轮共修 **11 处**（R1×5：done 门控/sg_set_buf 垃圾页/
   watch 打印失真/级联标注/掩码统一；R2×2：**timeout 单位**、
   **outstanding 幽灵位**两个真 bug；R3×2：D 无效实验、query_attr
   锁契约实证；另清理 2 处过期注释/文档化石）；
4. 历史误读修正三件：tx"有意偏差"（迭代 32 对齐错位化石）、"1MB
   READ"（CDB 字节序）、0x60 寄存器名。
