# SM7250 适配 review 复核与修复（2026-10-16）

已按 [提交 review](2026-10-16-kernel-commit-review.md) 逐项复核下游 4.19、
用户提供的 `live_dts.dts` 及主线 7.3 的调用关系。13 项问题均已确认并
修改。当前状态：目标对象编译、输入边界测试和设备树检查通过；后续
P41 镜像已完整构建并完成基础实机回归。

## 1. 问题与最终修改

| Review 项 | 复核依据 | 修改 |
|---|---|---|
| C01 NVTSPI 越界 | 下游同样直接信任头部长度；主线接收缓冲固定 1025B，原函数 ASan 复现越界 | 校验 count/编码长度/读缓冲容量/零长度；底层 SPI 读写也校验容量；入口改 0600 并要求 CAP_SYS_RAWIO；SPI 事务与固件更新使用同一 ts->lock |
| C02 GPU SMMU IRQ | live `kgsl-smmu@3da0000`：一个 global IRQ SPI673，其后 CB IRQ 从 SPI678 起 | `#global-interrupts` 改为 1 |
| C03 DSI ERR_PTR | 主线 optional clock 可返回错误指针，clk_set_parent 不接受 ERR_PTR；原 v2.9 路径有错误检查 | 四个 optional clock 逐项传播错误、检查时钟配对；reparent 失败返回错误，走上层已有上电回滚 |
| C04 触摸早期失败 NULL client | 总线规范化后 parse_dt 在 client 赋值前；GPIO 获取可 defer | 分配后立即设置 client 和 drvdata，再解析 DT；全部错误统一清理 |
| C05 触摸卸载 UAF/遗留入口 | 下游也漏清 lockdown 工作和部分入口；新 SPI driver 可正常解绑 | 初始化全部 work/timer 后才允许回滚；统一 stop 撤销入口、停止 IRQ、同步取消工作及关闭 timer，销毁队列，最后释放数据；补删 tp_lockdown_info 和 fwbuf/xbuf |
| C06 分配失败返回成功 | attrs 分配失败未设置 ret，旧标签还跳过 proc deinit | 返回 -ENOMEM；partial proc_init/sysfs 失败也进入同一清理；记录 input/IRQ/sysfs 注册状态 |
| C07 模块构建 | Kconfig 为 tristate，三个对象单独生成模块，但共享符号未导出 | 一个复合模块 novatek-nt36672c；增加 OF 依赖、移除不可用的 !OF 整数 GPIO 分支 |
| C08 pinmux selector | 下游 PINGROUP 的 NA 槽是硬件 selector，不可压缩 | 恢复 146 脚的九个函数槽位，保留所有 NA；同步生成 group 成员表；既有名称归并保留，未暴露 GPS_TX_MIRD 槽仍为保留槽 |
| C09 USB 分频 | 下游使用 GPLL0 MAIN /2.5；EVEN /1.25 在 F 宏的 u8 HID 编码中截断 | 240MHz 档恢复 MAIN /2.5 |
| C10 LCDB 注册 id | regulator_register 返回前调用 ops，ldo_rdev 尚未赋值；下游通过独立 ops 区分两轨 | descriptor 注册前设置 LDO/NCP id，电压 ops 用 rdev_get_id |
| C11 停狗影响其他平台 | 通用 Qualcomm probe 在注册后清硬件，核心仍可能认为它运行 | 仅 SM7250 compatible 在注册前 stop；其他平台保留正常接管，避免状态不一致 |
| C12 UFS workaround 作用域 | 32 位 DMA/关门控针对 Lito 实测，而公共 vops 服务多平台 | 仅 SM7250 保持 workaround；其他平台恢复 CAP 64AS 判断及 64→32 fallback、原门控 caps；同样约束此前增加的首次复位门、REINIT 清除和 saveconfigtime 补齐 |
| C13 固件解析边界 | 下游 size 是 inclusive，实际复制 size+1；旧加法检查可回绕 | 检查头部/overlay 窗口、实际 inclusive 数据范围；减法比较避免溢出，CRC 与写入均基于已验证范围；限制 checksum 回包容量 |

## 2. 同时处理的已确认观察

- GPIO 和 IRQ 的 valid-mask 回调签名不同：增加 void IRQ wrapper，调用
  SoC 的 GPIO 掩码函数后保留默认 IRQ 过滤，消除不兼容函数指针。
- 唤醒表虽然尚未启用，但与 live 不一致已确认：改为 live 的全部
  **87 条** GPIO→PDC line 映射，没有启用 `wakeup-parent`。
- 可选触摸 debugfs 的遗留 fops 成员拼写错误、短缓冲读取和固定 8B 写
  输入边界一起纠正；与解绑一起撤销 debugfs。
- IRQ 用 `IRQF_NO_AUTOEN` 注册，全部初始化完成后才启用，避免 handler
  抢先使用尚未初始化的状态；gpiod_to_irq 的负错误正常传播。
- 触摸配置数组使用带溢出检查的 kcalloc，数量与启用的 DT 子节点核对；
  固件文件名缺失返回错误，保留 proc/固件选择的既有流程。

PDC TCS 第四条命令的 ENABLE_BANK=7 在下游也是同样写法；缺少进一步
硬件语义证据，未修改其掩码。面板 mode 本身支持 120Hz，未按过期文档
的 60Hz 描述去改时序。P41 的 HPM/QPHY 修复继续保留。

## 3. 验证

本机 macOS arm64，使用 NDK r29 Clang 21 重放原 `.o.cmd` 的 AArch64
编译命令，输出到独立目录，不覆盖原 Linux 内核构建产物。仅去掉两个
本机 Clang 不支持的关闭警告选项。

- 10 个受影响 C 文件编译通过，最终检查日志无警告：触摸 3 文件、
  pinctrl-msm/pinctrl-sm7250、GCC、LCDB、watchdog、DSI host、UFS qcom。
- 触摸 3 文件的 MODULE 编译通过，并用 ARM GNU ld 合并为复合模块
  对象；不存在未解析的驱动内部 nvt/CTP/Boot_Update/ts 符号。
  没有运行完整 Linux modpost。
- 可选触摸 debugfs 分支单独编译通过，日志无警告。
- 从实际函数抽取的 ASan+UBSan 隔离测试通过：NVTSPI 短帧、零长度、
  声明长度超出缓冲、接收缓冲越界均在 SPI 前被拒；合法读写、权限和
  removing 状态返回符合约束。
- 固件 parser 的 ASan+UBSan 测试：四份实际 `.bin` 都通过 HW/SW CRC
  两条解析路径；短文件、header end 越界、inclusive end+1、u32 回绕、
  无效 overlay 输入被拒，未出现内存越界。
- 146 个 PINGROUP 按每个 selector 对照下游全部吻合（允许既有名字
  归并及未暴露的 GPS_TX_MIRD 保留槽）；group 成员与函数槽一致。
- 87 个 wakeirq entry 与 live DTS 一致。DTB 编译通过，GPU global IRQ
  数量为 1，P41 三轨 HPM 及 QPHY 引用保持正确。
- 本轮独立补丁 checkpatch：0 errors / 0 warnings（略过提交说明、
  Signed-off-by 和 binding 分拆等提交元数据规则）；git diff --check 通过。

产物与证据：`builds/review-fixes-checks/`，包括本轮独立补丁
`review-fixes.patch`、检查 DTB、命令、对象、测试源码/日志及
`verification.json`。补丁基于修改前工作树，不是干净上游树的完整适配。

## 4. 构建及实机边界

后续已在 Linux 环境完整构建 P41 镜像（内核 #79）并实机启动。基础
回归通过：亮屏、触摸固件下载、UFS READ/分区扫描均成功。

仍需专项回归：触摸解绑/重绑、模块加载，以及 GPU context fault 的
处理。P41 日志另暴露 `nvt_esd_check_wq` 缺 `WQ_UNBOUND` 的 workqueue
WARN，待修复后回归。
