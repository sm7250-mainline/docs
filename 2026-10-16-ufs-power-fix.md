# 2026-10-16：UFS 供电模式修复与 P41 验证基线

状态：**P41 实机验证成功，READ(10) 与分区扫描恢复**。实机日志为
`uart_early.log`，内核构建 #79，编译时间 2026-10-16 02:07:40 CST。
用户另在 shell 中确认 `/proc/partitions` 已列出 6 个 LUN 的完整分区表。

## 1. 本轮纠正的结论

- P39 的越界 LBA / LUN6 返回 CHECK CONDITION，证明 COMMAND/RESPONSE
  通路可用；它们没有正常的读数据相位，不能据此排除主机 DATA-IN / DMA，
  更不能直接断言 NAND 损坏。
- `bMaxDataInSize=64` 表示 64 个 512B 单元，即 32KiB。它是单字节属性；
  P40 写 `0x1000` 的实验基于错误单位，已撤销。
- P39 的 RTT=8 写入被拒，读回仍为 2，不构成有效变值实验；RTT 主要服务
  DATA-OUT。P41 只读这两个属性，并记录读取失败。
- UTRD 的 Response/PRDT 字段采用 DWORD 单位。原 `ufshcd_init_lrb()`
  把这些字段直接加入字节地址，导致调试输出少乘 4。CPU 指针及硬件
  描述符本身正确；这不是 READ 挂死的根因。

MDI 单位依据：[Kingston UFS 数据手册](https://www.mouser.com/datasheet/2/1389/Kingston_UFSnnn_CY14_02J01___v1_3-3314915.pdf)，
属性表 `bMaxDataInSize`：值以 512-byte units 表示，不能超过
`bMaxInBufferSize`，写入还要求 LU task queues 为空。

## 2. 供电模式：已确认的移植缺口

| 电源 | live DTS / 下游行为 | 修复前主线行为 | P41 |
|---|---|---|---|
| VCC / ldoc7 | UFS 申报 800000uA；模式阈值 `<0 1>`，切 HPM | 初始 LPM，无 DRMS 权限 | HPM |
| VDDA-PHY / ldoa5 | PHY 申报 90192uA；模式阈值 `<0 1>` | 初始 LPM，无 DRMS 权限 | HPM |
| VDDA-PLL / ldoa9 | PHY 申报 19000uA；模式阈值 `<0 1>` | 初始 LPM，无 DRMS 权限 | HPM |
| VCCQ2 / smpa4 | 1.8V，UFS 申报 800000uA | 已初始 HPM | 保留 |

源码依据：

- `live_dts.dts` 的 `rpmh-regulator-ldoc7/ldoa5/ldoa9` 均有
  `qcom,supported-modes=<2 4>`、`qcom,mode-threshold-currents=<0 1>`。
  下游 `drivers/regulator/rpmh-regulator.c` 依据模式表自动设置
  `REGULATOR_CHANGE_MODE | REGULATOR_CHANGE_DRMS`，`set_load` 选择 HPM。
- 主线 `drivers/regulator/of_regulator.c` 需要显式
  `regulator-allow-set-load` 和 `regulator-allowed-modes`。
  `core.c:drms_uA_update()` 缺少任一条件会返回 0，消费者看不到报错。
- 仅有 `vcc-max-microamp` 不能使 provider 获得模式切换权限。

三条供电在自包含 DTS 中均设为：

```dts
regulator-initial-mode = <3>; /* RPMH_REGULATOR_MODE_HPM */
regulator-allow-set-load;
regulator-allowed-modes = <3>;
```

本轮有意仅允许 HPM，作为 bring-up 的供电基线：主线 pmic5_nldo 的 HPM
阈值为 30000uA，L9A 的 19000uA 请求不足以切 HPM。直接允许 LPM/HPM
仍可能让它退回 LPM。后续恢复动态功耗管理时要单独处理阈值差异。

这已是确定的配置缺口，但 READ 的因果关系仍须实机验证。`ap_mode`
表示驱动持有的 AP 模式状态，不等于测得了 PMIC 的最终聚合状态或波形。

## 3. QPHY 投票

下游 `phy-qcom-ufs.c:ufs_qcom_phy_power_on()` 会通过 cmd-db 解析
`qphy.lvl`，并发送值 1；power_off 发送 0。GCC 的 `ufs_phy_gdsc`
不能替代这个独立的 RPMh 资源。

P41 复用主线已有的 QLINK 模型：

- `clk-rpmh.c` 的 SM7250 表注册 `RPMH_QLINK_CLK` 与
  `RPMH_QLINK_CLK_A`；不硬编码 cmd-db 地址。
- PHY 增加 `qphy` 时钟，引用 `RPMH_QLINK_CLK_A`（ID 26）。只用于
  prepare/unprepare 的 1/0 投票，不用其报告频率配置 UFS 参考时钟。
- 四个时钟顺序为 `qphy, ref, ref_aux, qref`；原参考源、aux、clkref
  仍使用原来的硬件。bulk prepare 先投 QPHY；反序 unprepare 最后撤票。
  投票在 PHY POWER_DOWN_CONTROL 解除前完成；HBA 继续持有 GCC GDSC。
- QPHY 获取或投票失败会走现有错误返回及资源回滚；SM7250 PHY 缺少
  `qphy` clock-name 会明确报错，避免继续依赖引导链残留。
- PHY binding 增加 SM7250 的四时钟形态。负载值保留于
  `sm7250_ufsphy_vreg_l`，删除 PHY 节点中主线不消费的下游 max-microamp 属性。

## 4. 调试路径清理

`drivers/ufs/core/ufshcd.c`：

- 修正 `ucd_rsp_dma_addr` / `ucd_prdt_dma_addr`：普通平台 DWORD→字节；
  保留 BYTE_GRAN quirk 平台的字节语义，中间变量用 u32 避免溢出。
- 删除手工 READ 矩阵、MDI/RTT 实验写入、自测 DMA 缓冲与手动清 tag。
  READ 验证走正常 SCSI/块层分区扫描。
- 删除每条 READ 的 50ms 轮询。提交日志在发布 doorbell 之前采集，
  避免成功完成并回收请求后还访问 `cmd/lrb`。
- `P41 submit/done` 分别限制前 8 条 READ 和前 16 条控制命令；READ
  有独立预算，不再被大量 INQUIRY 消耗掉。
- 完成日志包含 OCS、SCSI result、UPIU transaction/response/status、
  flags、residual；CHECK CONDITION 补有限长度 sense。
- 首个普通 LUN 配置期只读 MDI/RTT 并打印 VCC/VCCQ2 状态。
  PHY power_on 打印 L5A/L9A 模式、负载以及 QPHY 投票成功标记。
- 上述临时诊断仅作用于 Picasso / SM7250，以 `TEMP-DEBUG(P41)` 和
  `picasso_` / `P41` 标识，实机判读完成后可清理。

## 5. 本地验证

在 macOS arm64 上使用 NDK r29 Clang 21.0.0，重放三个文件原有 `.o.cmd`
中的 AArch64 编译选项，生成独立检查对象，不覆盖 Linux 构建产物：

- `drivers/ufs/core/ufshcd.c`：通过。
- `drivers/phy/qualcomm/phy-qcom-qmp-ufs.c`：通过。
- `drivers/clk/qcom/clk-rpmh.c`：通过。

原命令来自 Ubuntu Clang 21.1.8；本地检查仅去掉 NDK Clang 不认识的两个
关闭警告选项 `-Wno-default-const-init-unsafe`、
`-Wno-unterminated-string-initialization`，保留其他错误检查。
命令、编译日志、对象和仅含本轮差异的补丁在 `builds/p41-checks/`。

DTB 已编译并通过属性核查：三轨 HPM、负载权限、QPHY 的 phandle/ID/
顺序、UFS 10 对频率、22 个 no-map 保留区及 phandle 唯一性。
自包含 DTS 原有的数值 phandle 警告仍在；新增的一个同类警告对应
第四条 PHY 时钟的数值 phandle，引用已单独核验。

PHY binding 的条件规则已做本地 JSON Schema 正反例检查：SM7250 四时钟
及既有 SC8280XP 三时钟形态通过，缺少 QPHY/错误名称被拒绝。检查时展开
了 dt-schema 的单 compatible enum 简写；本机未安装 dt-schema，这不是
完整的 `dtbs_check`。本轮独立补丁的 checkpatch 为 0 errors / 0 warnings
（仅略过提交说明、签署和 binding 分拆等提交元数据检查）。

P41 检查 DTB：`builds/p41-checks/p41.dtb`，SHA256：
`c1761c0efaf827d61e5e5d1147c17e631bf5f3fe0afd669c1b1cb0b9c84b3673`。
仅本轮内核修改：`builds/p41-checks/p41-kernel.patch`（基于本轮修改前的
工作树，包含原有 P40 等本地改动的基线；不是针对干净上游树的完整移植补丁）。

当时的 macOS 主机只能完成上述独立对象检查，不能运行绑定 Linux 路径
的完整构建入口。随后源码回到 Linux 环境，完整构建内核 #79 并打包
实机镜像；本轮结果见 §7。

## 6. 实机判据

重建后的启动日志应包含：

```text
P41 PHY qphy.lvl vote enabled
P41 PHY vdda-phy: ... ap_mode=0x2 load_uA=90192
P41 PHY vdda-pll: ... ap_mode=0x2 load_uA=19000
P41 power vcc: ... ap_mode=0x2 enabled=1 max_load_uA=800000
P41 bMaxDataInSize=64 units512 (32768 bytes)
P41 submit ... op=28 bytes=4096 ...
P41 done ... op=28 ocs=00 result=0 trans=21 response=00 status=00 ... residual=0
```

`ap_mode=0x2` 是 regulator 框架的 NORMAL，即此 PMIC 的 HPM；不要与
DTS 的 HPM 枚举值 3 混淆。MDI 数值以设备实际读回为准。

成功需要正常 READ 完成、SCSI GOOD、分区枚举和数据读回一起成立。
仅 shell 启动或 OCS=0 不足以宣告存储已修复。若成功，再分别回退
HPM/QPHY 修复定位贡献；若仍超时，保留供电和投票证据，继续区分
设备数据生成、主机 DATA-IN 接收及 payload DMA。

## 7. 实机结果（2026-10-16，构建 #79）

实机镜像：`boot_7.3.img`，SHA256
`9b29d0b28cb074c1629ec16e0654abd28ca0c42ec0284da490bf56d03a1d0960`。

判据全部命中：

- `P41 PHY qphy.lvl vote enabled` 出现两次；L5A/L9A 均为
  `ap_mode=0x2`，负载分别为 90192/19000uA。
- VCC 为 2.96V、`ap_mode=0x2`、enabled；VCCQ2 保持 1.8V。
- `bMaxDataInSize=64 units512 (32768 bytes)`、`bMaxNumOfRTT=2`。
- 首批 4KB `READ(10)` 全部完成：`ocs=00 result=0x0 response=00
  status=00 residual=0`；首条 submit/done 间隔约 0.144ms。
- 6 个 LUN 全部 attach，`sda`～`sdf` 分区扫描完成，无 30s 超时、
  EH、offline 或 reset 记录。
- `Run /init` 提前到 2.176971s；显示链到达 `fb0: msmdrmfb`，
  触摸固件 fw01 在 15.363s 下载成功（fw_ver=20）。

因此，P39 的判别边界被实机闭环：命令/响应通路与 DATA-IN/DMA 均可
工作；此前 4KB READ 楔死与 UFS 供电模式/QPHY 独立投票缺口强相关。
尚未做 HPM 与 qphy.lvl 的单独回退实验，不能区分二者各自贡献。

本镜像的其他观察：

- 已知无害项保留：PSCI PC mode Firmware Bug、WLED short/ovp IRQ
  `-ENXIO`、内核镜像未 2MB 对齐提示、GPU dummy regulator 与 cooling
  注册失败、`sync_state pending due to gmu`。
- 新增需要收尾的触摸 warning：`nvt_esd_check_wq` 创建时缺少
  `WQ_UNBOUND`，7.3 workqueue core 将其强制为 per-CPU 并打印 WARN；
  功能未受影响，下一轮应补 flag 并回归。
- P41 的 `TEMP-DEBUG` 日志已经完成取证使命；见 §8 的清理与构建记录。

## 8. UFS 调试/测试代码清理（2026-10-16）

清理范围：

- 删除 `ufshcd.c` 的 P41 提交/完成轨迹、电源状态与 MDI/RTT 只读探针、
  描述符偏移调试修正，以及 `sdev_configure` 挂钩；
- 删除 `phy-qcom-qmp-ufs.c` 的 P41 电源模式打印；
- 同步保留索引中已完成的迭代 29 清理：`UFSDBG` 插桩、PWM 强制实验、
  `can_queue=1` 实验、`scsi_error.c` 超时打印和 `arm-smmu.c` SMR dump；
- 保留全部实证功能修复：三轨 HPM/负载权限、qphy.lvl 投票、RX LineCfg、
  ICE 时钟、dev-reset pinctrl/link_started 门、32 位 DMA 与 PM caps 平台限定。

验证：

- `ufshcd.o` 与 `phy-qcom-qmp-ufs.o` 目标编译通过；
- `bash scripts/build-boot.sh` 完整构建通过，manifest 全部断言为 True；
- 最终 `Image` 中 `P41/UFSDBG/SMMUDBG/ICCD/DBRG` 字符串均不存在；
- 清理版 `boot_7.3.img` SHA256：
  `e33ffec766cad31767a35e456d41f67ca7637ed1ff57a5146db85a79b4b74dee`。
  该镜像尚未实机回归；成功判据为 UFS 分区仍完整枚举且无 READ 超时。
