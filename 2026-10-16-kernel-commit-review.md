# Linux 7.3 / SM7250 适配提交 review（2026-10-16）

> 后续复核已完成，C01–C13 均确认并在工作树修复；GPIO 回调签名和 live
> 唤醒表也已处理。下文保留 review 时的状态，最新结果见
> [复核与修复记录](2026-10-16-review-fixes.md)。

范围：`51e2945e2..21407e49f`，首个提交 `51e2945e2` 是基线，后续 5 个
适配提交全部纳入。累计 diff 为 55 个文件，16816 行新增、12 行删除。
审查对象是提交内容；当前未提交修改仅用于判断问题是否已经修复。
本轮不修改内核源码。

发现 13 项确定问题。C01–C06 为 P1；C07–C13 为 P2。
下列定位采用当前工作树行号，并标明引入提交；未被本轮后续工作树修改的
文件与 HEAD 内容一致。相同资源在多处出错时合并给出完整修复边界。

## C01 [P1] NVTSPI 使用用户提供的长度直接复制，存在越界读写

- 引入：`98f4384ce`。
- 定位：`drivers/input/touchscreen/novatek-nt36672c/nt36xxx.c:805`，
  `CTP_SPI_READ()` / `spi_read_write()`；`nt36xxx.h:160` 附近的 `rbuf[1025]`。
- 状态：HEAD 和当前工作树均存在。

`nvt_flash_read()` 只检查 `count <= 63KiB`，随后把头两个字节编码的
长度直接用于 `memcpy(buf, str + 2, len)`。它没有检查 `count >= 2`、
`len <= count - 2` 或 `len >= 1`。读操作另有固定 `rbuf[1025]`，但 SPI
接收长度是 `len + 1`；即使修好外层 count 检查，长度大于 1024 的请求
仍会越界写接收缓冲。len=0 还会使后续 `memcpy(..., len - 1)` 下溢。

`/proc/NVTSPI` 以 0444 创建，读回调却从用户缓冲读取命令并执行 SPI
读写，因此普通可读访问也能到达这些路径。

验证：抽取提交中的原函数，在只替换分配/copy/SPI 环境的用户态 ASan
程序中运行。count=2、头部声明长度=4，复现 heap-buffer-overflow：从
2 字节分配末尾读取 4 字节，发生于 SPI 调用之前。没有访问实机。

建议：先验证完整协议帧，再分别限制读写长度；底层 CTP_SPI_READ/WRITE
也要检查接收/发送缓冲容量；调试原始 SPI 入口应限制访问权限。

## C02 [P1] GPU SMMU 的全局中断数量多了一个，导致 context IRQ 错位

- 引入：`cc8de2b23`。
- 定位：`arch/arm64/boot/dts/qcom/sm7250-xiaomi-picasso.dts:1975`，
  `iommu@3da0000` 的 `#global-interrupts = <2>`。
- 状态：HEAD 和当前工作树均存在。

用户提供的 `live_dts.dts:10689` 明确是 1 个全局 IRQ：SPI 673；其后
SPI 678–685 是 context IRQ。主线提交沿用同一中断列表却声明 2 个全局
IRQ。`arm_smmu_device_probe()` 从 `global_irqs + i` 开始填 context IRQ
数组，因此 CB0 被配到 SPI 679，SPI 678 被当成全局 fault IRQ；后续 CB
依次错位。

启动能过不代表映射正确：现有 UART 显示这颗 GPU SMMU 暴露 5 个 context
banks，剩余 IRQ 数量仍足够通过 probe 的数量检查。GPU/GMU 真正发生
context fault 时，错误 handler 无法处理对应 CB，可能造成 fault 无法
清除、IRQ 被禁用或 GPU 恢复失败。

建议：按 live DTS 改为 1，并核对各 CB 的 IRQ 分配。apps SMMU 当前是
1，没有把这条问题泛化到 apps SMMU。

## C03 [P1] DSI 新增可选时钟没有检查 ERR_PTR

- 引入：`cc8de2b23`。
- 定位：`drivers/gpu/drm/msm/dsi/dsi_host.c:286`，以及
  `dsi_link_clk_set_rate_6g()` 的 reparent 分支。
- 状态：HEAD 和当前工作树均存在。

四个 `devm_clk_get_optional()` 的错误没有传播。optional 仅表示属性
不存在时返回 NULL，provider 暂未就绪或分配失败仍返回 ERR_PTR。
probe 返回的是此前 byte_intf 的 ret，可能为 0；power_on 又仅判断指针
非 NULL，随后把 ERR_PTR 交给 `clk_set_parent()`。

本树 `drivers/clk/clk.c:3029` 的 `clk_set_parent()` 会直接访问
`clk->exclusive_count` / `parent->core`，不接受 ERR_PTR，因此这会把
本可重试的 deferred probe 变成非法地址访问。

建议：逐个检查 IS_ERR 并返回错误；校验源时钟与 PLL 时钟成对存在；
reparent 失败应中止当前上电并回滚，不能只打印后继续。

## C04 [P1] 触摸 DT 解析失败时清理路径解引用尚未赋值的 client

- 引入：`98f4384ce`。
- 定位：`nt36xxx.c:2410` 和 `nt36xxx.c:2795`。
- 状态：HEAD 和当前工作树均存在。

`ts` 来自 kzalloc；`nvt_parse_dt(&client->dev)` 在 `ts->client = client`
之前执行。获取 GPIO、读取必要属性或分配 config_array 失败后跳到
`err_spi_setup`，最终调用 `spi_set_drvdata(ts->client, NULL)`。
此时 ts->client 仍为 NULL。结果是错误返回/延迟探测路径自身触发崩溃。

建议：解析前设置 client，或清理时直接使用 probe 参数；按初始化阶段
拆分清理标签，确保所有字段只在初始化后使用。

## C05 [P1] 触摸解绑没有停止全部异步工作，也遗留可访问入口

- 引入：`98f4384ce`。
- 定位：`nt36xxx.c:2814` 的 remove、`nt36xxx_ext_proc.c:703` 的 deinit。
- 状态：HEAD 和当前工作树均存在。

probe 排队了 5 秒后的 `nvt_lockdown_work`。remove 取消 fw/ESD 工作，
但没有取消 lockdown 工作、销毁其 workqueue，然后直接 kfree(ts)。
若在 5 秒内解绑，工作队列仍持有嵌入已释放 ts 的 work；回调还会直接
读取全局 ts。正常解绑可造成 UAF/NULL 解引用。

清理还遗漏 event_wq/resume_work、rf_timer、switch/resist 工作、
panel_info sysfs 和可选 debugfs；`nvt_extra_proc_deinit()` 未删除
`tp_lockdown_info`，其 show 函数继续访问 ts。ts->xbuf 也没有在正常
remove 释放，fwbuf 为额外的持久分配。

建议：先撤销外部入口并停止 IRQ 生产，再同步取消全部工作/定时器、
销毁队列，最后释放数据；对 probe 失败、remove、shutdown 做对称核查。

## C06 [P1] 触摸属性分配失败后释放设备状态，却向 SPI core 返回成功

- 引入：`98f4384ce`。
- 定位：`nt36xxx.c:2655` 的 attrs_p 分配失败分支。
- 状态：HEAD 和当前工作树均存在。

前面的 `nvt_extra_proc_init()` 成功后 ret=0。attrs_p 分配失败分支没有
设置 ret，直接进入回滚，最终释放 input/IRQ/ts 并返回 0。SPI core
会把已经清空私有状态的设备视为绑定成功，后续 PM/remove 或遗留 proc
访问都可能使用 NULL/已释放状态。

此外，该 goto 的标签位于 `nvt_extra_proc_deinit()` 调用之后，导致
刚创建的额外 proc 项未清除；同标签也会漏清 partial proc_init 失败时
已经创建的条目。

建议：明确设置 -ENOMEM；所有 proc 初始化失败和后续失败都应经过完整
deinit；检查 sysfs_create_group 的返回值并对称撤销。

## C07 [P2] 触摸 Kbuild 将一个驱动拆成三个互不导出符号的模块

- 引入：`98f4384ce`。
- 定位：`drivers/input/touchscreen/novatek-nt36672c/Makefile:6`。
- 状态：HEAD 和当前工作树均存在；当前 =y 配置掩盖问题。

Kconfig 是 tristate，但 `obj-$(CONFIG_TOUCHSCREEN_NT36672C)` 直接列
`nt36xxx.o nt36xxx_fw_update.o nt36xxx_ext_proc.o`。选择 =m 会生成三个
独立模块，而三个文件大量互相调用，均没有 EXPORT_SYMBOL；后两者还
没有 MODULE_LICENSE。模块构建会在 modpost 阶段失败。

建议：用一个复合模块目标，三个对象放入该目标的 -y 列表。
同一 Kconfig 也缺少 OF 依赖；!OF 分支引用未定义的 NVTTOUCH_RST_PIN /
NVTTOUCH_INT_PIN，且保留了与 gpiod 字段不匹配的旧整数 GPIO 路径。
既然本驱动依赖 DT，应明确限制配置，或补齐可工作的非 DT 实现。

## C08 [P2] pinmux 表移除 NA 占位后改变了硬件 selector

- 引入：`cc8de2b23`。
- 定位：`drivers/pinctrl/qcom/pinctrl-sm7250.c:1146` 起的 PINGROUP 表。
- 状态：HEAD 和当前工作树均存在。

`msm_pinmux_set_mux()` 用 funcs 数组下标作为硬件 mux 值写入寄存器。
当前表把部分下游 NA 槽删掉并把函数向前压缩。例如 GPIO4/5 的 qdss_cti
应在 selector 2，当前变成 1；GPIO43 的 ddr_pxi0 从 5 变成 4；GPIO90
的 NAV_PPS_IN/OUT、GPS_TX 也各提前一位。函数名存在、分组数量正确，
都不能证明 selector 正确。

对 146 个 PINGROUP 做了逐槽比较；规范化 NA/_ 和已合并函数名后仍有
58 行差异，详见证据 JSON。当前 UART/触摸使用的主要 selector 仍正确，
因此已有点亮结果没有暴露所有复用功能的问题。

建议：原位保留下游每个保留槽；名称归并不能压缩数组或丢失槽位。

## C09 [P2] USB 240MHz 表项使用硬件无法表示的 1.25 分频

- 引入：`cc8de2b23`。
- 定位：`drivers/clk/qcom/gcc-sm7250.c:286`。
- 状态：HEAD 和当前工作树均存在。

`F(240000000, P_GPLL0_OUT_EVEN, 1.25, 0, 0)` 的 F 宏编码是
`2*h - 1`，存到 u8 pre_div。1.25 生成 1.5，截断为 1；实际分频为
`(1+1)/2 = 1`，300MHz 的 GPLL0_OUT_EVEN 因而输出 300MHz，表项却
声明 240MHz。这个 RCG 没有 CLK_SET_RATE_PARENT，不能靠改父频率补偿。

用原 F 宏和原表项做了隔离计算，得到
`requested=240000000 encoded_pre_div=1 actual_at_300MHz=300000000`。
当前 DT 默认配置 200MHz，所以首轮 USB console 成功不覆盖这个档位。

建议：恢复下游的 GPLL0_OUT_MAIN / 2.5，或使用等效且可编码的 M/N 配置。

## C10 [P2] LCDB 注册期间用尚未保存的 rdev 指针判断 LDO/NCP

- 引入：`cc8de2b23`。
- 定位：`drivers/regulator/qcom-lcdb-regulator.c:252`、`:280`、`:345`。
- 状态：HEAD 和当前工作树均存在。

set/get_voltage 通过 `rdev == lcdb->ldo_rdev` 区分 LDO 和 NCP；然而
`devm_regulator_register()` 在返回前会执行 machine constraints 并调用
这些 ops，此时其返回值还没有赋给 lcdb->ldo_rdev。首次 LDO 注册会
误走 NCP 路径，读取/设置另一条电源。

当前两条电压都为 5.5V，容易掩盖错误；不同的初始寄存器值或不对称
电压约束会使初始化错误。下游使用不同的 LDO/NCP ops，不依赖这个时序。

建议：在注册前为 descriptor 设置明确 id，用 rdev_get_id 区分，或
使用独立的 per-rail 数据/ops。

## C11 [P2] Picasso 停狗逻辑作用于全部 Qualcomm 设备且破坏状态同步

- 引入：`cc8de2b23`。
- 定位：`drivers/watchdog/qcom-wdt.c:357`。
- 状态：HEAD 和当前工作树均存在。

probe 在发现硬件已启动时，会先 start 并设置 WDOG_HW_RUNNING，再注册
watchdog；新增代码随后无条件把 WDT_EN 清零。没有 SM7250 判断，也
没有 bring-up 开关，因此其他已支持平台的启动看门狗也会被关闭。
核心仍认为硬件运行，后台 ping 只重置计数，不会重新设置 WDT_EN。

Picasso bring-up 停狗是已知需求；问题是它泄漏到所有平台并保留错误的
核心状态。建议把该行为限定到显式调试配置/平台，并通过一致的状态
管理实现；正式接管路径保持原状。

## C12 [P2] UFS 的 32 位 DMA 与禁用门控也没有限定 SM7250

- 引入：`21407e49f`。
- 定位：`drivers/ufs/host/ufs-qcom.c:601`、`ufs_qcom_set_caps()`、vops 表。
- 状态：HEAD 和当前工作树均存在。

`ufs_qcom_set_dma_mask()` 无条件设置 32 位，同时通用 set_caps 删除
CLK_GATING/HIBERN8_WITH_CLK_GATING。两处说明的是 Lito 特有现象，却
通过公共 qcom vops 作用于所有高通平台。其他平台丢失正常的 64 位 DMA
能力，增加 bounce/地址空间压力，并失去原有空闲门控。

建议：以 compatible/平台 drvdata 将这两个 workaround 限定到 SM7250；
其他平台保留硬件能力判定和原有 PM caps。

## C13 [P2] 触摸固件分区长度检查与实际复制长度不一致

- 引入：`98f4384ce`。
- 定位：`nt36xxx_fw_update.c:271`、`:590`、`:601`。
- 状态：HEAD 和当前工作树均存在。

校验使用 `BIN_addr + size > fwsize`，而写入前会执行 `size = size + 1`。
在硬件 CRC 路径中，BIN_addr+size 恰等于 fwsize 的描述符会通过校验，
随后复制到文件末尾之外。两者都是 u32，相加回绕也可绕过比较。
例如 BIN_addr=fwsize-4、size=4，通过检查后实际读取 5 字节。

头部 end、overlay header 的 pos 也在充分验证所在文件范围之前读取。
合法的当前固件不触发，但截断或损坏的固件应该被拒绝，而不是导致内核
越界读取并把邻接内容写入控制器。

建议：先检查 BIN_addr < fwsize，再以减法比较实际长度 size+1；对所有
头部窗口单独做范围检查，并避免在溢出后才检查结果。

## 各提交覆盖及历史问题状态

| 提交 | 范围与结论 |
|---|---|
| `cc8de2b23` | DTS、GCC/DISPCC/GPUCC、DPU/panel/DSI、TLMM、LCDB、ICC、watchdog、注册项；确定问题见 C02/C03/C08–C11 |
| `813cbd083` | a620 catalog、GMU/PDC/HFI、gfx3d workaround、zap 装载；核对了下游分支和调用链，未把已知 XO 19.2MHz 限制当作新问题；PDC 观察见下 |
| `c88e9cdac` | 触摸 DT 与 Kconfig/Kbuild 中间提交；引用了尚未提交的 novatek-nt36672c/Kconfig，单独检出不能完成 Kconfig 解析；已由下一提交补齐，不作为当前未修项 |
| `98f4384ce` | 标准 SPI 驱动及全部触摸文件；确定问题见 C01/C04–C07/C13 |
| `21407e49f` | UFS PHY、时钟、DMA/PM workaround 与调试代码；公共平台影响见 C12；部分实验遗留已在工作树清除 |

当前工作树已处理、但尚未包含于提交历史的事项：

- UFS 强制 PWM、can_queue=1 及旧 UFSDBG/SMMUDBG/scsi_error 打点清理。
- ICE RCG/两条时钟补齐、UFS reset pinctrl/link_started 修复。
- NoC 缺失 BCM 裁剪和 sync_state 保活处理。
- P41 的 L7C/L5A/L9A HPM、QPHY 投票和调试地址纠错。

因此只检出 HEAD 仍不等于当前能测试的工作树。P41 的 descriptor 调试
地址问题来自原始 core 行为，不能算作上述 5 个适配提交新增的缺陷。
未提交的 netfilter 等其他改动不在本次提交 review 范围内。

## 不作为确定回归的观察

- `pinctrl-msm.c:1507` 把 int 返回值的 GPIO valid-mask 回调直接用于
  void 返回值的 IRQ valid-mask 回调。重放该文件的 AArch64 编译命令，
  Clang 报 `-Wpointer-type-mismatch`，当前配置仍生成对象；没有证明它在
 当前调用方式下引发运行故障。建议给 IRQ 层加 void wrapper，显式调用
  GPIO 层函数，消除不兼容函数指针。编译日志已保存。
- a620 PDC TCS1/TCS3 新增第 4 条 vrm.soc 命令，但 ENABLE_BANK 仍为 7。
  下游 4.19 也采用同样写法；仅凭这点不能把它认定为移植回归。需要
  PDC sequencer/TCS 的硬件语义或实测确认后再改掩码。
- TLMM wakeirq 表与提供的 live 映射不一致；当前 TLMM 没有启用
  wakeup-parent，所以不是本轮运行故障。恢复深睡唤醒时必须改为已归档
  的 87 条真实映射，不能直接启用现表。
- panel mode 源码实际为 120Hz（clock 公式乘 120），旧文档的 60Hz
  描述不准确；面板本身支持该模式，这不单独构成代码 bug。
- 不把缺少完整桌面、GPU 高频恢复、触摸灭屏通知、外设尚未启用等已知
  bring-up 范围限制列为新发现。

## 验证与证据边界

证据目录：`builds/review-21407e49f/`。

- 保存了全部 5 个提交的范围清单；逐提交 diff 和 HEAD 文件快照用于
  审查，避免把未提交代码误当成提交内容。
- GCC/DISPCC/GPUCC 分别比较了 86/46/14 个同名对象的 offset、cmd_rcgr、
  halt_reg/halt_check、hwcg、enable_reg/mask；这些字段没有新增差异。
  频率表另行核查，发现 C09；不能把寄存器偏移一致推断为整体行为一致。
- 146 个 TLMM PINGROUP 做逐槽对照，差异写入 pinmux-slot-diff.json。
- ICC 的 link_nodes 数量检查未发现与 num_links 不符的条目；核对了
  provider 数组与 binding 编号、BCM 和同步屏障的状态。
- 已提交 DTS 编译成功，108 个 phandle 唯一、22 个 no-map；对启用节点
  的 clocks/assigned clocks/phys/resets/power-domains/iommus 做参数
  边界检查，无未解析 provider 或缺失参数。该检查不代表硬件语义正确，
  C02 正是参数合法但分组含义错误的例子。
- C01 使用原函数的隔离 ASan harness 复现；C09 使用原 F 宏和表项
  计算实际编码。没有通过 adb/fastboot 访问或修改设备。
- pinctrl-msm 的单对象编译通过但带回调类型警告；使用本机 NDK r29
  Clang，保留原目标编译选项，仅去掉两个该版本不支持的关闭警告选项。
- 本轮为代码与数据 review，没有构建全部历史版本或宣称通过实机回归。
  原 Linux 构建环境仍是后续完整镜像验证所需环境。
