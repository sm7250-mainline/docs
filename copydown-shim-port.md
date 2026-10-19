# Redmi K30 5G (picasso / SM7250) copydown bootshim 移植日志与技术文档

> 2026-10-16 后续：适配提交 review 的 13 项问题复核并修复，目标编译和
> 输入边界验证通过；尚未生成新镜像。见 [修复记录](2026-10-16-review-fixes.md)。

> 最新工作：**P41 供电模式/QPHY 投票修复实机成功**，4KB READ(10)、
> 分区扫描与数据读回全部恢复；见 [2026-10-16 记录](2026-10-16-ufs-power-fix.md)。
> 本文较早的“当前状态”表包含历史快照，最新状态以该记录为准。

> 记录从参考实现分析到主线内核成功启动的完整过程：每个操作、每个技术决策、
> 每次失败与修复，以及每次修改后的打包方法（全部命令可直接复现）。

---

## 目录

1. [背景与参考实现（lmi）](#1-背景与参考实现lmi)
2. [目标设备基线（boot_factory.img 实测）](#2-目标设备基线boot_factoryimg-实测)
3. [移植设计：shim 布局与关键决策](#3-移植设计shim-布局与关键决策)
4. [打包管线（构建脚本）](#4-打包管线构建脚本)
5. [操作日志（按迭代顺序）](#5-操作日志按迭代顺序)
6. [内核侧改动清单](#6-内核侧改动清单)
7. [runtime DTB 改动清单](#7-runtime-dtb-改动清单)
8. [复现手册（从零到 boot.img）](#8-复现手册从零到-bootimg)
9. [当前状态与遗留问题](#9-当前状态与遗留问题)

---

## 1. 背景与参考实现（lmi）

参考设备：Redmi K30 Pro / POCO F2 Pro（lmi，SM8250），其主线 Linux 通过
"copydown bootshim" 启动，代码位于相邻仓库 `sm8250-xiaomi-lmi-boot`。

### 1.1 为什么需要 shim

小米 stock ABL 按 Android 约定处理 boot image：自己解析 Image 头、对 DTB
做 board-id 选取并套 dtbo overlay、把 ramdisk 地址注入 DTB `/chosen`。
主线 Linux 需要的 boot 协议（raw arm64、x0=DTB）与之不完全兼容，
shim 的作用是在 ABL 与内核之间做一次"翻译"。

### 1.2 lmi shim 机制（`linux-copydown-embedded-dtb.S.in`）

```
偏移 0x000000   假 arm64 Image 头（ARMd magic、text_offset、image_size 覆盖全 shim、flags 0xa）
                + `b shim_entry`
偏移 0x200000   .incbin 内嵌的未压缩非 EFI Linux Image（2MB 对齐）
               copy_entry（拷贝循环本体，4K 对齐，位于目标区之外）
               runtime_dtb（.incbin 主线 DTB）
               image_end
```

核心流程（M2j 阶段）：

1. ABL 把 shim 当普通内核解压/搬运后跳入（shim 头是合法 Image 头，`ARMd`@0x38）；
2. 全位置无关（PIC）：所有地址经 `adrp`+`add` 按 PC 相对计算；
3. **copydown 拷贝**：把 2MB 处内嵌的 Linux Image 前向 8 字节拷贝到 `_start`
   （ABL 落点）。`dest < src` 且区间重叠 → 前向拷贝重叠安全；
   `copy_entry` 在目标区外 → 拷贝不会覆盖自身代码；
4. 缓存维护：`dc cvau` 按 `ctr_el0` DminLine 步进清理拷贝后的内核区与 DTB 区，
   然后 `dsb ish; ic iallu; dsb ish; isb`；
5. 交接：`x0 = runtime_dtb`、`x1=x2=x3=0`、`br _start`（base+0 处 code0/1
   是 NOP + `b primary_entry`，自动分支到 text_offset 处的真正入口）；
6. lmi 专用变体在跳转前拉低 TLMM GPIO46（panel reset，MMIO 直写 south-tile TLMM）。

lmi 打包（`mkboot-linux-copydown-lmi.sh`）：模板展开 → clang/ld.lld 汇编链接
（`-Ttext=0x0`，文件偏移=运行偏移）→ objcopy 裸二进制 → **gzip**（lmi 原厂
kernel 是 Image.gz）→ **追加 stock kernel DTB**（模仿 Image.gz-dtb）→
mkbootimg（header v2）→ manifest 断言。

### 1.3 两个移植关键差异（picasso vs lmi）

| 维度 | lmi | picasso |
|---|---|---|
| ABL | LK 系（跳内核前停看门狗、EL1 交权） | **UEFI 系**（ExitBootServices、可能 EL2 交权、**不停看门狗**） |
| 原厂 kernel | Image.gz + appended kernel DTB | **未压缩 Image、无 appended DTB** |
| dtb 字段 | QCDT 格式 blob | 12 个连续拼接 FDT（无 dt_table 头） |
| panel reset | TLMM GPIO46（MMIO） | PM8150L GPIO3（SPMI，未实现） |

---

## 2. 目标设备基线（boot_factory.img 实测）

`/home/yucong/Documents/redmi_k30_5g/boot_factory.img` 是直接可用的原厂
Android boot.img，全部事实从该文件直接解析：

| 项目 | 实测值 |
|---|---|
| 头版本 | v2，pagesize 4096，base 0，header_size 1660 |
| kernel_offset | 0x8000（压缩载荷暂存地址，非运行地址） |
| kernel_size | 33980440（**未压缩**下游 Image：code0 `0x14704000`、text_offset `0x80000`、flags `0xa`、magic `ARMd`） |
| kernel 后 | 全零填充，**无 appended kernel DTB** |
| ramdisk | gzip cpio @ file offset 0x206a000，1326879 字节 |
| dtb 字段 | file offset **0x21ae000**，1863938 字节 = **12 个连续拼接的 FDT**（`d00dfeed` 开头，无 dt_table 头，各带 `qcom,msm-id`/`qcom,board-id` 根属性） |
| dtb_addr | 0x1f00000（内存装载地址，非文件偏移） |
| boot 分区 | 134217728（128MB） |

ABL 行为（从 `live_dtb.dtb`/`live_dts.dts` 与启动日志证实）：

- 按 msm-id/board-id 扫描 dtb 字段选取 DTB（`qcom,msm-id=<0x190 0x10000>`、
  `qcom,board-id=<0x24 0>` 匹配当前设备）；
- 对选中的 DTB 套 dtbo overlay（dtbo_idx=0）——**字段里放主线 DTB 会导致
  `Error: Dtb overlay failed` 直接中止启动**；
- 把 boot header cmdline + `androidboot.*` 写入交付 DTB 的 `/chosen/bootargs`，
  并把 ramdisk 地址写入 `linux,initrd-*`；
- 内存基址 `0x80000000`，RAM Partitions：
  `[0x80000000, +0x3CC00000) [0xC0000000, +0x40000000) [0x100000000, +0x100000000)`；
- 跳内核前打印 `Shutting Down UEFI Boot Services ... Exit EBS ... UEFI End`。

其他硬件事实（`docs/picasso.md`）：TLMM 单段 `0x0f000000`；UART console =
SE2 `0x888000`；APSS 看门狗 `0x17c10000`（SPI 0/1）。

---

## 3. 移植设计：shim 布局与关键决策

文件：`scripts/copydown-shim.S.in`（模板，含 `@LINUX_IMAGE@`/`@RUNTIME_DTB@` 占位）。

```
偏移 0x0       外层 Image 头：b shim_entry / text_offset=0x80000 /
               image_size=image_end-_start / flags=0xa / "ARMd"@0x38
偏移 0x40      shim_entry: bl el_drop → 打印 'S' → br copy_entry
偏移 0x80000   alt_entry:  bl el_drop → 打印 'A' → br copy_entry
偏移 0x200000  linux_image（.incbin 未压缩非 EFI Image，2MB 对齐）
               copy_entry:  前向 8B 拷贝 linux_image → _start
               copy_done:   打印 'C' → dc cvau（DminLine 步进）清理内核区
                            → 清理 runtime_dtb 区 → dsb/ic iallu/dsb/isb
                            → 打印 'K' → x0=runtime_dtb, x1..x3=0 → br _start
               el_drop:     EL2→EL1 降级（CurrentEL 检查）
                            + 停看门狗（0x17c10000 的 +0x8、+0x40 写 0）
               uart_putc:   GENI SE2 @0x888000 直写单字符（有界等待）
               runtime_dtb / image_end
```

### 3.1 设计决策与理由

| 决策 | 理由 |
|---|---|
| 外层 text_offset 保持 **0x80000**（同原厂）+ alt_entry 桩 @0x80000 | ABL 无论跳 base+0（`b shim_entry`）还是按头字段跳 base+0x80000（alt_entry），都收敛到拷贝路径 |
| 内嵌 Image 放 **2MB 对齐**处 | manifest 断言 `linux_source_alignment_ok`；fallback 路径（若从 0x80000 滑入）内嵌 Image 的 code0/1 也是合法内核入口 |
| 内核字段**不 gzip、不追加 DTB** | 原厂就是未压缩 Image + 无 appended DTB，完全复刻其形态 |
| dtb 字段放**原厂 12-DTB blob 原样** | ABL 要做 board-id 选取 + dtbo overlay，字段必须是原厂下游 DTB 才通过校验；shim 随后丢弃 ABL 交付的 DTB，改传内嵌主线 DTB |
| **EL2→EL1 降级**（`el_drop`） | UEFI ABL 在 EL2 交权；EL2 下 `dc cvau`/`ic iallu` 未定义。常量取自本树 `head.S init_el2`/`sysreg.h`：HCR_EL2.RW=1、SCTLR_EL1=0x30580000、SPSR=0x3c5（EL1h+DAIF） |
| **shim 内停看门狗** | UEFI ABL 不停狗；内核 wdt 驱动此前因兼容串回退到错误布局（见 §6.4），停狗写错寄存器。shim 在跳内核前物理清零 WDT_EN（+0x8 与 +0x40 双写） |
| UART 探针字符 S/A/C/K | 无输出时区分"没进 shim / 挂在拷贝 / 挂在缓存 / 已跳内核"；所有等待有界，UART 无时钟也不挂死 shim |
| panel reset 不做 | picasso 的 panel reset 是 PM8150L GPIO3（SPMI 事务），成本高且设备未必有 lmi 的白屏问题，留待需要时实现 |

### 3.2 内嵌内核的要求（与 shim 的契约）

- **非 EFI**（`# CONFIG_EFI is not set`）：严格按 raw arm64 boot protocol 收 `x0=DTB`；
- 4K 页（flags 0xa 匹配）；
- **必须内嵌 initramfs**（`CONFIG_INITRAMFS_SOURCE`）：shim 换掉 ABL 的 DTB 后，
  ABL 注入的 `linux,initrd-*` 信息丢失，外部 ramdisk 不可达；
- cmdline 走 `CONFIG_CMDLINE_FORCE`，用**显式 earlycon 地址**（本树 earlycon
  的 OF 声明只有 `"qcom,geni-debug-uart"`，与 DTS 节点的 `"qcom,geni-uart"` 不匹配，
  裸 `earlycon`+stdout-path 绑不上）。

---

## 4. 打包管线（构建脚本）

### 4.1 文件

| 文件 | 作用 |
|---|---|
| `scripts/copydown-shim.S.in` | shim 汇编模板（`@LINUX_IMAGE@`/`@RUNTIME_DTB@` 占位） |
| `scripts/mkboot-linux-copydown-picasso.sh` | 一键打包：提取原厂 dtb 字段 → 模板展开 → 汇编/链接/objcopy → mkbootimg → manifest |

### 4.2 工具链

```
TC=/home/yucong/Application/arm-gnu-toolchain-15.2.rel1-x86_64-aarch64-none-elf/bin
$TC/aarch64-none-elf-gcc      # 汇编（-c）
$TC/aarch64-none-elf-ld       # 链接（-Ttext=0x0 --entry=_start）
$TC/aarch64-none-elf-objcopy  # -O binary
/usr/bin/mkbootimg            # Ubuntu 包版本（支持 header v2）
/usr/bin/dtc                  # runtime DTB 修改
```

注意：宿主 clang 的 MC 对 adrp 同类符号也发重定位，**不能无链接直接 objcopy**；
GNU 工具链的 ld 是必需环节。内核构建用 clang 21 + LLVM=1，ld.lld 从包提取：
`apt-get download lld-21 && dpkg-deb -x lld-21*.deb /tmp/lld-extract`，
构建时 `PATH=/tmp/lld-extract/usr/lib/llvm-21/bin:/usr/lib/llvm-21/bin:$PATH`。
**ld.lld 缺失时**：`build-boot.sh` 自动回退 `LD=<GNU TC>/aarch64-none-elf-ld`
（LTO_NONE/无 CFI 时功能等价；首次回退触发 syncconfig 重探测并移除
`CONFIG_RELR`，Image 增大约 1.2MB，属预期，详见 §5 迭代 11）。

### 4.3 打包流程（脚本内部步骤）

```bash
# 1. 从 boot_factory.img 提取原厂 dtb 字段（file offset 0x21ae000, 1863938 B）
#    header v2 几何: kernel@0x1000(33980440) → ramdisk 页对齐 → dtb 页对齐
python3 - <<PY
ks=33980440; rs=1326879; ps=4096
kend = ps + ks
rend = ((kend + ps - 1) // ps) * ps
dend = ((rend + rs + ps - 1) // ps) * ps      # = 0x21ae000
f = open('boot_factory.img','rb'); f.seek(dend)
open('$OUT/stock-dtb-field.img','wb').write(f.read(1863938))
PY

# 2. 模板展开
sed -e "s|@LINUX_IMAGE@|$LINUX_IMAGE|" -e "s|@RUNTIME_DTB@|$RUNTIME_DTB|" \
    scripts/copydown-shim.S.in > $OUT/copydown-shim.S

# 3. 汇编 + 链接 + 取裸二进制
$TC/aarch64-none-elf-gcc -c $OUT/copydown-shim.S -o $OUT/copydown-shim.o
$TC/aarch64-none-elf-ld -Ttext=0x0 --entry=_start $OUT/copydown-shim.o -o $OUT/copydown-shim.elf
$TC/aarch64-none-elf-objcopy -O binary $OUT/copydown-shim.elf $OUT/copydown-shim.bin

# 4. mkbootimg（header v2，参数与原厂头一致）
/usr/bin/mkbootimg \
  --header_version 2 --kernel $OUT/copydown-shim.bin \
  --ramdisk unpacked_boot_img/ramdisk --dtb $OUT/stock-dtb-field.img \
  --pagesize 0x1000 --base 0x0 --kernel_offset 0x8000 \
  --ramdisk_offset 0x01000000 --second_offset 0x0 --tags_offset 0x100 \
  --dtb_offset 0x1f00000 --board '' --os_version 16.0.0 --os_patch_level 2025-12 \
  --cmdline 'console=ttyMSM0,115200 earlycon=msm_geni_serial,0x888000 rw' \
  -o boot-linux-copydown-picasso.img

# 5. manifest：解析 shim.bin，断言
#    stage=M2j / x0=embedded_runtime_dtb / copy_direction=forward
#    linux_source_offset=0x200000 / linux_source_alignment_ok=True
#    copy_entry_outside_destination=True / copy_overlap_safe=True / boot_size_ok=True
#    + 各输入 sha256/size
```

### 4.4 打包命令（每次修改后复现）

```bash
cd /home/yucong/Documents/redmi_k30_5g
LINUX_IMAGE=sdm765_sm7250_linux-6.17.0/arch/arm64/boot/Image \
RUNTIME_DTB=builds/copydown/runtime-dtb-stdout.dtb \
bash scripts/mkboot-linux-copydown-picasso.sh
# 产物: boot-linux-copydown-picasso.img + builds/copydown/boot-linux-copydown-picasso.manifest
```

反汇编校验（每次 shim 改动后必做）：

```bash
TC=/home/yucong/Application/arm-gnu-toolchain-15.2.rel1-x86_64-aarch64-none-elf/bin
$TC/aarch64-none-elf-objdump -d builds/copydown/copydown-shim.elf
# 重点核对: shim_entry/alt_entry 的 adrp 目标、copy_entry 的 x0/x1/x2、
#           el_drop 的 msr 序列、uart_putc 的寄存器偏移、WDT 地址立即数
```

---

## 5. 操作日志（按迭代顺序）

### 迭代 0：直接打包尝试（错误路径，记录在案）

**操作**：把 `unpacked_boot_img/` 的主线内核按原厂头参数直接 mkbootimg，
dtb 字段放"主线 DTB + 原厂 blob"（主线在前）。

**结果**：ABL 日志：

```
ApplyOverlay: ufdt apply overlay failed
Error: Dtb overlay failed
```

**诊断**：ABL 会对 dtb 字段选取的 DTB 套 dtbo overlay——主线 DTB 被选中后，
stock overlay 无法应用 → 启动中止。**教训：dtb 字段必须保持原厂内容**。
（产物 `boot_6_17_0_direct.img` 作废，sha256 `df98411a...`。）

### 迭代 1：第一版 copydown shim

**改动**：建立 `scripts/copydown-shim.S.in`（外层 text_offset 0x80000 +
alt_entry）+ `scripts/mkboot-linux-copydown-picasso.sh`；内核用当时的
12.5MB 主线 Image；dtb 字段用原厂 blob 原样。

**结果**：ABL 日志通过 overlay（`Apply Overlay total time: 1250 ms`），
走到 `Exit EBS ... UEFI End` 后完全静默。

**产物**：sha256 `d122ab8a...`

### 迭代 2：EL 降级 + UART 探针

**诊断假设**：UEFI ABL 在 EL2 交权 → shim 的 `dc cvau`/`ic iallu` 未定义 → 异常。

**改动**：shim 加 `el_drop`（CurrentEL 检查 → HCR_EL2.RW=1 → SCTLR_EL1=
0x30580000 → SPSR=0x3c5 → eret）+ `uart_putc`（GENI SE2 直写，S/A/C/K 探针，
寄存器偏移取自 `qcom_geni_serial.c`：STATUS 0x40、TX_TRANS_LEN 0x270、
M_CMD0 0x600 写 `UART_START_TX(1)<<27`、TX_FIFO_STATUS 0x800、TX_FIFOn 0x700）。

**结果**：UART 出现 **`SCK`** —— shim 全链路正常（base+0 进入、EL1、拷贝、
缓存、跳内核），但内核仍零输出。

**产物**：sha256 `b138f1f4...`

### 迭代 3：内核没有串口驱动（决定性发现）

**诊断**：`strings arch/arm64/boot/Image | grep -c geni` = 0 —— **内核里根本没
串口驱动**。根因：`CONFIG_SERIAL_QCOM_GENI` 依赖 `CONFIG_QCOM_GENI_SE`，
后者未启用 → `olddefconfig` 静默丢弃串口选项（实时复现确认）。

**改动**：

```bash
scripts/config --enable QCOM_GENI_SE --enable SERIAL_EARLYCON \
  --enable SERIAL_QCOM_GENI --enable SERIAL_QCOM_GENI_CONSOLE \
  --enable SERIAL_QCOM_GENI_EARLYCON
scripts/config --set-str CMDLINE "earlycon=qcom_geni,0x888000 console=ttyMSM0,115200 rw"
scripts/config --enable CMDLINE_FORCE
```

用显式 earlycon 地址（本树 OF earlycon 声明只有 `qcom,geni-debug-uart`，
与 DTS 的 `qcom,geni-uart` 不匹配，裸 earlycon 绑不上）；`CMDLINE_FORCE`
让 cmdline 完全确定。

**结果**：内核日志出现，但立即 panic：

```
[Firmware Bug]: Kernel image misaligned at boot   ← 非致命（_end 非 2MB 对齐）
Kernel panic - not syncing: Failed to allocate page table page
```

**产物**：sha256 `723e8eb9...`

### 迭代 4：/memory 节点修复

**诊断**：主线 DTB 的 `/memory@80000000` 是占位符 `reg = <... 0x0>`（沿用
下游"ABL 运行时填充"模式，但 picasso 的 ABL 不填）→ memblock 为空。

**改动**：runtime DTB 用 ABL 日志的 RAM Partitions 替换该 reg：

```
reg = <0x0 0x80000000 0x0 0x3cc00000>,
      <0x0 0xc0000000 0x0 0x40000000>,
      <0x1 0x00000000 0x1 0x00000000>;
```

**结果**：8 核 SMP 全部上线、6GB 内存识别，但停在 DMA pool。

**产物**：sha256 `0fea5e18...`

### 迭代 5：initcall_debug 定位 DMA pool 挂死

**改动**：`CONFIG_CMDLINE` 追加 `ignore_loglevel initcall_debug`。

**结果**：定位到 `dma_atomic_pool_init` 第二个 pool（GFP_DMA）内挂死。

**产物**：sha256 `3de8edcb...`

### 迭代 6：打点定位到 `arch_dma_prep_coherent`

**改动**：`kernel/dma/pool.c`、`arch/arm64/mm/dma-mapping.c` 加 pr_info 打点。

**结果**：

```
atomic_pool_expand: order=8 page=fffffdffc0080000   ← alloc_pages 成功
arch_dma_prep: va=ffff000002000000 phys=82000000 size=1048576
                                                    ← dc civac 挂死在这里
```

**产物**：sha256 `d6d938e0...`（粗打点）、`0306c3d1...`（细打点）

### 迭代 7：QTEE 安全区未保留（根因）

**诊断**：物理 0x82000000 落在下游固件的 **QTEE 安全区（0x81e00000–0x86000000）**
内——主线 DTB 的 reserved-memory 比下游 `lito.dtsi` 少 10 个固件区，DMA 区
分配器把 TZ 拥有的内存发给了内核，`dc civac` 向安全内存回写时 CPU 挂死。

**改动**：runtime DTB 补齐 10 个 no-map 区（值全部取自下游 `lito.dtsi`）：

| 区域 | 地址 | 大小 |
|---|---|---|
| sec_apps | 0x808ff000 | 0x1000 |
| qtee_apps | 0x81e00000 | 0x4200000 |
| pil_camera | 0x86000000 | 0x500000 |
| pil_npu | 0x86500000 | 0x500000 |
| pil_video | 0x86a00000 | 0x500000 |
| pil_cdsp | 0x87400000 | 0x1e00000 |
| pil_wlan | 0x8ba00000 | 0x200000 |
| pil_ipa_fw | 0x8bc00000 | 0x10000 |
| pil_ipa_gsi | 0x8bc10000 | 0x5400 |
| dfps | 0xa2300000 | 0x100000 |

**结果**：DMA pool 通过，RPMh regulator/SMMU 陆续 probe，但 ~30s 后**静默重启**。

**产物**：sha256 `48b61b0e...`

### 迭代 8：看门狗咬合（第一轮）

**诊断**：静默重启、无 panic 输出 = bootloader 启动的 APSS 看门狗到期咬合。
UEFI ABL 不像 LK 系在跳内核前停狗。

**改动**：`qcom_wdt_probe` 末尾无条件 `writel(0, WDT_EN)` + 打标记。

**结果**：仍重启；日志最后停在 GPU SMMU（3da0000）probe 的寄存器读取阶段。

**产物**：sha256 `1fd9ba4c...`

### 迭代 9：三修复（含一个 shim 地址 bug）

**诊断**：① 内核 wdt 驱动通过 `"qcom,kpss-wdt"` 回退匹配拿到 **kpss 布局**
（WDT_EN=+0x8），而 lito 实际是 **apcs_tmr 布局（WDT_EN=+0x40）**——上一轮
停狗写错寄存器；② GPU SMMU 在 GPU CX GDSC 未上电时读寄存器挂死，是咬合的诱因。

**改动**：
1. shim `el_drop` 内直接停狗（+0x8、+0x40 双写）——**初版地址构造错误**
   （`movz #0xc100`+`movk #0x17` 得到 0x17c100 而非 0x17c10000）；
2. `qcom-wdt.c` match 表加 `qcom,apss-wdt-sm7250/sm8250 → apcs_tmr 布局`；
3. runtime DTB 的 `iommu@3da0000` 加 `status = "disabled"`（GPU 留待后续）。

**结果**：**SCK 都没了** —— shim 在错误的 0x17c108 地址写 MMIO 挂死。

**产物**：sha256 `239bd26b...`（作废）

### 迭代 10：修正 WDT 地址（当前版本）

**改动**：`movz x2, #0xc100`+`movk #0x17,lsl#16` → **`movz x2, #0x17c1, lsl #16`**
（= 0x17c10000）。反汇编验证 `mov x2, #0x17c10000` + 两处 `str wzr`。

**状态**：待实机验证。

**产物**：sha256 `8bef9b6b...`

### 迭代 11：链接器回退 aarch64-none-elf-ld（构建环境变化，非源码改动）

**背景**：`/tmp/lld-extract` 被系统清理，ld.lld 不可用。`build-boot.sh` 改为：
ld.lld 缺失时自动回退 `LD=aarch64-none-elf-ld`（本树 `CONFIG_LTO_NONE=y`、无 CFI，
无必须 LLD 的特性；顶层 Makefile 的 RELR 旗标经 `ld-option` 探测自动适配）。

**现象**：命令行 LD 变化被 Kbuild 检测 → 重跑 syncconfig（`.config` 重写）：
`CONFIG_LD_VERSION=0 → 24501`、`CONFIG_LD_IS_LLD` 清除、`CONFIG_RELR`/
`CONFIG_TOOLS_SUPPORT_RELR` 移除（**当时误判"GNU ld 无 RELR 打包"**）→
vmlinux 重新链接。

> **2026-10-11 更正**：上述 RELR 结论是**探测脚本假象**。`tools-support-relr.sh`
> 用裸 `$CC -c -x c` 编探测 .o——LLVM=1 下 CC=clang 默认 **x86 target**，
> x86 .o 交 aarch64-none-elf-ld 报 "file in wrong format"，探测失败被解读
> 为"链接器不支持"。实际 GNU ld 2.45 **支持** RELR（GNU 语法
> `-z pack-relative-relocs`，实测 `.relr.dyn` 正常生成）。两树脚本已修复
> （探测 .o 显式 `--target=aarch64-linux-gnu`）→ `CONFIG_RELR=y` 恢复，
> GNU ld 版 Image 与 lld 版持平——历史的 +1.2MB 差异消除。
`.config` 其余关键项（串口四件套/CMDLINE_FORCE/INITRAMFS_SOURCE/非 EFI）全部完好。

**产物差异**（源码未变，仅链接器不同，功能等价）：

| 项 | 迭代 10（lld） | 迭代 11（GNU ld） |
|---|---|---|
| Image | 13244424 B | 14489608 B（+1.2MB：RELR 打包取消，`.rela.dyn` 全量保留） |
| boot.img | 18403328 B | 19648512 B |
| 内嵌 runtime DTB 偏移 | 跟随旧 Image（更靠前） | 0xfd3000（跟随更大的 Image） |

**校验**：manifest 全部 True；对最终 img 独立解析（header v2 字段与原厂头逐一
一致、dtb 字段逐字节等于原厂 blob、内嵌 Image/runtime DTB 逐字节等于构建产物、
ramdisk gzip 魔数）；反汇编确认 `mov x2, #0x17c10000` 停狗代码在位。

**状态**：待实机验证。**注意：本构建覆盖了迭代 10 的产物文件**；如需精确复现
迭代 10（sha256 `8bef9b6b...`），须重新提取 lld-21 并用 ld.lld 链接。

### 迭代 12：以 live_dts 为准的 DT 大修 + 时钟链打通（目标：UART 命令行）

**背景**：内核已能启动且 UART（earlycon）有输出，但始终到不了 shell——串口驱动
永远 deferred。以原厂运行时导出的 `live_dts.txt` 为准逐一核对，定位出**三层叠加**
的时钟链断裂，全部修复：

**根因 1（配置）**：`CONFIG_QCOM_CLK_RPMH is not set`——rpmh 时钟驱动根本没编进
内核！GCC 的 `bi_tcxo` 父时钟（RSC 的 rpmh-clk 子节点）无驱动注册 → GCC -517 →
geniqup/串口全链 deferred → 无 `/dev/ttyMSM0`。
修复：`scripts/config --enable QCOM_CLK_RPMH`（build-boot.sh 已加防回归检查）。

**根因 2（驱动缺表）**：主线 `clk-rpmh.c` 没有 sm7250 条目，`qcom,sm7250-rpmh-clk`
匹配不到驱动。且**不能**借 sm8250 表：lito 的 cmd-db 实测资源集（live dump +
`other_works_sm7250_mainline` 参考）为 `xo.lvl + lnbclka2/3 + rfclka1-4`，**无
lnbclka1、无 IP0 BCM**——而 clk-rpmh 缺资源是整体 `-ENODEV`（不是跳过）。
修复：内核 `drivers/clk/qcom/clk-rpmh.c` 增加 `clk_rpmh_sm7250` 表（照参考项目
的实机验证过的资源集）。

**根因 3（DT 错误，以 live_dts 实测为准）**：

| 节点 | 原值（错） | 实测（live_dts.txt） | 修复 |
|---|---|---|---|
| serial@888000 compatible | `qcom,geni-uart`（绑 ttyHS 高速驱动，无 console） | 下游即 console（`qcom,msm-geni-console`） | 改 `qcom,geni-debug-uart` → 注册 ttyMSM0 console |
| RSC power-controller | `qcom,sm7250-rpmhpd`（无驱动） | lito cmd-db 有 cx/mx/lcx/lmx/gfx/mss/ebi 级别，**无 mmcx** | 改 `qcom,sm7150-rpmhpd`（集合全在；sm8250 集含 mmcx 必失败） |
| regulators-1 (PM8150L) pmic-id | `"l"`（kona 惯例） | **PM8150L 资源前缀是 "c"**：`ldoc1-11/smpc2,7,8/bobc1` | 改 `"c"`，`ldol1`→`ldoc1` 等，probe 修复 |
| regulators-2 (PM7250B "b") | 存在 | cmd-db **无任何 smpb/ldob 资源** | 删除（probe 必 -ENODEV） |
| regulators-3 (pm8150a) | 存在（虚构 PMIC，picasso 无此件） | SPMI 实为 pm8150@0,1 + pm7250b@2,3 + pm8150l@4,5 | 删除；l7c(UFS vcc) 实为 PM8150L 的 `ldoc7`，并入 regulators-1 |

**禁用（防挂死/无驱动，bring-up 阶段）**：MDSS 整树（无 panel 驱动 + 无 MMCX）、
dispcc、gpucc（GPU CX 域未上电读寄存器挂死风险）、UFS + phy（主线无 sm7250
QMP-UFS-phy 驱动，保留节点待移植）。

**reserved-memory 补 3 处**（对照 live dump）：xbl_aop 区扩为 0x80600000+0x260000
（原漏 0x80600000-0x80700000）、补 xbl_uefi_log 0x80880000+0x14000、补
reserved_pil 0x8bc17400+0x3e8c00（原 0x8bc17400-0x8c000000 有洞）。
no-map 保留区 20 → **22 条**。

**UART 引脚核对**：live dump `qupv3_se2_2uart_active` = gpio36/37 + function `qup02`
+ drive-strength 2 + bias-disable，与现 DTS 的 qup-uart2-default **完全一致**，无需改。

**预期效果**（待实机验证）：cmd-db → rpmh-rsc → rpmh-clk(新表) → GCC → geniqup →
ttyMSM0 console 注册 → initramfs 的 `exec /bin/sh`（已核对 init 脚本）在 UART 出
交互 shell。

**产物**：sha256 `d11f14ebb6d87157ea88d6caacc018e64d761b98956f90c43027321c38c31fc0`
（Image 14503944 B，较上版 +14KB = 新增 rpmh-clk 表；第二次构建 sha256 复现一致）。

### 迭代 13：修 panic 两根因（x86 initramfs + pinctrl tiles）并收缩 rpmh-clk 表

**实机现象（迭代 12）**：interconnect 全部 probe 成功（cmd-db 可用）、RSC/populate 正常
（bcm-voter 在，qnoc 才能成功），但 GCC 仍 -517，pinctrl 最终 -EINVAL，然后
`Failed to execute /init (error -8)` → panic "No working init found"。

**根因 1：initramfs 是 x86-64 的**——`unpacked_boot_img/ramdisk` 里的 busybox 是宿主机
静态 busybox（x86_64, BuildID 45585f...），aarch64 内核 ENOEXEC(-8)；且 cpio 里
`/dev/` 为空（无 console 节点 → "unable to open an initial console"）。
修复：新 initramfs = **aarch64 busybox + musl**（取自
`/home/yucong/Documents/redmi2/boot_img/initrd-new`，已固化到 `scripts/initramfs/`）
+ `scripts/initramfs-init.sh`（mount proc/sys/dev + exec /bin/sh）+ `/dev/console`
（c 5 1）等节点。生成器 `scripts/make-initramfs.sh`（内核 usr/gen_init_cpio，
newc 格式、无需 root）。932KB / 44 项。

**根因 2：pinctrl-sm7250 移植照抄了 sc7180 的 north/south 双 tile 模型**——
6.17 msm_pinctrl_probe 在 `soc_data->tiles` 非空时走 `platform_get_resource_byname`
（按名字取 reg），而 lito TLMM 是单一区域（下游 dts：`reg = <0xf000000 0x1000000>`，
无名 reg）→ `res=NULL` → `devm_ioremap_resource` 报 "invalid resource (null)" -EINVAL，
tlmm 永远失败（串口的 pinctrl-0 因此不可用）。
修复：pin group 全部 `.tile = 0`、soc_data 去掉 tiles（走 index-0 单区域路径），
偏移 `0x1000*id` 与下游一致。

**附带：rpmh-clk sm7250 表收缩为 bi_tcxo/bi_tcxo_ao（仅 "xo.lvl"）**——迭代 12 的
15 时钟表含 lnbclka2/3、rfclka1-4（照抄参考项目），而 clk-rpmh 缺资源是整体
-ENODEV；qnoc 实测也缺 SN11/SN14/SH10/SH8，说明 lito cmd-db 比 kona 精简，这些
资源不保证存在。GCC 硬依赖只有 bi_tcxo（"xo.lvl" 必在——下游 rpmhclk 靠它工作）。
WiFi 阶段再按需增补。

**预期（待实机验证）**：rpmh-clk(bi_tcxo) → GCC → geniqup → serial@888000
（debug-uart + tlmm pinctrl 就绪）→ ttyMSM0 console → `Run /init` →
busybox shell 提示符。

**产物**：sha256 `f7871f727fd5aa73cc3ab5cf370751fca66b1e967cddbf2e7a6d55e44232324b`
（Image 内嵌 initramfs 已验证逐字节一致；19468288 B）。

### 迭代 14：GCC probe 静默复位修复（GDSC 旗标 + PLL 类型）

**实机现象（迭代 13）**：**RPMh 链全通**——rpmh-clk(bi_tcxo) probe 成功、sm7150-rpmhpd
的 8 个域（cx/cx_ao/gfx/lcx/lmx/mx/mx_ao/mss）全部从 cmd-db 解析成功（28ms，含真实
TCS 传输）、RSC probe 成功。GCC 开始 probe：5 个 GDSC（ufs_phy/usb30_prim/tbu_hf0/
hf1/sf0）全部注册返回 0，然后**无任何打印直接重启**（无声 = 硬件级事件，非内核 panic）。

**诊断**：GCC probe 全程唯一的 MMIO 写操作来自 GDSC（时钟注册只读 recalc）。对照
other_works_sm7250_mainline（lito 实测参考）与上游 gcc-sm8250.c，发现移植版两处偏差：

| 项 | 移植版（错） | 正确（other_works/kona 惯例） | 后果 |
|---|---|---|---|
| TBU GDSC 旗标 | `VOTABLE \| ALWAYS_ON` | TBU 仅 `VOTABLE`；ufs/usb 无旗标 | ALWAYS_ON → gdsc_init 强制 `gdsc_enable()` 给 MMU TBU 上电投票；VOTABLE 域投票后不轮询状态（probe 显示"成功"），TBU 岛父域/NoC 未上电 → 静默复位 |
| ufs/usb GDSC 旗标 | `VOTABLE`（非投票域） | 无旗标 | 对非投票域写投票位语义错误 |
| GPLL 类型 | FABIA + enable 0x52000 | **LUCID** + enable **0x52010**（bit n = GPLLn） | 布局不同 → rate 计算错（串口波特率会乱）；enable_reg 错地址 |

**修复**（`drivers/clk/qcom/gcc-sm7250.c`）：GDSC 旗标全部清零（与 other_works 一致，
gdsc_init 只写标准的 init-mask 位，与所有主线 qcom 设备相同）；GPLL0/6/7 改 LUCID
（lucid_vco 表 + `clk_alpha_pll_fixed_lucid_ops`）、gpll0_out_even 改
`postdiv_lucid_ops`（÷2 表 {0x1,2}）、enable_reg 统一 0x52010。

**若仍静默复位**（下一步 bisect）：把 desc 的 num_gdscs 置 0（完全不注册 GDSC），
再不行则裁剪时钟表——以此确认剩余嫌疑。

**产物**：sha256 `3c202e0a3e3e75bc7b66dddc6ae02f85cb4c164020ee6071c6f437c67fd08371`。

### 迭代 15：GDSC 注册整体下线（bisect 第一步）

**实机现象（迭代 14）**：清掉 GDSC 旗标（迭代 14）后**死点不变**——5 个 GDSC probe
完成后仍然静默复位。这说明杀机在两次构建**共有**的操作里：`gdsc_init` 的标准
init-mask 写（清 HW_CONTROL/SW_OVERRIDE + 写 wait 计数）**无论旗标如何都执行**。

**排查收窄**：
- regmap `max_register = 0x1f0000` 兜底 → 时钟注册循环（只读）**不可能**越界访问
  未映射内存（越界返回 -EINVAL），读已映射 GCC 寄存器也不会复位系统；
- `qcom_cc_drop_protected` 需 DT `protected-clocks`（无）→ no-op；
- **GCC probe 全程唯一的 MMIO 写 = 5 个 GDSCR 的 init-mask 写**；
- TBU 三连 GDSCR（0x7d050/0x7d058/0x7d054）与上游 kona **逐字节相同**（kona 实为
  4 个：0x7d050/54/58/6c），明显照抄——lito 的 GCC 布局未必一致，写错地址的
  "标准写"可能破坏关键寄存器。

**改动**（`gcc-sm7250.c`）：desc 注释掉 `.gdscs`/`.num_gdscs`——GDSC 注册整体
下线（本 DT 无任何活跃 GDSC 消费者：UFS/USB 已禁用、apps SMMU 无 power-domains，
下线零损失）。

**判读**：
- 若本次能打出 `Registered gcc-sm7250 clocks` 并继续 → GDSCR 写就是杀机 →
  下一步逐个加回（从最可信的 ufs_phy 0x77004 开始，TBU 三连最后/不加）；
- 若仍静默复位 → 嫌疑转向时钟注册本身或 GCC 之后第一个动作，抓完整日志再定。

**产物**：sha256 `f274e75134cc06912227f09239defd45602c20cd8ae3a2945b7096d9e5378f2b`。

### 迭代 16：逐时钟打印定位注册循环杀手（bisect 第二步）

**实机现象（迭代 15）**：GDSC 整体下线后死点**前移**到 `calling gcc_sm7250_init` 之后
（连 GDSC probe 打印都没有了）——证明 **GDSC 是无辜旁观者**，真凶在它后面的时钟
注册循环（13/14 里 GDSC 只因先运行而成为"最后说话者"）。

**推理修正**："regmap max_register 兜底所以读安全"有漏洞——max_register 只限地址
范围，不保证目标寄存器**有响应**；lito GCC 空间存在时钟门控/未实现的洞，读之会
让 AXI 总线停摆 → CPU 挂死 → 看门狗复位（与"卡住后重启"现象吻合，全程无打印）。
嫌疑：移植时钟表里某个偏移落在无响应区域。

**改动**（`drivers/clk/qcom/common.c`，临时调试打点）：
- `devm_clk_hw_register` / `devm_clk_register_regmap` 前各打印一行
  `qcom_cc: reg hw[N]/clk[N] <时钟名>`；
- rclks 循环结束后打印 `all clks registered, adding provider`、provider 注册后
  打印 `provider added`。

**判读**：死前**最后一行**的时钟（或紧随其后那个）即凶手——记下名字即可精确定位
偏移，修正或从表里剔除。

**产物**：sha256 `4c4aa56c07408a8296e5618b0c4f49b7b6dbeec5f3cce6662d35406aac1b2712`。

### 迭代 17：凶手落网——gpll7（kona 幻影 PLL）移除

**实机现象（迭代 16）**：逐时钟打印完美工作——死前最后两行：
`qcom_cc: reg clk[2] gpll6` ✓（0x13000 可读）→ `qcom_cc: reg clk[3] gpll7` →
总线挂死复位。**GPLL7 就是凶手**。

**根因**：移植表里的 gpll7 @ **0x27000 是 kona 的地址**，lito 没有这颗 PLL
（other_works 参考无 gpll7，只有 gpll9@0x1c000）。注册时 recalc 读 0x27000——
无响应地址把 AXI 总线挂死，看门狗复位，全程零打印。

**改动**：删除 gpll7（struct/enum P_GPLL7_OUT_MAIN/数组项）。它没有任何
parent map 或 DT 消费者引用，干净移除。lito 如需额外 PLL 按下游源补 gpll9。

**调试打点保留**（common.c 逐时钟打印）——表里可能还有其他幻影偏移
（ours-only 偏移清单：0x09ffc/0x0b00x/0x6a004/0x6d00x/0x71008/0x750xx/0x8c034/
0x8c060/0xb024/0xf0xx/0x170xx-0x186xx RCG 系列等），下一轮注册若有第二个凶手，
打印会直接点名，逐个歼灭。

**产物**：sha256 `8018901910b49ad3b5744f916e1a3191154b5482a146199479c01404bf4fcbd5`。

### 迭代 18：clk_ignore_unused（目标：shell）

**实机现象（迭代 17）**：**时钟表全量注册通过**（gpll7 是唯一幻影）、interconnect
全 probe、**geniqup + ttyMSM0 全功能 console 注册成功**（earlycon 退役）——
串口链全线贯通！死点变为 `clk: Disabling unused clocks` 之后重启。

**根因**：late initcall `clk_disable_unused` 把无使用者的分支时钟关掉。移植表一个
`CLK_IS_CRITICAL` 都没有（上游惯例：各子系统 AHB 配置时钟标 critical，防止
cont-splash/ABL 留开的硬件被断时钟；other_works 参考就是这么做的）。关掉某个
关键 AHB 分支（QU P wrap / ABL 留开的子系统）→ 硬件死亡 → 看门狗复位。

**修复**：内核命令行加 **`clk_ignore_unused`**（跳过全部 unused 关闭，bring-up
标准做法）。后续按上游惯例给 AHB 配置时钟补 critical 后可移除。
同时移除已完成使命的逐时钟调试打印（common.c，迭代 16）。

**遗留观察**（非阻塞，shell 之后处理）：
- `gcc_qupv3_wrap0_s2_clk_src: rcg didn't update its configuration` WARNING——
  console set_rate 时 RCG UPDATE 不清零（偏移存疑或 hw-ctl；串口输出正常，
  实际波特率沿用 ABL 配置）；
- `f000000.pinctrl` 仍 -517（死循环 defer，原因未明；UART 引脚沿用 ABL 配置
  所以串口不受影响；查 PDC/wakeup-parent 链）；
- qnoc 缺 SN11/12/14/15/SH8/SH10 BCM 地址（警告，非致命）。

**产物**：sha256 `b2979c79a18802835f9e33412c71d21def70e80ce9b84f05f1e7f157f2875586`。

### 迭代 19：禁用 interconnect + 修 gpio-ranges 自引用（纯 DT）

**实机现象（迭代 18）**：`clk_ignore_unused` 生效（`Not disabling unused clocks`）、
genpd/regulator late initcall 全过，死点推进到 **`of_platform_sync_state_init`**。

**根因 1（ICC sync_state）**：`of_platform_sync_state_init` 触发所有 ICC provider 的
sync_state → qnoc 给全部 BCM 提交带宽投票（RPMh TCS 批量写）。移植的 qnoc-sm7250
表里有 **6 个本机 cmd-db 无地址的 BCM**（SN11/12/14/15、SH8/SH10），且
`qcom_icc_bcm_init` 的 -EINVAL 在 probe 处被忽略（icc-rpmh.c:276 无错误检查）——
addr=0/零带宽投票冲进 RPMh → 复位。
**修复**：runtime DT 禁用全部 interconnect 节点（10 qnoc + osm-l3；当前无任何
消费者）。UFS/USB 阶段修好表（删缺失 BCM、对照下游核对 node 表）再启用。

**根因 2（pinctrl 永久 -517）**：tlmm 节点的 `gpio-ranges = <0x3b 0 0 146>` 的
0x3b 是**它自己的 phandle**——gpiochip 注册时要等自己的 pinctrl 设备 → defer 死
循环（上游 sm8250.dtsi 的 tlmm 无此属性）。PM8150L 的 SPMI gpio 同病
（`<0x3e 0 0 12>` 自引用）。
**修复**：两处 `gpio-ranges` 删除（runtime DT + 树 dts 已同步）。

**产物**：sha256 `7bed931c15d9e78b932ef8c6d3e9e06e15d5a8a1fde402b5e085282cadef0605`
（--skip-kernel 构建，内核沿用迭代 18 的 Image）。

### 迭代 19：✅ 实机验证成功——UART 交互 shell 达成（阶段目标完成）

**实机结果**：`Run /init` → init banner → `/bin/sh: can't access tty; job control turned
off`（预期，此 busybox 无 cttyhack）→ **`~ #` 交互提示符**，内核持续运转（crng init done）。

**里程碑**：ABL → copydown shim → 主线 6.17 → cmd-db/rpmh-clk/rpmhpd/GCC → geniqup →
ttyMSM0 全功能 console → initramfs busybox shell，**全链路贯通**。

遗留小项（不阻塞）：qup s2 RCG 的 set_rate WARNING（改波特率前必须修，当前波特率
沿用 ABL 配置正常）；`can't access tty` 提示（cttyhack 缺失，仅无作业控制）。

### 迭代 20：显示 bring-up——dispcc/DPU 6.1/LCDB 从下游内核移植补齐（待实机验证）

> **完整技术文档**（显示链全阶段：移植方法/gcc 幽灵偏移/regulator/
> pinctrl 大战/simplefb 弯路/冲刺六刷，含方法论）：
> [display-bringup.md](display-bringup.md)

**背景**：UART shell 已通（迭代 19），本迭代目标 = 屏幕点亮（企鹅 logo + fbcon 内核
日志 + tty0 shell）。四大缺口全部从 `Xiaomi_Kernel_OpenSource-picasso-r-oss/`
（Android R 4.19 下游内核）提取/移植：

| 缺口 | 下游源码 | 移植产物 |
|---|---|---|
| dispcc | `drivers/clk/qcom/dispcc-lito.c` | `drivers/clk/qcom/dispcc-sm7250.c`：45 时钟偏移原样保留，API 主线化（parent_data/去 vdd_class/GDSC 折入 gdscr=0x3000 HW_CTRL）；PLL0 LUCID@0x0（L=0x35→1035MHz） |
| DPU catalog | `techpack/display`（lito=SDE_HW_VER_610 实锤） | `catalog/dpu_6_1_sm7250.h`：偏移=live_dts 实测-0x1000；4ctl/2vig+3dma/4lm/2dspp/4pp(merge-3d@0x3000,0x3100)/2dsc/4intf/wb/cdm；+ dpu_kms/msm_mdss 匹配条目 |
| UBWC | lito-sde：ubwc3.0/swizzle6/hbb1/macrotile | `drivers/soc/qcom/ubwc_config.c` 加 `qcom,sm7250`——不加则 `qcom_ubwc_config_get_data()`（按根 compatible 匹配）-EINVAL，DPU probe 必死 |
| LCDB（LCD bias） | `drivers/regulator/qpnp-lcdb-regulator.c`（2323 行） | `drivers/regulator/qcom-lcdb-regulator.c` 裁剪版：模块使能（0x46 bit7+0x10 轮询）+ldo/ncp 电压（0x71/0x81）+BST 联动（0x41，+150mV headroom）；裁掉 PM660L WA/TTW/休眠/SC IRQ |
| 面板供电 | `dsi_panel_pwr_supply_avdd` | `panel-novatek-nt36672c.c`（树内实为 G7A 生成驱动，本已 CONFIG=y）补 regulator_bulk + 上电时序（vddio→1ms→avdd/avee→10ms） |

DSI 走主线现成链：ctrl=`qcom,mdss-dsi-ctrl`（6G v2.4→sdm845_dsi_cfg，io 0xae94000
精确匹配，需 vdda+refgen 供电）、PHY=`qcom,dsi-phy-7nm`（v4.1，需 vdds）。

**DT 修正（runtime dts + 树 dts 同步）**：refgen **0x88e7000→0xff1000**（live 实测，
旧值是 kona 地址）；mdss_mdp/dsi/dispcc 域 **MMCX→CX**（sm7150 集无 MMCX）；phy
compatible 改通用；**WLED/LCDB 移入 pmic@5**（下游实测 USID 5，放错 USID 写入无
响应；旧 DT 的 backlight 挂在 spmi 总线下是错的）；l1a→1.896V（面板 vddio）；
dispcc 节点改 sm8250 式输入 + gcc 补 `gcc_disp_gpll0_clk_src`（lito：AON 0x52000
bit20，ID=160）；panel@0/DSI/PHY 补全供电；mdss/dispcc 解除禁用。

**内核配置新发现**：`CONFIG_SPMI` 整链未开——**PM8150L 子树（gpio/wled/lcdb）
此前从未 probe 过**。已补开 SPMI/SPMI_MSM_PMIC_ARB/MFD_SPMI_PMIC/
PINCTRL_QCOM_SPMI_PMIC + SM_DISPCC_7250 + REGULATOR_QCOM_LCDB。

**initramfs**：/init 在 tty0（fbcon）后台 spawn shell（触摸未 bring-up 无输入源，
仅显示侧验证；交互仍在 UART）。

**预期链路**：SPMI→pmic@5→LCDB/WLED；dispcc→GDSC→mdss（`dpu hardware revision:
0x601...`）→DSI→panel attach→DSC PPS→video burst→fbcon logo/log/tty0 shell。

**实机验证（第一轮，uart_early.log）**：LCDB/WLED/dispcc/mdss_gdsc 全部 probe
成功，但 **mdss_runtime_resume 无限 -EBUSY 重试（21 次）**：`gcc_disp_hf_axi_clk
status stuck at 'off'` → `Failed to enable clk 'bus': -16` → mdss probe 永远失败
（DSI/panel 全被拖死）。现象：屏幕一闪（GDSC 上电瞬间）后黑屏、背光常亮。

**根因（gpll7 翻版——kona 幽灵偏移）**：我们的 gcc-sm7250.c 里
`gcc_disp_hf/sf_axi_clk` 用了 kona 偏移 0xb020/0xb024，**lito 实际 0xb030/0xb034**
（gcc-lito.c 实锤）——写入无效地址，halt 位永不变 on。

**gcc 全量审计（对照 gcc-lito.c 同名时钟逐一比对 72 个，59 个偏移不符！）**：
全部按 lito 修正。受影响的还有：全部 QUP wrap0/1 SE RCG（**这解释了迭代 17 的
`rcg didn't update its configuration` 警告——UART RCG cmd_rcgr 也指错了地址**，
0x17290→0x17270）、SDCC1/2（整套在错误区域 0x750xx/0x130xx）、UFS 全家、USB
全家、prng/video/gpu/npu 分支。审计脚本归零确认。唯一 lito 没有的
`gcc_usb_phy_cfg_ahb2phy_clk`（kona 0x6a004）留待 USB 阶段核对。

**实机验证（第二轮，uart_early.log）**：gcc 偏移修正生效——**mdss/DPU probe
成功**（`ae01000.display-controller returned 0`，mdss_gdsc/dispcc/spmi arbiter v5
全部起来）！但链路卡在新的两处：

1. **rpmh-regulator 两组注册失败（-131）**：`vreg_l2c_3p0/vreg_l1a_1p896:
   failed to get the current voltage: -ENOTRECOVERABLE` → regs-0/1 整组失败 →
   l9a/l5a/l1a 全缺失 → **DSI/PHY/panel 全 -517**。
   根因 A（类型错配）：主线 rpmh-regulator 里 **ldoa1 = pmic5_nldo（上限
   1.328V）**，装不下 1.896V；**ldoc2 = nldo** 装不下 3.008V。machine_constraints
   的 apply_uV 路径 set 后 map 不中线性范围 → get 仍 -ENOTRECOVERABLE →
   devm_regulator_register 拒绝。
   根因 B（**命名陷阱**）：live_dts 里 1.896V 的 "pm8150a_l1" **父块其实是
   rpmh-regulator-ldoc1**（小米 R 树命名误导；4.19 picasso overlay 改的正是
   &L1C 佐证）——面板 vddio 实为 **PM8150L ldo1**（主线 pldo_lv 1.504-2.0V@8mV，
   1896000 精确命中）。
   修复：vddio 改挂 l1c@1.896V；l1a 复原 0.752V；l2c 改 1.144V（nldo 范围内，
   无消费者）。**此前迭代 regs-0/1 可能一直静默失败**（l1a=0.752 在 nldo 内
   可过，l2c=3.008 超 nldo——regs-1 组从移植起就没成功过，无人消费故无感）。
2. **`gcc_qupv3_wrap_0_s_ahb_clk status stuck at 'off'` → 's-ahb': -16**：
   第一轮审计只对了寄存器**地址**，没对 AON 投票**位**与 hwcg 字段——
   wrap0 m/s_ahb 的投票位 bit2/bit3 是错的（lito 为 **bit6/bit7**，写错位=
   投票了别的时钟），且缺 hwcg_reg/hwcg_bit（主线 clk-branch 在 HW gating
   模式下靠 hwcg 跳过 halt 轮询）。第二轮全面审计（halt_check/enable_mask/
   hwcg）修正 36 处（qup AHB 投票位+19 处 hwcg+HALT_SKIP/VOTED/DELAY 类型）。

**实机验证（第三轮）**：巨大进展——**面板链路真实跑通**（LCDB 偏压上电点亮
裸面板=白屏，随后 dsi probe 失败断电=黑屏、背光亮）。regs-0 组成功
（vdda/vdds 就绪，DSI host probe 推进到 assigned-clock-parents 阶段）。剩两个
小问题：

1. **regs-1 仍 -131（乌龙）**：l2c 值写成 0x1179D0=1,145,296（**十六进制换算
   错误**，非 8mV 步进点；1,144,000 应为 0x117730）→ l1c（面板 vddio）缺失 →
   panel/dsi -517。
2. `clk: failed to reparent disp_cc_mdss_byte0_clk_src to dsi0_phy_pll_out_byteclk: -22`
   ——来自 defer 轮次的 assigned-clock-parents 处理（of_clk_set_defaults 在
   -517 设备上也会跑），上游 sm8250 同款 DT 写法可正常工作；等 dsi 最终 probe
   轮（PLL 已在）应消失，暂不处理。

修复：l2c → 0x117730（纯 DT）。

**实机验证（第四轮）**：与第三轮表现相同——l2c 仍是 -131：`Setting
1144624-1144624uV`。**教训**：连续两次手算十六进制出错（1,144,000 = 0x1174C0，
先后误写 0x1179D0/0x117730）。改用 python 换算并全面复核 runtime dts 手写
数值：l2c→0x1174C0 ✓、l1c 0x1cee40=1,896,000 ✓（pldo_lv 步进命中）、
LCDB 0x53ec80=5,500,032→顺手改精确值 0x53EC60（对驱动无影响，内部按 mV 取整）。

**实机验证（第五轮）**：**🐧 企鹅出现了**（`Console: switching to colour frame
buffer device 135x150`——simpledrm 接管 ABL 帧缓冲出图），随后黑屏 = mdss probe
顶掉 simpledrm（aperture 移除冲突帧缓冲）而 msm 完整输出还差最后一步。链路状态：
regs-0/1 全通、DSI PHY/显示控制器 probe 成功；**dsi/spmi/pinctrl/tsens 仍在
defer**。

**根因（终极）**：`CONFIG_QCOM_PDC is not set`——**pdc 驱动根本没编进内核**！
pdc 是 wakeup 中断域（spmi 的 periph_irq、pinctrl 的 wakeup-parent、tsens 的
中断都挂在它下面）。pdc 域不存在 → `of_irq_get()` 的 `irq_find_host()` 返回
NULL → -EPROBE_DEFER 永久循环 → spmi/pmic@5/LCDB 永远起不来 → panel 缺
avdd/avee → dsi defer。**这也是历史遗留"pinctrl 永久 -517"的真因**
（gpio-ranges 修复后仍 defer 的另一半）。qcom_pdc_init 所有失败路径都有
pr_err 而日志全无——因为 init 根本没被调用（驱动未编译）。

修复：`CONFIG_QCOM_PDC=y`（顺带说明：spmi 首次 probe 打印 "PMIC arbiter
version v5" 后 returned 517——defer 点在 bus_init 的 `of_irq_get_byname
("periph_irq")`，与版本打印先后吻合）。

**实机验证（第六轮，PDC 开启后无法启动）**：PDC 生效——**spmi 全链路通了**
（pmic@4/@5、gpio@c000 全部 probe 成功），但暴露三个新问题：

1. **启动挂死**（deferred_probe_initcall 里 power-domain probe 后无输出=MMIO 挂
   死，watchdog 无 reset 输出）：凶手是 **pinctrl**（首次真正 probe）。
   **迭代 13 的"单区域"修复是错的**：下游 pinctrl-lito.c 实为 **4-tile 布局**
   （NORTH=0/SOUTH=0x100000/WEST=0x500000/EAST=0x900000），且 146 脚的 tile
   归属与我们的移植完全不同（如 gpio0 下游=EAST，我们错标 SOUTH）——单区域
   模式下全部寄存器偏移错乱，读挂。当时 -EINVAL 的真因是 DT 缺具名 reg 段。
   修复：从下游提取全部 146 脚 tile 归属重写 `.tile`，恢复 tiles 数组
   （north/south/west/east）+ DT 4 段具名 reg。lito 实际只用到
   south(26)/west(65)/east(55) 三个 tile。
2. **`lcdb@ec00 returned 22`**：`missing reg resource`——SPMI 子节点在
   `#size-cells=0` 下 **reg 必须单 cell**（`<0xec00>`；两 cell 被解析成两个
   无长度 reg 项，地址翻译失败）。gpio@c000 单 cell 所以一直正常。
3. **`backlight@d800 returned 1`**：同上 reg 问题（WLED 拿到 start=0x100，
   写 0x152 被 arbiter 拒："disallowed SPMI write to sid=5, addr=0x0152"）
   + 主线 qcom-wled **不允许同时设 num-strings 与 enabled-strings**（删前者）。

**实机验证（第七轮，66778ed4）**：企鹅再现后重启（挂死在 deferred_probe 的
power-domain 后，疑似看门狗复位）。对照主线 pm8150l.dtsi 惯例修正三处 +
全面审计：

1. **WLED reg**：主线标准 `reg = <0xd800>, <0xd900>`（ctrl+sink **两个地址段**，
   #size-cells=0 下 reg 是地址列表不是 addr+size）——此前 `<0xd800 0x100>` 把
   sink 指到 0x100（"disallowed write addr=0x0152" 全对上）；单 cell 又缺 sink。
2. **LCDB reg**：改 `<0xec00>, <0xed00>` 两段（规避单段 resource 之谜；驱动用 [0]）。
3. **SDC pingroup**：下游在 **WEST tile + 0xb8000**（绝对 0x5b8000），我们错写
   NORTH+0x9a000。UFS_RESET(0x9f000) 下游无参照（UFS 禁用中，记入遗留）。

**全面审计（按用户要求，对照下游内核/dts + 主线惯例）**：

- **审计 A（DT compatible vs 内核驱动覆盖）**：扫 runtime DTB 全部 80 个
  compatible 对照 Image 字符串，命中 4 个真遗漏并已开启：
  **`CONFIG_REGULATOR_QCOM_REFGEN`（关键！DSI 的 refgen-supply 非可选，
  缺了 dsi 永远 -517）**、`CONFIG_POWER_RESET_MSM`（qcom,pshold 重启）、
  `CONFIG_HW_RANDOM(_QCOM)`（prng-ee）、`CONFIG_KEYBOARD_GPIO`（gpio-keys）、
  `CONFIG_HWSPINLOCK(_QCOM)`（tcsr-mutex——此前 smem 一直无锁裸奔）。
  其余 MISSING 均为 fallback compatible 已覆盖（qcom,sm7250-* 第二字符串）。
- **审计 B（节点地址）**：tsens/watchdog/aoss/llcc/refgen 等均直接源自
  live_dts 真值，与下游一致（低风险）。
- **挂死隔离试验**：**tsens 两个节点暂禁用**（pdc 打通后首次获 probe 资格、
  SROT 可能 TZ 锁，头号嫌疑）；**pinctrl probe 加打点**
  （`sm7250_tlmm: probing/done`——下轮日志若见 probing 后挂死即凶手）。

**实机验证（第八轮，d24643e1）**：WLED probe 成功（reg 段修复生效，仅缺
ovp/short 中断 ENXIO 警告）；tsens 排除；**打点命中真凶——日志最后一行
`sm7250_tlmm: probing` 后挂死**（msm_pinctrl_probe 内部）→ 看门狗重启。
LCDB 两段 reg 仍 -22（同结构 wled 可取 resource 而 lcdb 不可取，机制之谜
未解）。

修复：
1. **msm_pinctrl_probe 三阶段打点**（regs mapped → msm_gpio_init → done）
   二分定位挂点（下轮日志直接看到挂在 gpiochip 还是 pinctrl 注册）。
2. **LCDB 改为 DT 直读 reg**（`of_property_read_u32(np, "reg")` 绕过
   platform resource 机制——SPMI 孙节点 #size-cells=0 下不可靠）。

**实机验证（第九轮，eb2f6844）**：三阶段打点二分——`regs mapped ✓ →
registering pinctrl ✓ → calling msm_gpio_init → 死`。**挂在 msm_gpio_init**。
对照 lmi 7.1 树（pinctrl-msm.c 核心逻辑与我们 6.17 一致，排除版本差异）+
审计 wakeirq 路径后定位：`devm_gpiochip_add_data` 内部会 request TLMM 汇总
中断（SPI 208）——**ABL 遗留的 GPIO 中断使能+pending 状态在使能汇总中断的
瞬间引发风暴**（中断吞掉 printk，与"打点后死无对证"完全吻合）。

修复：
1. **gpiochip 注册前清全部 146 脚中断**（intr_cfg 写 0 + intr_status 写 ~0U
   W1C，msm_gpio_init 内 bring-up 循环 + 打点 `gpio intrs cleared`）。
2. **TLMM 移除 wakeup-parent**（bring-up 简化：PDC wakeirq 映射机制是 5.x
   加入而下游 4.19 没有 lito 数据可抄——我们 map 的 71 对是从 kona 抄的，
   正确性存疑；GPIO 中断直走 GIC 汇总即可工作，深度睡眠唤醒后续再补）。

**实机验证（第十轮，132da978）**：`gpio intrs cleared` 未出现——清中断循环
在 **gpio 0（tile 3=east，cfg off 0x8）** 写挂（打点最后一行直接暴露）。
即**推断的 east 段基址 0xf900000 无响应**（虽然 = 0xf000000+下游 EAST 偏移
0x900000，但独立段映射方案被实机否定）。

**最终方案（回归下游实证的访问算式）**：放弃主线多 tile 段机制，改用
**单段映射 + tile 偏移并入寄存器偏移**（= 下游 4.19 的同一算式，Android
实证可用）：
- 驱动：`NORTH/SOUTH/WEST/EAST` 改回偏移宏（0/0x100000/0x500000/0x900000），
  宏体 ctl/io/intr_cfg/intr_status/intr_target 全部 `_tile + 原`，`.tile=0`，
  soc_data 去 tiles；
- DT：TLMM 恢复原厂单段 `reg = <0xf000000 0x1000000>`（去 reg-names）；
- SDC（WEST+0xb8000）、UFS_RESET（NORTH+0x9f000）同步绝对化。

**实机验证（第十一轮，58c6e2f9）+ 用户纠偏**：每脚打点实锤——**gpio0
（intr_cfg 0x900008，EAST tile）写挂**。live_dts 决定性证据：
`onewire_gpio` 节点 `gpio_number=0x42(66)` + `cfg-addr=0xf942000` 精确吻合
`EAST(0x900000)+0x1000×66`——**下游布局公式与 tile 宏完全正确**；触摸
reset 用 gpio4（Android 常态读写）→ EAST 头部非整片洞，**仅 gpio0-3 槽位
无响应**（疑 TZ/安全侧保护；Android 从不访问故无感）。

**用户指示（本轮核心）**：对齐下游行为——Android 的 pinctrl probe **从不
主动触碰任何引脚寄存器**（完全惰性：mux/conf 仅在 pinctrl state 应用时写、
中断 handler 只扫 `enabled_irqs` 位图中已启用的引脚）。gpio0-3 无人使用
（qup01 全禁用）就永远不会被访问。**停止纠结该区为何不可写**。

**全量回滚临时修改**：
- 删除 pinctrl-msm.c 的"清全部引脚中断"循环（自作聪明的风暴假说产物，
  正是它第一个撞上 gpio0 鬼区）、三阶段打点、每脚打点、flags 声明；
- 删除 pinctrl-sm7250.c probe 打点；
- **tsens 恢复启用**（挂死隔离试验结束，凶手已定位）；
- 主线多 tile 段机制在第十轮已回滚（DT 单段 + tile 偏移并入寄存器偏移
  = 下游 4.19 同一算式，本轮维持）。

**tlmm-verify 工作流结论（3 agent 并行取证，全部证据归档
docs/tlmm-wakeirq-map.md）**：
- 下游 146 脚 PINGROUP 的 index==id 全量核对 **0 错位**，tile 宏/公式正确
  （onewire 0xf942000=gpio66 精确吻合）；**gpio0/3/4/5 是 PDC direct-connect
  唤醒脚**（尾参 0xB4000）——gpio0 intr 寄存器疑被 PDC/TZ 接管（写挂的
  合理解释，按用户指示不再深究）；
- 抓到并修正 **2 个真实偏移错误**：UFS_RESET 0x9f000→**WEST+0xbc000**、
  SDC2 0x98000→**SOUTH+0xb7000**（下游绝对 0x5bc000/0x1b7000）；
- 提取 **87 条 gpio→PDC 官方 wakeirq 映射**（live irqdomain-map）归档备用；
- other_works 实机移植**完全没有 TLMM**（引脚全靠 ABL 预配置）——反向
  印证"不碰无主引脚"的惰性路线。

**实机验证（第十二轮，adfd486b）**：**直接无法启动、屏幕全黑**——日志仅
419 行且无 pinctrl probe 返回行：wakeup-parent 已删后 pinctrl **第一次
probe 即真跑并当场挂死**（此前各轮先 -517 defer、几百行 initcall 含
simpledrm 企鹅照常执行后才挂，故有画面；这次在 initcall 早期就死，企鹅
也没了）。**回滚无效 = msm_gpio_init 内部另有元凶**。

**真凶（铁证）**：6.17 `gpiolib.c` 的 `gpiochip_add_data()` 会**对每个
valid 引脚调用 `gc->get_direction()`（读 ctl_reg）**初始化 FLAG_IS_OUT：
```c
if (gc->get_direction && gpiochip_line_is_valid(gc, desc_index))
    assign_bit(FLAG_IS_OUT, &desc->flags, !gc->get_direction(gc, desc_index));
```
gpio0 第一脚即读 0xf900000 → 挂死。**下游 4.19 的 gpiolib 没有这个逐脚
初始化循环**（新内核行为），所以 Android 从不读死区——之前的"惰性"推论
对下游成立、对 6.17 不成立。

**修复（第十三轮）**：`gpiochip_line_is_valid` 正是开关——给 soc_data 加
可选 `init_valid_mask` 回调（pinctrl-msm.h/c 支持覆盖默认），sm7250 提供
`sm7250_init_valid_mask`：默认逻辑 + **清 EAST 头部 gpio0-5**（实测死区，
0/3/4/5 为 PDC direct-connect 唤醒脚疑 TZ 接管；无消费者——qup01 全部
disabled 已核对）。valid_mask=false 使 gpiolib 逐脚初始化、gpio request、
irq 分配**全层跳过**死区。触摸 bring-up 需 gpio4 时再精确放开。
（残余路径：pinmux state 应用不受 valid_mask 挡——无消费者已核对安全。）

**实机验证（第十三轮，6831bfcc）**：表现与十二轮完全相同——**修复无效**。
复盘：上轮把 soc 回调挂在了 `girq->init_valid_mask`（gpio_irq_chip，**irq 层**
掩码），而逐脚 get_direction 循环检查的是 `gpiochip_line_is_valid()` →
`gc->gpiodev->valid_mask`（**gpio 层**掩码）——**两套独立开关**；且 gpio 层
掩码为 NULL 时"全部 valid"（gpiolib.c:808），只有设置 `chip->init_valid_mask`
才会分配。教训：修复必须核到目标代码行的完整调用链，不能只对字段名。

**修正（第十四轮）**：`msm_gpio_init` 中同时
`chip->init_valid_mask = pctrl->soc->init_valid_mask`（soc 回调签名改 int 对齐
gpio_chip 定义）。三环调用链已逐环核对：pinctrl-msm.c:1470（挂载）→
gpiolib.c:738-756（`gpiochip_init_valid_mask` 分配 valid_mask 并调回调清
gpio0-5）→ gpiolib.c:1171（`gpiochip_line_is_valid` 挡住死区脚的
get_direction 读）。

**实机验证（第十五轮，b69356dd）+ 全链路打点判读**：fix14 的掩码**生效**
（`GDBG: valid_mask init ret=0 have_mask=1`，逐脚循环从 gpio6 开始，0-5
被跳过）；gpio6→58 **全部读通**（含 EAST 11-45、WEST 46-58），**gpio59
打印后死**——挂点 = gpio59 的 ctl_reg 读（0xf53b000，WEST 槽位）。

**下游 gpio 行为系统审计（用户要求，本轮核心）**：

1. **4.19 gpiolib 注册零硬件访问**：`gpiochip_add_data` 的逐脚循环只做
   `desc->flags = !chip->direction_input ? (1<<FLAG_IS_OUT) : 0`（纯软件）；
   `get_direction`（读 ctl_reg）在 `gpiod_request_commit` 里**单脚惰性调用**
   ——Android 只读它 request 的脚。6.17 把这个读搬进了注册循环遍历全部
   valid 脚，这是行为差异的根源（不是"哪个 tile 算错"）。
2. **live_dts 全量解析**（197 个 pinctrl 状态 + 全部 gpio 引用属性，含
   启用状态判定）：**73 脚白名单**（启用消费者引用；剔除 tx_swr_ch_map/
   lut-col-legend/clocks 等数据数组伪匹配 3/26/92/93/95）。
   - **gpio59-62 仅被禁用的 SE6（i2c/spi@980000）引用**——死槽与
     "Android 不用"完全吻合；同批禁用外设 SE1(1,2)/SE8-uart(51,52)/
     SE9-spi(48)/SE10(53,54)/sdc-cd(69)/dp(114) 同理
   - **gpio0 = cci1 相机 sensor 供电脚（启用）**——ctl 读 Android 实证；
     此前"EAST 0-5 死区"结论修正：挂死的是清中断循环对 gpio0
     **intr_cfg 的写**，ctl 读从未在 0-5 上测过
   - gpio4/5=触摸 reset/irq、gpio10/11=DSI TE、gpio55=G7A ESD irq、
     gpio46/47=SE9 i2c 等全在白名单（后续 bring-up 直接可用）
3. **msm 模板 4.19 就有 get_direction**——白名单内每脚 ctl 都被 Android
   的 request 路径读过，读安全性有实证。

**修复（fix15）**：
1. `sm7250_init_valid_mask` 改为**白名单**（`sm7250_used_gpios[]` 73 脚，
   bitmap_zero + set，程序化校验与审计集合双向零差）——get_direction
   循环、gpio request、irq 分配只碰 Android 实证可访问的脚；
2. `qup02_groups` 补 gpio36/37（下游 4 脚 34-37；uart 状态在 36/37 上
   应用 function "qup02" 原会 lookup 失败）；
3. 回滚本轮临时的 `no_get_direction` 通用代码（白名单覆盖其职责）。
4. runtime DT 消费者核对：唯一启用的 TLMM 消费者 = serial@888000
   （gpio36/37）+ 面板 reset-gpios（tlmm gpio46），均白名单内；其余
   QUP/SDC 全 disabled（gpio59 的 qup-i2c6-default 状态无消费者不会应用）。

**fix15 产物**：sha256 `6c15200227e6a93fdbdfbd42c629c89bedee8570fad77849257a440e7a0e2f4e`。

**实机验证（第十六轮，6c152002）**：白名单生效但 **gpio0 在白名单内被读、
ctl 读挂死**——"启用的数据属性消费者 ⇒ 槽位可读"推断被证伪（gpio0 是 PDC
direct-connect 脚，疑 TZ 接管；相机消费者引用不代表 Android 真 request 成功）。
教训：**槽位可访问性无法从 dts 推断，只有 pinctrl 状态应用（set_mux 写）是
强证据，probe 阶段干脆一个都不读**。

**修复（fix16）**：恢复并保留 `soc_data.no_get_direction`（6.17 注册循环以
`gc->get_direction` 非空为门，置 NULL → else 软件分支 → **probe 零引脚
寄存器访问**，与下游 4.19 注册行为完全一致）；gpio0 从白名单剔除（72 脚）。
get_direction 语义保留给单脚 request 路径（下游同款惰性）。

**fix16 产物**：sha256 `adbfee4a2b0f46c1822b3cf62a5a816ab228c66caa7b12ab0364c899f1fd684e`。

**实机验证（第十七轮，adbfee4a）**：**pinctrl 彻底打通**（probe 零引脚访问，
`msm_gpio_init done ret=0`，PM8150L SPMI gpiochip 也注册成功）→ dsi/panel
解除 defer → **DPU hw revision 0x60010000、dsi bound、Initialized msm 1.13.0**
——显示链全部起来！新问题两个：

1. **uart 状态应用 WARN**（pinctrl-msm.c:203 `i == g->nfuncs`）：qup02 加进
   函数组表后，gpio36/37 引脚侧 funcs 数组没有 qup02——**逐脚 funcs 表根本
   不是从下游 lito 提取的**（机械 diff 全表：qup04 错含 37、mdp_vsync 缺
   12/70/71、qup01 缺 12/37、qup10 缺 63/70/71 等 25+ 处不一致；tile 归属
   反而零偏差）。serial probe -22（当前串口输出全靠 earlycon）。
2. **首次 modeset 静默挂死**：日志止于 `adreno_load_gpu` 的 "no GPU device
   was found"（fbdev 首开触发 modeset），之后 panel prepare/DSI 上电/
   PHY/video 全无打印 = 挂在其中某步。

**修复（fix17）**：
1. **pinctrl 函数表整体重生**：python 从 pinctrl-lito.c 提取全部 146 脚
   funcs + tile + 函数组（下游命名→主线合并式：phase_flag0-31→phase_flag、
   qdss_gpio*→qdss、GP_PDM*→gp_pdm* 等），替换 enum/groups 数组/表；
   生成后 tile 复核零偏差、SDC/UFS 偏移吻合（0x5b8000/0x1b7000/0x5bc000）。
2. **modeset 链路 DBRG 打点**：dpu_encoder atomic_enable / dsi pre_enable /
   dsi power_on（phy enable 前后+host power_on 前后）/ dsi_host power_on
   （vregs→link clks→pinctrl→timing→sw reset→ctrl enable 六段）/ phy
   enable（regs→hw enable）/ panel prepare（vddio→avdd/avee→reset→init
   cmds→PPS→compression）——共 20+ 点，挂点将以一行日志精确现形。

**fix17 产物**：sha256 `e637cd11b2257014c93ac4aa5250301837629463153f46d7c038d442c1ea09b6`。

**实机验证（第十八轮，e637cd11）**：uart 修复生效（**ttyMSM0 全功能 console
注册、earlycon 退役**✓），但挂点纹丝不动——`no GPU device was found` 后
静默，**20+ 个 DBRG 打点一个都没触发**（dpu_encoder enable / dsi pre_enable /
phy / host power_on / panel prepare 全在更后面）→ 挂点在这些之前的
msm fbdev/modeset 内部（dpu_crtc enable 等未打点区）。

**用户决策：转 simpledrm 先点亮**（把 msm 链临时屏蔽）：

- **fix18（纯 DT）**：display-subsystem@ae00000 `status = "disabled"`——
  msm 不再抢 ABL cont-splash 的 aperture（此前"企鹅后黑屏"正是 mdss probe
  顶掉 simplefb 所致）；ABL 配置的 MDP/DSI 扫描管道继续跑，
  `CONFIG_FB_SIMPLE` 的 simplefb（framebuffer@a0000000）→ fbcon →
  企鹅 + 内核日志 + /init 在 tty0 的 shell。WLED/LCDB 保持使能（背光/偏压）。
- msm 链后续恢复时放开 status，DBRG 打点仍在（挂点定位续作）。

**fix18 产物**：sha256 `46e63829a44179de7ebdeab7d2405d57127c657528c25ac6e89b908c20f6cc08`。

**实机验证（第十九轮，46e63829）**：**引导全链贯通不再挂死**（deferred →
ttyMSM0 → Run /init → shell ✓），simplefb 注册成功、fbcon 接管
（135x150）——但**屏幕黑**。根因：mdss 屏蔽后显示资源无消费者，late-init
把 ABL 留下的管道电源全关了——日志实锤 `PM: genpd: Disabling unused power
domains`（CX 域被关）+ `regulator_init_complete`（panel vddio l1c 等会被关）。
之前企鹅能亮正因 mdss probe 时这些资源都有使用者。

**fix19**：CMDLINE 追加 **`pd_ignore_unused regulator_ignore_unused`**
（与既有 `clk_ignore_unused` 同族；两参数在 6.17 pmdomain/regulator core
确认存在）——禁止内核关掉"无主"电源域/电压轨，ABL cont-splash 管道
（CX 域 + LCDB 偏压 + vddio + WLED 背光）持续供电，simplefb 写帧缓冲
直接上屏。

**fix19 产物**：sha256 `2eac1b1f7c7cf7878721773de059dd2842ba2b0034c2eb1bae8c40c0ee5c67b6`。

**实机验证（第二十轮，2eac1b1f）**：keep-alive 参数生效（`Not disabling
unused power domains`）但**屏幕仍全黑**——simplefb 路线存疑（ABL 管道存活性/
fastboot-boot 模式的 cont-splash 行为未验证）。**用户决策：撤屏蔽回归 msm
主线修复，并开启高通 GPU 驱动**（"no GPU device was found" 后的 modeset
挂点疑与无 GPU 的 fbdev 路径有关）。

**fix20（GPU 链整体使能 + 盲区打点）**：

1. **gpucc-sm7250.c 移植**（自下游 gpucc-lito.c，结构照 gpucc-sm6350.c）：
   13 时钟（gmu_clk_src RCG 0x1120 含 19.2/200/500MHz 档、ahb 0x1078
   CRITICAL、cx_gmu 0x1098、gx_gmu 0x1064、hlos1_vote_gpu_smmu 0x5000
   VOTED 等）+ LUCID PLL1@0x100（≈500MHz）+ **CX/GX GDSC**（live_dts 实证
   0x3d9106c/0x3d9100c，hw_ctrl 0x1540 / clamp_io 0x1508，旗标照 sm8250）；
   Kconfig/Makefile 注册 SM_GPUCC_7250。
2. **gcc 补 GPU gpll0 父时钟**：gcc_gpu_gpll0（AON 0x52000 bit15）/
   gcc_gpu_gpll0_div（bit16，父 gpll0_out_even），ID 161/162。
3. **a6xx_catalog 补 0x06020200（adreno 620.2）条目**（picasso DT 实名）。
4. **DT 启用四节点**：gpucc（clocks 改 bi_tcxo+gpll0 两父）、adreno-smmu
   （power-domains=<gpucc CX> 防 probe 读挂）、gmu、gpu。
5. **盲区 DBRG 打点**：msm_drm_init（register 后/post_init 后）、
   kms_post_init（poll init/client_setup 前后）、msm_fbdev probe
   （entry/stolen fb/done）、dpu_crtc_enable entry、dpu_kms_hw_init complete
   ——上轮挂点（"no GPU"与 dsi 链 DBRG 之间）全覆盖。
6. **已知风险**：GMU/SQE/zap 固件（a621_gmu.bin/a650_sqe.fw/a620_zap.mbn）
   不在 boot ramdisk 且无法下载——adreno probe 将在固件请求处优雅失败
   （msm DRM 无 GPU 仍可工作，错误信息会指明）。

**fix20 产物**：sha256 `7a6b5957ca924a311cc63a7ed33f3044891407b46c1ccb97ea5053114f8703af`。

### 迭代 21：修复 uart_putc 的 TX 序列 bug（来自 uart_test 项目的根因发现）

**背景**：独立小项目 `uart_test/`（与 copydown 打包格式一致的裸机 UART echo 固件，
v1-v9 迭代见其 README）在实机上暴露了 shim `uart_putc` 的潜在 bug：**"每字符一条
M_CMD0、不等 CMD_DONE、用 SE_GENI_STATUS/TX_FIFO_STATUS 做同步"的序列，在
~11-13 次命令后把 GENI 引擎楔死**（TX 引脚悬空，终端收到 0xFF 洪流；raw log 实锤
98% 0xff + 周期性 7f 7f = 引脚噪声，非数据错位）。shim 只打 S/A/C/K 四个探针字符，
低于楔死阈值，所以 copydown 引导一直正常——但任何想从 shim 打长输出的场景必崩。

**对照主线**（用户建议方向）：`qcom_geni_serial.c` 的 `poll_put_char`/`console_write`
用完全不同的同步方式，且本机 earlycon 实证能连打几分钟。uart_test v9 逐条移植内核
`__qcom_geni_serial_console_write`（整串一条命令 + TX watermark +
M_IRQ_STATUS 的 WATERMARK/CMD_DONE 同步 + 超时 CANCEL）后连续输出稳定 ✅。

**修复**：`scripts/copydown-shim.S.in` 的 `uart_putc` 重写为
`qcom_geni_serial_poll_put_char` 逐条对齐（单字符 API 不变，S/A/C/K 调用点零改动）：

1. M 活动中（SE_GENI_STATUS bit0）先等 M_IRQ_STATUS(0x610) 的 CMD_DONE(bit0)；
2. 超时走 CANCEL（0x604 bit2 → 等 0x610 bit4 → 0x618 清 CANCEL|DONE）；
3. 清 0x618 → TRANS_LEN=1(0x270) → M_CMD0(0x600) → **立即写 FIFO(0x700)**；
4. **再等 CMD_DONE 才返回**（超时同样 CANCEL）；所有等待有界（0x100000 迭代）。

反汇编已核对（0x40/0x610/0x604/0x618/0x270/0x600/0x700 全部在位）。manifest
全部 True；boot.img 重建成功。

**注意**：本迭代只改 shim 的 TX 子程序，引导行为不变（探针仍为 S/A/C/K），
待实机回归验证。

**产物**：sha256 `105660e06e2bc2b47f97771e87224599da995aeb716fc2b88f9ce17a1d6c98d1`。

### 迭代 22：USB gadget console——启动早期交互输入（UART RX 物理缺失的替代通道）

**背景**：uart_test 项目实锤本机 PCB 调试口只引出 UART TX（gpio37/RX 无测试点，
pad 悬空实测；RX 引擎通路经 GENI 内部回环验证完好）。交互输入改走 USB：
手机 USB-C 本来就是全功能口（fastboot 在用）。

**设计**（用户指示：设备为 USB2.0，只用 HS；数据源以下游为准、主线对照）：
- `g_serial`（CDC-ACM）gadget console → 设备 `/dev/ttyGS0`，电脑 `/dev/ttyACM0`
- 仅启用 HS phy `phy@88e3000`（femto v2 驱动，走 `qcom,usb-snps-hs-7nm-phy`
  回退串匹配，无需改驱动）；QMP combo（USB3/DP，0x88e8000）保持禁用
- dwc3 child：`dr_mode=peripheral` + `maximum-speed=high-speed` + 只挂 usb2-phy
- HS phy 三路供电对照下游 lito-usb.dtsi（pm8150_l5/l2/l12 → l5a 0.875/l2a 3.1/
  l12a 1.8，femto v2 驱动 devm_regulator_bulk_get 强制要求三路）
- 6.17 注意：高通 dwc3 胶水层在 `dwc3-qcom-legacy.c`（匹配 `"qcom,dwc3"` 回退
  串）；ICC 缺失返回 NULL 跳过（无 ICC 也能 probe）

**改动清单**：
1. `gcc-sm7250.c`：GDSC 注册回加 `.num_gdscs = 2`（ufs_phy+usb30_prim——
   usb@a6f8800 的 power-domains=<gcc 1> 需要 genpd 提供者；TBU 三连仍不注册；
   迭代 17 已证明当年静默复位真凶是 gpll7 幻影而非 GDSC 写）
2. 配置：`USB_G_SERIAL=y`、`PHY_QCOM_USB_SNPS_FEMTO_V2=y`
   （注意 Makefile 符号带 USB_ 前缀；曾误写不存在的 PHY_QCOM_SNPS_FEMTO_V2）
3. `sm7250-xiaomi-picasso.dts` + `runtime-dtb-picasso.dts` 同步：hsphy okay+
   供电（runtime dts 需给 l2a/l12a 补 phandle 0x61/0x62）；usb@a6f8800 okay；
   dwc3 child 三属性
4. initramfs `/init`：等待 /dev/ttyGS0 出现（≤120s）后 spawn USB 交互 shell

**预期**：UART 日志照旧 → dwc3/femto probe → 电脑 /dev/ttyACM0 出现 → USB shell。
**风险**（按概率）：GDSC 回加回归（仅 2 个无旗标域，可控）；femto 缺下游
param-override 调参（眼图余量问题，HS console 无碍）；pmic-glink 无 ADSP 服务
（挂起等待，无害）。

**产物**：sha256 `146a0da1f3df3bc8de6196248e98cf195d9615368f2e018d42c23318399a9d04`。

**下游对照审计（用户要求，同 uart_test 的机械审计纪律）**：
- ✅ 6 个 USB 包装器时钟 enable/halt 偏移与 gcc-lito.c 逐字一致；
- ✅ GDSC 偏移：下游 lito-gdsc.dtsi 为绝对地址（0x10f004/0x177004/0x17d050），
  减 GCC 基址 0x100000 = 我们的相对偏移（0xf004/0x77004/0x7d050），精确吻合；
- ✅ LDO 电压：l5a=880mV 与下游 `qcom,vdd-voltage-level 880000` 精确一致，
  l2a 3.072V/l12a 1.8V 与设备值一致；
- ✅ 驱动表以头文件宏索引（include 同一头文件），DT phandle 解析自洽；
- ✗→✓ **抓到 3 个 reset 表错误（照抄 kona 的又一例，gpll7 同类）**：
  `GCC_QUSB2PHY_PRIM_BCR 0x26000→0x12000`（原值是下游 SDCC1_BCR 的地址！
  本次 femto phy probe 的复位脉冲正好会踩错——审计直接省掉一轮调试）、
  `GCC_USB3_PHY_PRIM_BCR 0x50020→0x50000`、
  `GCC_USB3_DP_PHY_PRIM_BCR 0x50000→0x50008`。

**修正后产物**：sha256 `94cb7511d85b23c977cb67032e53444c3cd2ff9aac3c9aa9a16faa56c98a705b`。

### 迭代 23：实机首验迭代 22——USB 链路通、死于显示老坑；mdss 暂禁让路

**实机结果**（94cb7511）：
- ✅ GDSC 回加无回归（`usb30_prim_gdsc returned 0`）；
- ✅ **femto HS phy probe 成功**（首轮 -517 等供电 → defer 周期 `probe of
  88e3000.phy returned 0`——修正后的 QUSB2PHY 复位 0x12000 与三路供电生效）；
- ⏸ dwc3 首轮 -517（等 phy），下一轮 defer 重试前机器死亡；
- ❌ 死点 = `fbdev probe done` 后 = **迭代 18 的老 modeset 死亡点**——fix20
  首次实机验证即暴露：GPU 因 `Unknown GPU revision: 6.2.0.2`（fix20 加的
  a6xx catalog 条目未匹配）+ 无 GMU 固件而未绑，显示链回到修复前状态。
  **与 USB 改动无关**。

**处理**：按 fix18 已验证剧本把 `display-subsystem` 暂禁（runtime dts + 树 dts
同步），让 USB console 链路先走完。显示链修复（catalog 匹配 0x06020200 +
GMU 固件入 initramfs）留待后续迭代。

**产物**：sha256 `10a2ff955898a3160e29fe4b0cc3bcb177868b9749d766e805aa4841b16b869e`。

**实机（10a2ff95）——USB 控制器全通**：dwc3 包装器 probe ✓、dwc3 核心
`dr_mode forced to gadget` ✓、`probe of a600000.usb returned 0` ✓、系统存活到
shell ✓（显示暂禁生效）。唯一卡点：**legacy `CONFIG_USB_ETH`（g_ether，
"Memorial Day 2008"）抢先绑上 UDC**，g_serial 只能 `couldn't find an available
UDC` → 设备枚举成 CDC 网卡而非串口 → 无 ttyACM。单 UDC 单 gadget，禁
USB_ETH/USB_ETH_RNDIS 保 g_serial。

**产物**：sha256 `216a64a9442afc2f38b8a98370e534984b8aba49d103bf728b1702b3d3352638`。

### 迭代 24：✅ USB gadget console 实机打通 + 设备树统一到内核树

**实机（216a64a9）**：`/dev/ttyACM0` 出现，**USB 交互 shell 成功**——启动早期
输入通道落地（UART TX 出日志 + USB CDC-ACM 进命令）。

**设备树统一（用户指示：整体覆盖，不逐项同步）**：
- `sm7250-xiaomi-picasso.dts` ← `scripts/runtime-dtb-picasso.dts` 全量覆盖
  （迭代 0-23 实机验证状态的最终形态，自包含/显式 phandle，dtc 原样保留）；
- 原 dtsi 组织结构备份于 `sm7250-xiaomi-picasso.dts.dtsi-based.bak`；
- `scripts/runtime-dtb-picasso.dts` **保留为对照参考，不再参与构建**；
- `build-boot.sh` 步骤 2 改为编译树内源（系统 dtc 直编——树内 dts 无 include，
  与内核 make dtbs 产物已验证逐字节一致；且绕开 kbuild 的 syncconfig，
  避免 lld 临时目录被清理后 dtb 目标被 linker 探测卡死的问题）；
- **无损验证**：切换后 DTB sha `084dca55…` 与 boot.img sha `216a64a9…`
  均与实机验证版完全一致。

### 迭代 25：GPU catalog chipid 修复 + GMU/zap 固件入 initramfs（数据源：YAAP ROM 解包）

**背景**：迭代 23 实机暴露 GPU 未绑（`Unknown GPU revision: 6.2.0.2` + 无 GMU
固件）→ 无 GPU 的 fbdev 路径 modeset 挂死 → mdss 暂禁。本轮修 GPU 链并恢复
mdss。数据源：`YAAP-16-HOMEMADE-picasso-20260110/`（完整 Android ROM）。

**根因 1（fix20 catalog 条目 chipid 算错）**：`find_chipid`（adreno_device.c）
对 compatible `qcom,adreno-620.2` 的解析是 core=6/major=2/**minor=0**/patch=2
→ **0x06020002**（打印 6.2.0.2），而 fix20 手写的条目是 0x06020200（6.2.2.0，
把 minor/patch 位算反了——与迭代 20 l2c 十六进制乌龙同类，**chipid 应从代码
逆推、不能手拼**）。

**根因 2（固件缺失）**：GMU/SQE/zap 均不在 initramfs。zap 缺失不是"跳过"：
`zap_shader_load_mdt` request 失败返回 -ENOENT → `a6xx_gpu_init` 整体失败
（仅"无 zap-shader 子节点"走 -ENODEV 的 SECVID_TRUST_CNTL fallback）。

**ROM 解包工具链**（全部免 root 提取，不安装）：
1. `vendor.new.dat.br` → brotli（`apt-get download brotli` + dpkg-deb -x）；
2. transfer.list 为 **v4 范围对格式**（`new <count>,<start>,<end>`，count 也在
   逗号列表首位，[start,end) 连续块）——自写 python 拼 `vendor.img`（772MB）；
3. vendor 是 **EROFS**（非 ext4）：`erofs-utils` 的 `fsck.erofs --path=/firmware
   --extract=…` 直接提取。

**下游实锤**（`Xiaomi_Kernel_OpenSource…/drivers/gpu/msm/adreno-gpulist.h` 的
`adreno_gpu_core_a620`）：`REV(6,2,0,ANY_ID)`、gmem=SZ_512K、GMU 2.0、
**a650 家族寄存器规格**（GEN3 标法正确）、`sqefw=a650_sqe.fw`、
**`gmufw=a650_gmu.bin`**（不是上游 a620 条目的 a621_gmu.bin——那是 sm6350
生态）、`zap_name=a620_zap`（PIL pas-id=13，与主线 `GPU_PAS_ID 13` 一致）。
ROM vendor 固件目录印证：有 a650_gmu.bin/a650_sqe.fw/a620_zap.mdt+b00-b02，
无 a621*。

**改动清单**（用户纠偏后定稿：固件名与 ROM 文件逐字一致）：
1. `a6xx_catalog.c` 加 `0x06020002` 条目：fw=`a650_sqe.fw`+`a650_gmu.bin`、
   **zapfw=`a620_zap.mdt`（ROM 原始文件名）**、a620_hwcg + a650_protect、
   GEN3、APRIV+CACHED_COHERENT、speedbins {0→0, 132→1, 115→2}；
   **删除 fix20 幽灵条目 0x06020200**（对照 linux-6.17.0.zip 原始包实锤：
   上游无此 chipid，无任何平台是 6.2.2.0——gpll7 同类手算产物）；
2. 6 个固件固化到 `scripts/initramfs/firmware/qcom/`（ROM 原名原内容，
   已 cmp 逐字节验证一致），`make-initramfs.sh` 打包到 `/lib/firmware/qcom/`
   与 `/lib/firmware/` 双路径（`adreno_request_fw` 先试 qcom/ 前缀再裸名）；
3. 树内 dts：display-subsystem 恢复 `status = "okay"`（GPU 链就绪后 modeset
   不再走无 GPU 的 fbdev 死路）。

**ROM vendor 固件全集与选用依据**（本机 GPU 相关共 14 文件 4 组）：
| ROM 文件 | 用途 |
|---|---|
| a650_gmu.bin | **用**（下游 gpulist gmufw_name 写死） |
| a650_sqe.fw | **用**（下游 sqefw_name 写死） |
| a620_zap.mdt+b00/b01/b02(+elf) | **用**（下游 zap_name="a620_zap"，PIL 链） |
| a630_sqe.fw、a650_zap.* | 不用（老平台/kona 残留，共用 vendor 包） |

**查证记录**（用户要求：不确定的硬件信息上网核实后再落码）：
- 上游 6.17 的 `0x06020100`（A621）条目为独立平台（有专属 "Fix gpucc
  register block for A621" patch 序列，lkml/patchew 可查），**不是 sm7250**；
  sm7250 的 a620（6.2.0.x）上游无条目，自加是唯一路径；
- `a621_gmu.bin` 不在 linux-firmware WHENCE（A621 平台 vendor 专供），
  a650_gmu.bin 在 WHENCE 中为不限 SoC 的通用文件——与 lito 下游/ROM 用
  a650_gmu.bin 三方自洽（此前"a621 是 sm6350 生态"的说法有误，已更正）；
- Adreno 620 @ SM7250：最高 750MHz（用户确认 + 下游 opp 表
  opp-750000000/TURBO 实锤）、GMEM 512KB（下游 gpulist SZ_512K + 上游
  A621 条目 SZ_512K 双源一致）。

**契约核对**（本轮全部过）：
- zap region：`pil-gpu-mem@8bc15400` 0x2000 够装——`qcom_mdt_get_size` 按
  **PT_LOAD 的 paddr 差值**算（mdt 唯一 PT_LOAD：paddr 0x5000、span 对齐后
  0x1000 ≤ 0x2000；注意不是文件体积也不是 vaddr 绝对值）；
- `.mdt` 后缀可行：mdt_loader 拼分段名 = `fwname 末 3 字符替换为 b%02d`
  → a620_zap.b00/b01/b02，与 ROM 文件名一致，initramfs 无需任何改名复制品；
- speedbin 安全网：fuse 匹配不到 speedbins → 默认 BIT(0)（非致命）；qfprom
  cell 位定义若与 kona 不同也只是读到任意值走默认；
- GMU DT 资源（fix20 已就绪）：gmu 4 段 reg（含 rscc/PDC 与下游一致）、
  5 时钟、cx/gx 域、gmu iommus=<smmu 5>；gpu 3 段 reg + opp 表 275-750MHz。

**已知风险（实机观察点）**：GMU bring-up 本身可能有新挂点（CX/GX GDSC 时序、
GMU fw 版本协商）；zap 的 TZ pas 认证若被拒 → GPU probe 失败（日志会指明）；
gpucc（fix20 移植，未实机验证过 GDSC 路径）。

**实机验证（第一轮，3c1dffd2）**：巨大进展但未亮屏——
✅ DPU probe 成功（`ae01000.display-controller returned 0`，历史首次）、
✅ DSI host + panel probe 成功（`ae94000.dsi(.0) returned 0`，此前永远 -517）、
✅ gpucc CX/GX GDSC probe 成功、adreno_smmu 成功、merge_3d 修复后无挂死，
系统存活到 shell。❌ GPU 反复 `returned -517 after 8 usecs`（连
deferred_probe_initcall 后仍 defer）→ mdss master 不 bind → 黑屏（同时 DPU
probe 顶掉 simpledrm 企鹅，画面全黑）。

**根因（fw_devlink 陷阱）**：`CONFIG_NVMEM_QCOM_QFPROM is not set`。gpu 节点的
`nvmem-cells = <qfprom gpu_speed_bin>` 使 fw_devlink 把 qfprom 登记为 gpu 的
supplier；qfprom 驱动缺编 → supplier 永不 ready → gpu 的 really_probe 在**调用
驱动之前**即被打回 -517（8 usecs、无任何驱动打印为特征）→ gpu 不 component_add
→ msm DRM master 永不 bind。gmu 无 probe 行是主线常态（a6xx_gmu_init 由
master bind 调用，gmu 无独立 driver），`sync_state pending due to gmu` 只是
表象。修复：`scripts/config --enable NVMEM_QCOM_QFPROM`。

**顺带发现（config 静默回归，本轮不动避免多变量）**：`CONFIG_QCOM_SMEM`、
`CONFIG_HW_RANDOM_QCOM` 又被丢（日志 793000.rng sync_state pending 印证；
文档 §6.3 曾记录 SMEM 编入——olddefconfig 静默丢弃的又一例）。

**实机验证（第二轮，07b25f39）**：qfprom 驱动编入，但 probe **-22**：
`nvmem: invalid bits on …/gpu-speed-bin@1ea`——cell 写 `reg=<0x1ea 0x01>` 而
`bits=<5,8>` 跨字节需要 2 字节，nvmem core 校验不过（kona 上游模板是单字节
cell，长度抄串了）。qfprom 失败 → supplier 仍不 ready → gpu 照旧 -517。
**佐证**：live 实测 cell = `reg=<0x1ea 0x02>`（bit5 宽 8；kona 是完全不同的
@19b<5,3> 布局，不可照抄）。arm-smmu 侧有新信息：adreno-smmu v2、
"preserved 0 boot mappings"。

**25-fix2**：树内 dts cell 改 `reg = <0x1ea 0x02>`（纯 DT）。

**第二轮审计（用户指示：再对照 live/下游 kernel 完整检查）**——上轮显示侧
矩阵之外的盲区补查：

| 项 | 对照 | 结果 |
|---|---|---|
| gpucc 13 时钟+PLL1+CX/GX GDSC | 下游 gpucc-lito.c 逐项 | ✓ 全一致（gmu RCG 0x1120、19.2/200(gpll0_div÷1.5)/500(pll1)、PLL1 LUCID@0x100 l=0x1A；probe 的 0x1098 WAKE/SLEEP 0xF 写 = 下游 probe 360-363 行实锤） |
| gmu 节点 gcc 时钟 id | dt-binding | ✓ axi=142(DDRSS)/memnoc=140 |
| gcc gpu 两时钟 | 下游 gcc-lito.c | ✓ memnoc_gfx 0x7100c VOTED / snoc_dvm_gfx 0x71018 逐字一致 |
| **adreno_smmu** | 上游 sm8250 模板 | **缺 clocks(ahb/bus/iface)+dma-coherent → 本轮补**（缺 AHB 时钟时 CX 域 off/on 循环后寄存器访问会挂——此前靠 ABL 残留态） |
| DSI PHY strength [55 03…] | kona 下游 dtsi | 同值 → 主线 7nm 驱动默认适用 ✓ |
| zap region/pas-id | live | 0x8bc15400+0x2000 / 13 逐字印证 ✓ |
| qfprom 其他 cell | 我们 DT 仅引用 speed_bin | 无同类 invalid bits 风险 ✓ |

**实机验证（第三刷，938341d5）**：**GPU 链全线贯通到 panel 初始化**——
qfprom probe 0、gpu bind（`bound 3d00000.gpu`）、`dpu hardware revision
0x60010000`、**`Initialized msm 1.13.0`**、fbdev 1080x2400 stolen fb、
DSI→PHY（enable done）→host power_on→panel prepare（vddio/avdd/avee/reset
全通）全部 DBRG 打点触发；用户观察"企鹅一闪→背光关→背光开→黑屏" =
simpledrm 企鹅被 msm 顶掉 + panel off/on 循环（modeset 在跑）。剩三个问题：

1. **`disp_cc_mdss_byte0_clk status stuck at 'off'` → link clk -16 →
   host power_on 失败 → panel DCS -22**（显示阻塞点；DCS 失败是下游症状）。
   dispcc byte0 链偏移与下游逐字一致（0x2028/0x2110/0x2128/0x202c），根因在
   **reparent 时机**：assigned-clock-parents 在 DSI probe 时 of_clk_set_defaults
   执行，reparent -22 失败后**无人重试**，byte0/pclk0_clk_src 悬在错误 parent
   → enable 后 mux 无输出 halt 恒 off。上游 v2.9 平台正是为此改为 power_on
   时显式 reparent（dsi_link_clk_set_rate_6g_v2_9 的注释："DSI PHY PLLs have
   to be enabled to allow reparenting to them, so cannot use
   assigned-clock-parents"）。
2. **`a650_sqe.fw` 加载 -2（-ENOENT）**（GPU 微码缺失，不阻塞 DPU 软件路径）。
   本地证据链已穷尽全部正常：boot.img 内嵌 initramfs 解包验证含固件（13 个
   条目路径拼写正确）、usr/initramfs_inc_data 与 cpio.gz 逐字节一致（无截断）、
   populate_rootfs 正常、fw_path 含 /lib/firmware、wait_for_initramfs 无 warn。
   **待实机 USB shell 验证**：`ls -la /lib/firmware/qcom/`。
3. GMU 固件加载打印未出现（SQE 失败在前即返回，顺序内）。

**25-fix4（byte0 显式 reparent）**：照上游 v2_9 方案——`dsi_clk_init_6g_v2()`
尾部可选 get byte_src/dsi_pll_byte/pixel_src/dsi_pll_pixel（devm_clk_get_optional，
老平台无这些名行为不变）；`dsi_link_clk_set_rate_6g()` 开头（power_on 阶段，
PHY 已 enable）可选 reparent byte/pixel src → DSI PLL；DT dsi 节点补 4 个
clock 引用（dispcc[9/10]=BYTE0/PCLK0_CLK_SRC、phy[0/1]=byte/pixel PLL）。

**实机（fix4，79671db9）**：显式 reparent 仍 -22 → 诊断版（6a1da5b8）dump 出
框架视角 parent 表：`byte0_clk_src parents = {bi_tcxo, gcc_disp_gpll0_clk_src,
(null)}`——**DSI PLL 位置被 gpll0 占据**。第二轮框架级 debug（c1ca6bed，
clk_core_fill_parent_index 打印）实锤：`fw=dsi0_phy_pll_out_byteclk idx=0 ->
gcc_disp_gpll0_clk_src err=0`——**fw_name 字符串正确却解析成 clocks[2]**。
而 dtb 三重验证（本地 sha=manifest=boot.img 内嵌、fdtget 原始数据）clock-names
与 clocks 均正确 5 项、of_parse_clkspec 源码 name 优先匹配——机理未明（疑似
6.17 该解析路径异常）。

**25-fix5（绕行尝试）**：parent_data 改 .index。**实机（9cd429f4）再次升级证据**：
`idx=3 -> gpll0(clocks[2])`、`idx=5 -> dsiclk(clocks[4])`、`idx=0 -> bi_tcxo`——
**运行时对 index≥1 系统性 -1 偏移**（fw_name 时代全部数据点同规律吻合）。

**25-fix6（根因落网）**：偏移的物理根源 = **dts 里 dispcc 的 sleep_clk 引用
写 `<0x29 0x00>`，而 0x29 是 #clock-cells=0 的 fixed-clock**（fdtget 查证：
phandle 0x29 节点 `#clock-cells = <0x00>`）。多余的 arg u32 在运行时被
of_phandle_iterator 当作下一个条目的 phandle（无效 phandle 0，占据条目位
且其 cells 按 0 推进），此后条目索引全部 +1：
`[0]bi [1]sleep [2]无效0 [3]gpll0 [4]dsi0byte [5]dsi0dsi`——idx=3→gpll0、
idx=5→dsiclk 与 DBRG 实测逐点吻合。**dtc 不校验 arg 数与 #clock-cells 匹配、
fdtget 打印原始 u32 也不暴露此错**——隐藏极深的 dts 手写错误。
修复：`<0x29 0x00>` → `<0x29>`（全局唯一一处）；dispcc .index 值
（data_0[1]=3、data_4[1]=4）在条目表恢复后即正确。**教训：手写 phandle
数组的 0-cell 引用绝不能带 arg；审计 dts 要核对每个 provider 的
#clock-cells 与引用 arg 数**（dtc/fdtget 都不查）。

**产物**：sha256 `0c645147a188291b572e9ae4f3973fec98071c73bc7007493ba92d313fec6d73`。

**显示链全量审计（用户指示：对照下游内核 + live 设备树逐组件核对）**：

下游启动组件（live 实测状态）→ 主线对齐矩阵：

| 下游组件 | live 形态 | 主线现状 | 对齐 |
|---|---|---|---|
| dispcc | lito-dispcc syscon，vdd_cx | dispcc-sm7250（45 时钟+MDSS_GDSC+CORE_BCR） | ✓ |
| MDSS/DPU | sde-kms @ae00000，8 时钟，GIC 83，apps_smmu 0xb40 | mdss+dpu（4 时钟等价，lut/rot 主线不用；iommus 0xb40） | ✓ |
| DPU 块偏移 | live qcom,sde-*-off 全表 | catalog 除 merge_3d 外逐块一致（ctl/lm/dspp/pp/sspp/intf/wb/cdm/perf/vbif） | ✓（merge_3d 修复后） |
| **merge_3d** | **0x84000/0x84100** | **原 0x3000/0x3100（幽灵，老 catalog 残留）→ 修正 0x83000/0x83100（=live-0x1000，sm8250 同代佐证）** | **✗→✓ 本轮修复** |
| DSI0 ctrl | v2.4，L9A+refgen，mdss intr 4 | dsi@ae94000（io 精确命中 sdm845_dsi_cfg 组0；vdda+refgen+iface/bus） | ✓ |
| DSI1 ctrl | 存在（面板 ctrl-num=0 不用） | 不需要 | ✓ |
| DSI PHY0 | v4.1，L5A，strength/lane DT 调参 | dsi-phy-7nm（v4.1=sm8250 同款主线驱动；调参走驱动内建默认+mode 推导） | ✓（机制差异可接受） |
| DSI PLL | 7nm v4.1 @0xae94900 | phy 节点 dsi_pll 段 0xae94900 | ✓ |
| G7A 面板 | cmdline 实锤 dsi_display0=g7a_37_02_0a_dsc_video；时序 hfp40/hs12/hbp40、vfp32/vs2/vbp30；DSC 1.1/540x20 slice/2 slice-pkt/8bpc/8bpp/block-pred | panel-novatek-nt36672c（G7A 驱动）mode 与 DSC 参数逐项一致 | ✓ |
| 面板 reset | PM8150L gpio3 | reset-gpios ✓ | ✓ |
| 面板供电 | vddio=L1C 1.896 + avdd/avee=LCDB（avdd 版 supply 无 vdd 项，L16A 不消费） | vddio/avdd/avee 三路 ✓ | ✓ |
| 背光 | WLED 3 strings+cabc（bl_ctrl_wled） | backlight@d800 enabled-strings<0,1,2>+cabc | ✓（缺 ovp/fs 微调，非阻塞） |
| TE gpio | tlmm10（video mode 非必需） | 无（vsync 走 DSI 中断） | ✓ |
| ESD gpio | tlmm55（保护功能） | 无 | 可接受 |
| sde_rscc | 独立组件（idle pc 功耗管理） | 上游 msm 无 display RSC 驱动 | 可接受（仅功耗） |
| rotator/DP/ext_disp | disabled/无消费者 | 无节点 | ✓ |
| interconnect | msm-bus 向量投票 | qnoc 禁用（迭代 19），mdss 无 ICC 依赖可跑 | 可接受（无带宽投票） |

**merge_3d 修复的意义**：G7A 为 DSC 双 slice 面板（1080 宽=2×540 slice），
DSI 单 port 输出必走 PP0+PP1 → merge_3d 合并路径；基址写错 0x3000（实际
0x83000）时 modeset 写未映射区域——**这可能就是迭代 18 首次 modeset 静默
挂死的直接元凶**（叠加在 GPU 缺失之上）。

**产物**：sha256 `3c1dffd2c611bd9fe5d21e3864484469d253e9321410b7345d036ead79916551`
（含 merge_3d 修复；中间产物 98b40309/5dcb63a7/d10ea3df 均作废）
（校验：Image 内 `a650_gmu.bin`/`a650_sqe.fw`/`a620_zap.mdt` 字符串在位、
initramfs 6 个固件与 ROM **cmp 逐字节一致**、DTB mdss/DPU 均 okay、manifest
全 True；catalog 与上游原始包 diff 仅余一个新增条目）。
**待实机验证**。中间产物 `98b40309…`（无固件版）、`5dcb63a7…`（.mbn 复制品版）
均作废。

注意（后续编辑）：
- 设备树现在只有一个源（树内 dts），改 DT 即改它；临时文件仅作对照；
- 树内 dts 里 dwc3 用 `dr-mode`（连字符，实机靠内核 fallback 强制 gadget
  工作）；如需显式生效可改 `dr_mode`（下划线，绑定正名）——两者实机行为一致；
- sm7250.dtsi 已不被 picasso.dts 引用（覆盖后为自包含），后续如回归
  dtsi 组织结构，从 .bak 起步重新整理。

### 迭代 26：Kernel image misaligned 修复（shim 拷贝目标 2MB 对齐）+ 固件之谜重大进展

**misaligned 根因**（setup.c:326 实锤：`_text % MIN_KIMG_ALIGN(2MB) != 0`）：
ABL 把 boot.img 的 kernel 载荷装载在 `base+0x8000`（0x80008000，kernel_offset
字段），shim 的拷贝目标沿用 `_start`（shim 自身基址）→ 内核 `_text=0x80008000`
非 2MB 对齐 → **内核触发 relocate 自搬移**并打印 FW_BUG 提示。且外层 shim 头
的 `image_size=0x10CF000`（整个 shim，含 runtime_dtb）远大于真实内核尺寸——
relocate 按它搬移整个区间，引入破坏 initramfs/dtb 的风险变量（见下）。

**修复**（shim.S.in 两处，反汇编已核对）：
1. `copy_entry`：拷贝目标 `_start` 经 `bic x0, x0, #0x1fffff` 向下 2MB 对齐
   （dest=0x80000000 < src=0x80208000，前向重叠拷贝安全；copy_entry 在 shim
   尾部位于目标区之外；dc cvau 清理范围随 x7 自动正确）；
2. `copy_done` 跳转：`adrp x4,_start` → `mov x4, x7`（= 对齐后 dest）。

**固件之谜（a650_sqe.fw -2）同日重大进展**：实机 shell 确认
`/lib/firmware/qcom/` **不存在**（"固件入 initramfs"从未在运行时生效——
6.17 与 7.3 的 -2 由此统一解释）。本地三连验证（源 cpio.gz / usr/
initramfs_inc_data / boot.img 内嵌 Image 解包）**全部含 13 个固件条目**、
shim 拷贝长度精确（linux_copy_size=0xec3008=Image 完整大小）——断点锁定
**内核解包前的内存内容**：**头号嫌疑即上述 relocate 搬移**（按伪造的
image_size 大范围搬移时 initramfs 尾部受害——固件恰在 cpio 尾部）。
本修复消除 relocate，若假设成立则固件问题随之解决——待实机验证。

**实机（49655072）：完全无法引导**——shim 三步打印全过后异常。异常 dump
的寄存器地址域（X30=0xA8F440B0、iframe 0xA6A197F0）暴露根本事实：
**ABL/UEFI 把 kernel 载荷装载在任意分配的高地址（本次 0xA8F08000 域），
并非手册 §2 推理的 base+0x8000=0x80008000**。bic 向下对齐把 dest 打到
0xA8000000——UEFI 活内存（代码/页表），毁掉 UEFI 后 jump 即炸（异常被
UEFI 自身的 handler 捕获打印）。

**回滚（当日）**：恢复 dest=_start / br _start。**结论修正**：
1. misaligned 提示的真正成因 = UEFI 任意装载地址，**内核 relocate 正是
   为此设计且一直正常工作**（6.17/7.3 均能跑到 shell 实证）——提示本身
   为无害 warn，**不再尝试消除**（任何 shim 侧"对齐"都在赌 UEFI 内存
   布局，教训记录在案）；
2. relocate 破坏 initramfs 的嫌疑**不成立**（relocate 可靠），固件丢失
   （运行时 /lib/firmware/qcom 不存在）的战场转回 **initramfs 解包环节**
   ——下一步：实机 `dmesg | grep -iE "initramfs|unpack|cpio"` 取解包
   错误证据。

**固件之谜取证轮（回滚版 e034e11b 实机）**：用户实机 `dmesg` 无任何解包
错误、`/lib/` 仅剩 musl 两文件——**解包"干净地"在 cpio ~90% 处截断**（丢的
恰为尾部固件段）。静态层全面排除：vmlinux `__initramfs_size`=1008428 精确、
`__initramfs_start` 处 gzip 头在位、boot.img 内嵌流完整（59 条目含 13 固件，
偏移与符号精确对上）、shim 拷贝长度精确、nullfs 打印为 7.3 正常 VFS 流程
（排除）。**新增 DBRG-INITRAMFS 打印**（unpack 终点统计）待下轮实机取内核
视角证据。另：6.17 从未实机 ls 过（-2 可能另有原因），不排除此为 7.3 特有
回归（unpack_initramfs/initramfs_options 为 7.x 重构）。

**取证轮 2（d6b56c41 实机）**：`my_inptr=1008428`（解压器完整消费）+
`state=Reset`（干净收尾）+ `message=NULL`（零错误）+ `/init`、`/bin/busybox`
md5 与源一致；实机完整清单与基准 diff：**丢失集 = 固件段 13 条（cpio 最尾
连续段），其余 58 条全部落地**（/bin 33 项与 spec 完全一致）。打包层以
严格 newc 解析器（修正字段偏移后）三重闭合：boot.img 按 `__initramfs_start/
size` 精确切片 == 源 cpio.gz 逐字节、TRAILER 正常终结、12 个固件文件与源
逐字节一致——**"没打包进去"方向排除（用户质疑已复核）**。
矛盾三角（TRAILER 在固件后被读到、但固件没落地、输入完整消费）指向
解包状态机在末段的行为。**新增 DBRG-CPIO 逐条目探针**（do_name 打印
name/mode/body）待下轮实机——直接看到内核"看到"的最后条目。

**✅ 案结（DBRG-CPIO 探针轮）**：内核**完整处理了全部条目**（12 固件 +
qcom dir + TRAILER，name/mode/body 全部正确）——但实机 /lib 仍只有 musl
两文件。对照 do_name 源码破案：`filp_open(O_CREAT)` 失败 **`return 0` 静默
丢弃**、`init_mkdir` **返回值完全忽略**——而 **spec 从 6.17 第一天起就缺
`dir /lib/firmware` 父目录条目**（cpio/gen_init_cpio 不自动创建父目录、
不校验层级）→ `mkdir(qcom)` ENOENT 静默 → 12 个固件打开全部 ENOENT 静默
跳过 → fw loader -ENOENT。**这就是贯穿 6.17/7.3 两个月 sqe -2 之谜的
完整因果链**。此前所有静态验证（cpio -t/切片/逐字节）只验"条目在 cpio
里"，从未验"父目录存在"——盲区所在。
**修复**：spec 补 `dir /lib/firmware`（+探针清理）。**教训：initramfs
spec 的目录层级必须显式声明每一级**；文件"在 cpio 里"≠"能落地"。

**✅ 固件修复实机验证（0651f626）**：`loaded qcom/a650_sqe.fw` + `loaded
qcom/a650_gmu.bin`（from new location）——**两个月 sqe -2 之谜正式终结**。
推进到下一关：`a650 SQE ucode is too old. Have 93 need 95`（主线为上游
安全修复设的门槛；ROM 的 Android R 固件为 v93，下游 4.19 运行多年）。
**处置**：门槛放宽 0x095→0x093（a6xx_gpu.c a650 分支，注释记录量产前
应恢复或换新版固件）——按"固件与 ROM 一致"原则不更换固件文件。

**GPU 链推进（39d81e69 实机）**：SQE 门槛过 → **GMU fw v2.1.8 加载启动** →
HFI 前 4 条（INIT/FW_VERSION/PERF_TABLE/BW_TABLE）全部成功（GMU 活、中断
通、协议对）→ 死于 HFI VOTE（`GX_BW_PERF_VOTE`，id 5）超时 → GMU 停滞 →
OOB GPU_SET 连锁超时 → hw_init -110。根因：上游非 legacy 路径的 VOTE 按
linux-firmware 新版 a650_gmu 调校，ROM 老 fw 2.1.8 不处理（且被打懵）。
**修复（bring-up）**：`a6xx_gmu_set_freq` 非 legacy 分支退化为内核直接
`dev_pm_opp_set_opp`（跳过 GMU DCVS 投票）；上游化时换新固件恢复。

**GPU OOB 排障链（固件替换轮起）**：用户提供的 linux-firmware 仓库
（/home/yucong/Documents/linux-firmware.git-refs_tags_20260916-qcom）。
关键事实：**linux-firmware 的 a650_gmu.bin 与 ROM 逐字节相同**（v2.1.8）
——kona 主线设备用同款 GMU fw，固件版本维度排除；SQE 已换 linux-firmware
新版（旧兼容 hack 全部回滚：SQE 门槛恢复 0x95、VOTE 路径恢复上游）。
HFI VOTE 超时消失后 **OOB GPU_SET 独立超时**；votes 探针（DBRG-VOTES）
实证 **cmd-db ARC votes 构建完全正常**（ret=0、5 档、gx_arc=0x1000404）——
votes 机制排除。推理收敛：**上游非 legacy 从不发 OOB_BOOT_SLUMBER**（boot
末尾序列：BOOT_SLUMBER_OPTION=0 + GX/MX_VOTE_IDX 写入 + OOB bit30/31，
下游 4.19 a6xx_gmu_gfx_rail_on 必做）——fw 2.1.8 或要求先完成它才处理
GPU_SET。实验：a650 家族在 fw_start 也执行 gfx_rail_on（复用已构建的 ARC
votes）。附带发现：GPU OPP 表 750MHz 档未进 freq 表（max=625M，疑
speedbin 过滤，非致命待查）。

**OOB 之谜根因落网（PDC 编程缺失）**：BOOT_SLUMBER OOB（补 new 位 30/31 后）
同样超时——但 HFI 正常（GMU 活着）。寄存器偏移主线 vs 下游逐一对证**全部一致**
（HOST2GMU_INTR_SET 0x1f994 等）；ARC votes 正常。最终对照下游
`adreno-gpulist.h` 破案：**kona a650 的 `pdc_in_aop=true`（AOP 代编 PDC），
lito a620 无此标志（内核必须编 PDC）**——主线按 `adreno_is_a650_family`
硬判 `pdc_in_aop=true` 跳过 PDC 序列/TCS 加载 → lito 的 PDC 空 → GMU/RSC
无法执行电源切换 → OOB（切 GX 域）全超时，HFI（不涉电源）正常。
**修复**：a620（chipid 6.2.0.x）排除出 pdc_in_aop 组；xo/mx/cx.lvl 的
TCS 地址照下游用 cmd_db_read_addr 动态获取（主线写死 kona 值）。

**GPU GMU 系统性修复轮（用户指示：一次修完，勿零敲碎打）**：
黑屏事件实为 ea923920 的 OPP 破坏（opp-peak/avg-kBps 需 interconnect path
配对，GPU 无 icc 引用 → OPP 解析 -22 → GPU bind 失败连带 DSI/panel 全挂）。
系统性排查后锁定 **GMU graceful death 真凶**：`a6xx_hfi_send_bw_table` 的
fallback 链中 a620（0x06020002）不匹配任何分支（`adreno_is_a650` 是精确
chipid 非 family）→ 落到 `a6xx_build_bw_table`（a630 时代的 cmd-db 地址）→
fw 2.1.8 按错误地址执行 DDR/CNOC 投票 → fault → graceful death（cm3_status
0x900 实锤）→ 之后 VOTE/OOB 全超时。**修复包**：① 回滚 opp-bandwidth；
② a620 bw 表分支用 a650 同款（"single off"表）；③ catalog 补 .bcms
（SH0/MC0/ACV）；④ 750MHz 档 supported-hw 扩 0x07（修 freq 表缺最高档）；
⑤ EN_0 实验回滚、探针清理（保留 DBRG-OOB 分流器与 DBRG-VOTES 观察）。

**GMU 打通后的 GPU 核心层（e8f11fec 实机）**：✅ GMU 全链贯通——VOTE 无超时、
OOB 无超时（bw 表修复击中真凶）、750MHz 档恢复（nr=6）、zap 认证通过（无
fallback 打印）。GPU hw init 走到 **ringbuffer 提交（9 条命令）后 CP 不消费**
（rptr=0/wptr=9）——死因升级到 SQE/GPU 核心层。**新假设：SQE 微码与 GMU fw
版本配套**——此前换的 linux-firmware 新版 SQE（v95+）与 ROM 的 GMU fw 2.1.8
跨代（SQE 启动与 GMU 的握手不兼容）。**修复**：SQE 换回 ROM 原配 v93
（从 YAAP ROM 重新解包恢复），SQE 门槛重新放宽至 0x093（注释更新配套关系）。

### 迭代 27：✅ GPU 全链打通——gfx3d 根时钟内核补刀 + zap 装载区对齐修复（2026-10-12）

> **完整技术文档**（含全部阶段的现象/排查/根因/修复与方法论）：
> [2026-10-12-gpu-bringup.md](2026-10-12-gpu-bringup.md)

**起点**（迭代 26 尾）：GMU 全链贯通（HFI/VOTE/OOB 无超时、750MHz 档在表），
但 hw_init 时 ringbuffer 不 drain——CP1-5 探针实证 GPU 寄存器全零、
`sptprac=0x80`（bit6 GDSC_POWER_OFF=0 轨已上、**bit7 GX_HM_CLK_OFF=1 时钟死**）。

**排查方法演进（每轮一次刷机、逐层证伪）**：

1. **gpll0 分支保活**（假设：RCG 低档源无人 enable）：gpucc probe
   clk_prepare_enable 两分支 → CP5 逐位不变 → **否定**（已回滚）。
2. **系统性对照**（用户指示）：主线 sm8250（kona/a650，同 fw 实机工作）vs
   sm6350/sm7225（lagoon/lito 表亲）vs 下游 lito 4.19——DT 节点、gpucc、
   a6xx_gmu 全路径对齐审计。抓到并落地 **vrm.soc 的 TCS1/TCS3 第 4 条命令**
   （下游 `adreno_is_a620` 独有，主线缺失；cmd-db 实测 addr=0x43400）。
   保留（下游正确行为）但冷启动路径不走 PDC 序列、非本关卡钥匙。
   IFPC/WFI_CONFIG/perf-counter 等下游差异经 kona 主线实证非阻塞，不引入。
   sm7225=sm6350=lagoon 同管芯（smmu@0x3d40000、无独立 rscc 段）非 lito
   直系；其 gpucc 注册 gfx3d 树是 lagoon fw 代差行为，lito/kona 下游都不注册。
3. **CP0 判别**（OOB 前后 sptprac）：`0xc0→0x80`——**fw 真的执行了上电**
   （GDSC 塌缩→抬起），卡点精确定位在"使能 GX 时钟"一步。
4. **GPU CC 全量 dump**（CP5G）：fw 足迹完整——PLL0 被 fw 配置
   （L=0x4e≈750 档）、**LOCKED**（mode bit31/PCAL_DONE bit27）、gfx3d RCG
   （0x101c）CFG=0x101 指到 PLL0、gfx3d 分支（0x1054）enable=1、gx_cxo(0x1060)
   与 ACD 时钟(0x1164/0x1168)已开、夹钳已放（DOMAIN_MISC 0x1508 bit0=0）——
   **唯独 RCG CMD_ROOT_OFF(bit31) 不清**。
5. **fence 假说**：CPU 访问被 GMU AHB fence 拒绝 → `fence_status=0x0` → **否定**。
6. **源判别**：RCG 切 XO（19.2MHz 必活）/gpll0 均不动 → **PLL0 无罪**，
   死点在 mux 之后的根使能路径。
7. **GMU fw log 解码**（32 字环形，4 字/条）：`(0x23, seq=1..58, 0xdeadba55)`
   ×58——**fw 自述某操作重试 58 次全败**，与"命令到位但根不起"吻合。
8. **逐项补完判别**（ST1-ST8）：gx_gmu 使能✓（分支机制正常）、分支重触发✗、
   cx_gfx3d(0x10a4/a8) **写不粘=寄存器不存在**（lagoon 独有）、UPDATE✗——
   最终 **ST8 组合拳点亮**：`CMD_ROOT_EN(bit1)` + gfx3d 分支 0→1 重触发 +
   `CMD=0x3`（ROOT_EN|UPDATE）→ `sp=0x0 rbbm=0x1`，CP 全活、ringbuffer drain。
   **根因一落定：lito fw 无法独自点亮 gfx3d 根时钟，内核须补刀组合拳。**

**zap 关卡（GPU 点亮后战线自然推进暴露）**：`PAS_MEM_SETUP(13, 0x8bc15400,
0x1000)` 被 TZ 拒（-22；PAS_INIT_IMAGE 通过=签名被认）。法证：下游
pil-tz-generic→`pil_alloc_region` 对 relocatable 镜像用 **dma_alloc_attrs
动态分配 4K 对齐内存**、从不用 DT 固定区；主线的 pil-gpu-mem@8bc15400
**0x400 未对齐**被本机 TZ 拒。修复：adreno_gpu.c zap lito 分支
`dmam_alloc_coherent(SZ_4K)`。**根因二落定。**

**终局（647dbc2b 实机）**：zap 无错 → panel 链 modeset →
`Console: switching to colour frame buffer device 135x150` →
**`fb0: msmdrmfb frame buffer device`**（adreno_load_gpu 成功）→ Run /init
→ shell，1.02-1.45s **零错误**。**GPU 显示渲染链全通。**

**清理版（6275731d）**：ST/CP/CP5G/CP5L/OOB/VOTES/VRM/panel DBRG 全部
移除；保留为正式代码的：
- a6xx_gpu.c hw_init OOB 后的 **gfx3d 根时钟 quirk**（lito 判定 +
  SPTPRAC bit7 幂等门 + 组合拳 + 失败告警）；
- a6xx_gmu.c 的 **vrm.soc TCS 命令**（a620 独有，下游对齐）；
- adreno_gpu.c 的 **zap 动态分配**（lito 分支）；
- dispcc 的 .index 引用（注释保留 DBRG-CLK 出处）；
- qcom-wdt 停狗打印（记录在案的 bring-up 特性，量产前恢复喂狗）。

**遗留观察**：`sync_state pending due to gmu`（化妆性）；GMU log 的重试条目
在成功路径仍存在（fw 正常行为）；zap 成功后 SECVID 走 secure 路径的长期
正确性（当前无 secure rendering 消费者，无影响）。

### 迭代 28：触摸 bring-up——NT36672C 下游驱动主线化移植（2026-10-13）

> **完整技术文档**（实机取证/主线化全表/DT 决策/两刷判读/工具坑）：
> [touch-bringup.md](touch-bringup.md)

**数据源三源对齐**（下游 4.19 驱动 + live_dts 节点 + **实机取证**）：

实机取证（YAAP Android 16，adb+root，对 bring-up 决策性的三条）：
1. **系统无任何 TP 固件文件**（/vendor /odm 全面搜过），触摸完全正常
   （`NVTCapacitiveTouchScreen` 活跃、fw_ver=0x14）——IC flash 常驻 fw
   自启动，`request_firmware` 失败不影响功能 → **initramfs 不带固件**
   （照抄实机行为，变量最少；下游 oss 树的 .bin.ihex 已转换备用，
   `tools/touch-fw/` 4 个 ×139264B）；
2. **触摸 irq = msmgpio 5 Edge**（计数活跃）——TLMM gpio→GIC 汇总中断
   路径实证可用（主线首次真实消费者），Edge 触发与驱动
   `INT_TRIGGER_TYPE=IRQ_TYPE_EDGE_RISING` 一致；
3. **lockdown 实测 `0x41,0x37,0x32,0x01,0x47,0x7b,0x31,0x00`**
   （/proc/tp_lockdown_info）→ 匹配 cfg_0（fw01）；
   `dsi_panel_lockdown_info_read` stub 直接填此值（面板匹配不再依赖 DSI 读）。

**架构**（与 live 一致）：ts_novatek 是 /soc 下独立 platform 节点
（"novatek,NVT-ts-spi"，late_initcall probe）；SPI 传输经 spi@984000 下
`touch_spi@0`（"xiaomi,spi-for-tp"）垫片设备转交（下游
`drivers/misc/spi-xiaomi-tp.c`，98 行，一并移植）。

**驱动移植**（`drivers/input/touchscreen/novatek-nt36672c/`，5 文件，
mp_ctrlram 工厂校准未随移植 NVT_TOUCH_MP=0）：

| 主线化改动 | 说明 |
|---|---|
| gpiod 化 | 7.3 无 of_get_named_gpio/legacy gpio API；DT 属性改主线惯例 `reset-gpios`/`irq-gpios`（`<&tlmm 4 0>`/`<&tlmm 5 0>`），驱动 devm_gpiod_get |
| proc_ops | file_operations → proc_ops（成员带点）；debugfs 的两个 fops 保持 file_operations 不变 |
| fb notifier 移除 | 7.3 fb.h 无 FB_EVENT_BLANK 事件——挂起通知整体未接（bring-up 触摸常开） |
| 4 个下游依赖 stub | dsi_panel_lockdown_info_read（填实测值）/dsi_panel_doubleclick_enable/touch_irq_boost/lpm_disable_for_input |
| probe defer 化 | ts_xsfer 未 init 时 -EPROBE_DEFER（本节点与 spi 设备无 fw_devlink 关联，不能假定顺序） |
| 其它 | strlcpy→strscpy、del_timer_sync→timer_delete_sync、timeval→ktime、remove 签名 void、rf resist 函数 TOUCHFEATURE 外 stub、`#define NVT_SAVE_TESTDATA_IN_FILE 1` 跳过未移植 testdata 接口 |
| CONFIG | +INPUT_TOUCHSCREEN / SPI_QCOM_GENI / SPI_XIAOMI_TP / TOUCHSCREEN_NT36672C |

**DT**（自包含 dts，phandle 新增 0x63-0x6a 无重复）：
- geniqup@9c0000（wrap1）disabled→okay（仅 SE7 启用，其余 SE 节点保持 disabled）；
- spi@984000：okay + pinctrl default/sleep（gpio6/7/8 qup11）+ touch_spi@0
  子节点 + **删 dmas 走 PIO**（FIFO 模式；免去 gpi_dma1+apps_smmu 链路变量，
  触摸报文小 PIO 足够；下游走 GPI 但量大）；
- tlmm 新增 8 组：qup-spi7-active/sleep + ts-int/reset/cs-active、
  ts-cs/int/reset-suspend（参数照 live 实测：CS=gpio9 qup11、
  int=drive16 pull-down、reset=drive16 pull-up 等）；
- ts_novatek 节点：全 cfg_0/1/2 子节点 + swrst-n8-addr/spi-rd-fast-addr
  + pinctrl 引用（active 3 组 / suspend 3 组）。

**白名单**：gpio4/5/9、6/7/8 本就在 72 脚白名单（fix15 时已按 live 消费者
预置），无需改动。

**产物**：sha256 `820b9ffd42710a6bbf9b60a8c2a2ed1c220b97ee8aa7a1b102426172cacd9d5a`
（Image 17,248,264B；构建前 beae4aad 清理版 v2 的 GPU 回归与本迭代合并验证）。

**实机验证（第一刷，820b9ffd）：触摸 probe 全链一次通过** ✅——
SPI 控制器/wrap1/垫片全 probe；**chip id 首读即中**
（`buf[1]=0x0C, buf[4..6]=0x72,0x66,0x03` → NT36675 map，PIO 模式完美）；
input1 注册；**irq 113 注册 + enable（TLMM gpio5 中断路径主线首次实证）**；
lockdown stub 打印实测值、面板匹配 cfg_0（fw01）与实机 Android 行为一致；
/proc/NVTSPI、nvt_fw_version 等 proc 节点全部创建。暴露两问题：

1. **`FW info is broken! fw_ver=0x50`**——fw request -2（无文件）后
   update_firmware_release() 无 IC 复位，IC 停在 bootloader 态；下游
   fallback 的 y_num=32（实际 36）证明 **Android 从未走过此路径**
   （其 fw info 从不 broken）→ IC 正常态依赖 fw 下载成功（RAM fw 模型）。
   **修复 = 固件入 initramfs**（此前"不带固件照抄实机"的判断错了——
   实机 Android 侧 fw 来源在内核之外不可见，我们自建 initramfs 必须自带）；
   4 个固件（.ihex 转换，tools/touch-fw/，cmp 验证）固化
   scripts/initramfs/firmware/touch/ → /lib/firmware/ 裸路径；
2. **`Handlers=kbd` 无 event**——CONFIG_INPUT_EVDEV 未开，
   /dev/input/event* 不存在 → **+CONFIG_INPUT_EVDEV=y**；
   顺带补 spi_device_id 消 "touch_xsfer has no spi_device_id" 警告。

**第二刷产物**：sha256
`a6292420815715ae5678492f845541081bf84ac29acacb2f8084d6eed41340aa`
（cpio 64 条目含 4 触摸固件，fw01 抽查逐字节一致）。判据追加：
15s 后 `nvt_match_fw → fw 下载成功 → nvt_get_fw_info 正常（fw_ver 合法
+ x_num=18/y_num=36）`；`/dev/input/event1` 出现；
`hexdump -C /dev/input/event1` 触摸出坐标流。

**首刷判据**（NVT_DEBUG=1 → NVT_LOG 全 pr_err 常显）：
```
probe 链:  spi@984000 probe → touch_xsfer → NVT-ts-spi nvt_ts_probe start
SPI 通:    buf[1]=0x0? ...（chip id 读回；全 0x00=总线死、全 0xFF=无设备）
识别:      This is NVT touch IC → request irq NN succeed
14s 后:    nvt_fwu_work（fw request 失败预期——flash fw 常驻，仅日志）
验证:      /proc/bus/input/devices 有 NVTCapacitiveTouchScreen
          USB shell: hexdump -C /dev/input/event* 触摸看数据
```

**风险预判**（按概率）：① SE 时钟 RCG wrap1 s1 偏移（迭代 20 已全量审计，
应当正确）；② geniqup wrap1 首次 probe（m/s-ahb 投票位迭代 20-fix2 修过）；
③ TLMM gpio5 中断路径首次实机使用（实机 Android 实证 msmgpio 可用）；
④ SPI 首传输挂（CS 时序/pinctrl 应用顺序——ts 驱动 probe 先于首传输
应用 pmx_ts_active，含 CS）。

### 迭代 29：UFS bring-up——sm7250 QMP-UFS-PHY 主线表移植（2026-10-13）

**侦查三源定性**（动手前全部结论先行）：

1. **下游 4.19**：UFS PHY 是独立老式框架驱动 `phy-qcom-ufs-qmp-v4-lito.c`
   （compatible `qcom,ufs-phy-qmp-v4-lito`，calibrate/power_control/
   start_serdes 回调制）。三张表：rate_A_no_g4（84 项）/2nd_lane_no_g4
   （45 项，内容=lane0 TX/RX 镜像+MULTI_LANE_CTRL1）/rate_B（1 项）。
   布局 COM=0x000/TX(n)=0x400+0x400n/RX(n)=0x600+0x400n/PCS=0xC00。
2. **主线 7.3**：UFS 主机栈在 `drivers/ufs/{core,host}/`（ufshcd-qcom 绑
   `qcom,ufshc`，reset controller/gear/set_mode/calibrate 链全兼容）；
   qmp-ufs 驱动有 sm8150/8250 表但**无 sm7250**。关键对拍发现：
   - lito 的 serdes 24 项与 sm8150_ufsphy_serdes **逐值相同**；rate-B 表
     （VCO_TUNE_MAP=0x06）= sm8150_ufsphy_hs_b_serdes 逐值相同 → 复用；
   - **TX/RX 尾部寄存器偏移 lito 专属**：RX 无 RX_PWM_*_TIMEOUT 段，
     RX_MODE_00 起全部 -0x14（DCC_CTRL1 0x1A8 vs 0x1BC）；TX 的
     TRAN_DRVR_EMP_EN（0xC0 vs 0xB8）与 PWM 分频器（0x168+ vs 0xD8+）
     移位——下游头文件即本机硅片权威（Android 实跑），不能套 sm8150 宏；
   - PCS 区 100% 吻合（仅缺 RX_SIGDET_CTRL1 宏，补定义 0x154）；
   - 主线 calibrate 流程（assert ufsphy reset→写表→deassert→清 SW_RESET
     →START→poll PCS_READY）与下游回调序列逐步对应；2nd lane 由
     `.lanes=2` 自动把 tx/rx 表写两遍（等效下游"2nd_lane 表"）。
3. **gcc/实机**：gcc-sm7250 驱动 UFS 时钟（60-70 号）、GCC_UFS_PHY_BCR
   （201）、UFS_PHY_GDSC（0）全部在位——迭代 20 全量偏移审计的红利，
   gcc 侧零改动；apps_smmu 实机已绑定翻译（display 同路径实证）；
   MIUI dtb 实证 UFS sid=0xe0，补 0x4e0（上游 sm8250/8350/8450 的 cmd
   描述符流惯例）。

**qnoc 不阻塞**：主线 ufshcd-qcom 的 ICC 路径在 DT 无 `interconnects` 时
优雅退化（of_icc_get→NULL，icc_set_bw(NULL)=0）——§9.2 的
"qnoc 缺 BCM 表"与本迭代解耦，rootfs 阶段再处理。

**修改清单**：

| 文件 | 改动 |
|---|---|
| `drivers/phy/qualcomm/phy-qcom-qmp-ufs.c` | +`sm7250_ufsphy_cfg`（lanes=2、v4 布局、max gear=3、复用 sm8150 serdes/hs_b）；+`sm7250_ufsphy_{tx,rx,pcs}` 表（下游 lito 值逐项转换，lito 专属偏移用本地 SM7250_* 宏注明）；+`sm7250_ufsphy_vreg_l`（90192/19000µA，live 实值）；+of_match `qcom,sm7250-qmp-ufs-phy`（字母序入位） |
| `drivers/phy/qualcomm/phy-qcom-qmp-pcs-ufs-v4.h` | +`QPHY_V4_PCS_UFS_RX_SIGDET_CTRL1 0x154` |
| `sm7250-xiaomi-picasso.dts` | ufshc/phy 双节点 okay；iommus 补 0x4e0；`limit-hs-gear=<3>`（下游仅 no_g4 表）；vcc/vccq2-max-microamp 显式 `<800000>`（下游 dtsi 实值） |
| `.config` | 零改动（UFSHCD/UFS_QCOM/QMP_UFS/SD 已全 =y） |

**dtc 显示陷阱（本迭代新坑）**：`vcc-max-microamp = "", "\f5"` 不是垃圾值——
0x000C3500=800000 的 u32 字节恰可被 dtc 反编译的字符串启发式显示成
字符串列表；dts 里 `<800000>` 与 `"" "\f5"` 编译产物逐字节相同。判读
dtb 反编译输出时，数值型属性显示成字符串≠数据错误。

**fix1 实判（954841b9）**：时钟链全通（`clk: core_clk 200MHz / unipro
150MHz`、make_hba_operational err=0、RS=1）——freq-table 缺失即不建
clk_list 的判断正确。新失败点：首条 dev cmd（NOP，走保留通道
`queue_reserved_command`，op=0x00/tag0/200ms 属正常形态）超时 →
EH 死锁 → async_synchronize_full 卡住 → 无 /init（"无法开机"的机制）。
**插桩判读（v3-v6）伪案全清**：① 0x400=UIC_COMMAND_COMPL（本树位表
非标准 UFSHCI），12+3 次 = link startup DME 正常完成；② enable=0x470
= UIC 阶段标准使能集；③ UIC（寄存器接口）全通、UTP（需 DMA）零完成
→ 指向 SMMU 流问题。
**sid 实锤（e22217f1 三证闭环）**：arm-smmu 复位前 SMR 残留转储 =
XBL/ABL 用过的流映射——`SMR[1]=0x800f0080`（mask 0x0f→0x80-0x8f，
XBL 从 UFS 引导）+ sm6350（lito 同族）上游 `<&apps_smmu 0x80 0x0>`；
而 MIUI dump 给 ufshc 的 0xe0 实为 USB 流（USB console 在 0xe0 实机
工作佐证）。**UFShc 真实 sid=0x80**。USB(0xe0)/MDSS(0xb40，SMR[3]
残留亦为 0xb40)经实机验证保留；sm6350 的 0x540/0x800 与本机不同——
sm6350 仅作结构参考，sid 须逐机实证。
**对照审计（用户指令"参考下游重新实现"）**：下游 dtsi/内核逐项——
freq-table 8 对逐对一致 ✔；gcc UFS 偏移全一致 ✔（补 aux_clk_src RCG
0x770a0 父 bi_tcxo，本树编号 111）；reset-gpios 实为错误映射（真复位
= 专用 ufs_reset pin，下游 pinctrl 153/本树 146@0xbc000）已删；
dev-ref-clk-freq=<0> 已补；quirks（下游仅 hw1.x/2.x）无缺口 ✔；
vdd-hba→GDSC 与 power-domains 等效 ✔；触摸 pinctrl 下游 OSS 树不含
picasso 专属组，以 live_dts.dts（用户提供的运行时权威）+ 实机功能
验证为准保留。

**首刷判据**：
```
probe 链:  phy@1d87000（qmp-ufs probe 无错）→ ufshcd-qcom 1d84000.ufshc
phy:      calibrate → PCS_READY（"phy initialization timed-out"=表/复位错）
link:     ufshcd 1d84000.ufshc: Host Controller registered（无 -ETIMEDOUT/挂死）
枚举:     scsi 0:0:0:0: Direct-Access ... → sd 0:0:0:0: [sda] ... 512-byte
          blocks（容量≈UFS 实际大小；/proc/partitions 出 sda1..N 分区表）
smmu:     无 "arm-smmu 15000000.iommu: Unexpected global fault"
          （若 context fault on sid 0x4e0 → 去掉该 iommus 条目即可）
风险位:   vcc/l7c 上电（rpmh）、ref_clk 19.2MHz 解析、UFS_PHY_GDSC 投票
```

**风险预判**：① sid 0x4e0 猜错（低概率，fault 则单 sid 回退）；② ICE reg
区多余段无害（无消费者）；③ `freq-table-hz`/opp 缺失仅影响 clk scaling。

**首刷实判（92ab3382）**：PHY 链全过（qmp-ufs probe 无错、PCS_READY 达成、
fw_devlink 双向环正常解开、scsi host0 创建），其余子系统零回归（触摸 fw01
下载 180ms 成功 + fw_ver=20——**迭代 28 遗留的"下载完成态"顺带确认**）。
失败链：`UNIPRO clk freq 0 MHz not supported` + `cfg core clk ctrl failed`
→ `uic cmd 0x16（LINK_STARTUP）completion timeout` ×3 → probe -110。
**根因**：`ufshcd_parse_clock_info` 在 DT 无 `freq-table-hz` 时直接 goto out
**一个时钟都不进 clk_list**（live dtb 本带此表，我组 DT 时漏了）——驱动
从未 devm_clk_get/enable 任何时钟（能走到 DME 全靠 XBL/ABL 残留态），
`set_core_clk_ctrl` 列表遍历找不到 core_clk_unipro → cycles_in_1us=0 →
40ns 周期表 default 报错；链接自然无法训练。
**fix1**：DT 补 `freq-table-hz` 8 对（live 实值：core 50-200M、unipro
37.5-150M、其余 0；ICE 两对删去）。经验：**ufshcd 的时钟"可选"是陷阱**——
clock-names 给了名字不代表驱动会去拿，freq-table-hz 才是建 clk_list 的
开关（此坑上游所有 qcom dtsi 从不暴露，因为它们全都带着这张表）。

### 迭代 30 实判：qnoc 启用即死机（1.78s）——sync_state 屏障清票断供 fabric（2026-10-14 深夜判读）

**实机（50effd35，凌晨 02:13 抓取）**：启动推进到 **1.779398s 整机死机重启**
——不是日志截断，是 UART 字符流中途损坏（`scsi 0:0:0:5` 行行首 `[` 处
字节损坏，时钟/电源轨瞬断的指纹），随后 TZ/XBL 侧看门狗硬复位。死点在
第 6 个 LUN attach 中途，**早于分区扫描——READ(10) 判据本轮根本没被测到**。
死前窗口一切正常：qnoc 启用零警告、6 LUN 枚举、触摸/GPU/显示零回归。

**根因（静态证据链闭环）**：`icc_sync_state` 的**全局屏障**语义。

1. provider 注册时（core.c:1084）每个节点 `init_avg=init_peak=INT_MAX`，
   全部 BCM 以 `BCM_TCS_CMD_VOTE_MASK` 全速票写入（存活——本机前 1.0s
   一直跑在全速票上）；
2. ufshc probe（1.66s）投 MODE_MAX、link startup POST_CHANGE 投表值——
   聚合取 `max(init, request)`，init 恒赢，无净变化（存活）；
3. **屏障**：11 个 provider 的 sync_state 全部被调用后（无消费者的 7 个
   走 late_initcall，ufshc bind 供给最后 4 个 supplier），`core.c:1235`
   把所有节点 init 票清零重聚合提交——ufshc 路径 BCM（CN0/SN9/MC0…）拿
   真实票，**其余 BCM 票值 0/0 → `tcs_cmd_gen` valid=false → 显式禁用**
   （keepalive=false 的 ACV/SH2/SH3/SH5/MM2/MM3/CE0/SN1/SN2/SN4/QUP0 全
   部裸奔，keepalive=true 的 MC0/SH0/MM0/MM1/SN0/CN0 也只剩票值 1 的地板）；
4. 正在运行的 MDSS 扫描（MM2/MM3）与 gemnoc/snoc 通路瞬间断供 → CPU 卡死
   在下一个总线访问（恰是 UART FIFO 写 → 字符流中途断流）→ 硬复位。

**为何上游 sm8250 不死**：上游设备每个 provider 都有大量真实消费者
（USB/显示/CPU/spi/i2c 全带 interconnects，聚合后有真票保底）；我们只启
ufshc 一个消费者。迭代 19 的"qnoc 缺 BCM 会冲 RPMh 复位"其实只描述了注册
层，**sync_state 屏障才是真正的行刑时刻**——启用 qnoc 的完整攻击面比
当时认知的大一层。

### 迭代 31：sync_state 移除（bring-up 保活）+ ICCD 探针（2026-10-14 晚）

**修复**：`sm7250.c` 删 `.sync_state = icc_sync_state`——init 的 INT_MAX
票永远保留（ICC 治理完全惰性化 = cmdline `clk/pd/regulator_ignore_unused`
同族的第四个 bring-up 保活）。安全论证：消费者票被 `max(init, request)`
恒钳为 INT_MAX，任何运行期投票都无法再拉低任何 BCM。

**TEMP-DEBUG 探针（结案后删）**：
- `core.c icc_sync_state`：屏障达成打印（本镜像应**永不出现**——sm7250
  provider 不再回调；若出现说明另有 provider 挂了 sync_state）；
- `bcm-voter.c qcom_icc_bcm_voter_commit`：逐 BCM 打印 AMC 提交
  `ICCD amc voter=<dev> n=<n> NAME:x/y[(dis)]`——注册期约数百行（全速票），
  若仍死机，**死前最后一行 = 凶器 BCM 实名**。

**迭代 31 实判（1f44d304，2026-10-14 晚实机）**：✅ **死机消失，机器存活到
`Run /init`（36.77s）——sync_state 屏障因果确认**。无 "ICCD sync_state
barrier" 行（修复生效指纹）；ICCD 90 行实证投票链健康：provider 注册票
（CN0:0x4444444 系列 = init INT_MAX 标度）、ufshc 的 MAX/POST_CHANGE 两轮
投票均被 init 钳制（与注册值相同，如预期）。**READ(10) 楔死原样复现**：
6 LUN 全枚举（sdf=28MB LUN5 首次完整现身），分区 READ 全部 30s 超时 →
EH → offline → 六盘 unable to read partition table；WLU runtime suspend
-16 次生现象同 fix20。**ICC 带宽假说正式证伪**（投票到位仍挂）。
探针自查：功能正确但冗余（config-noc 注册期 56 条重复行、+0.5s 启动）、
voter 名无信息量——结案即删（迭代 32）。

### 迭代 32：ICCD 清场 + dev-reset pinctrl 补齐 + 下游审计收口（2026-10-14 深夜）

**下游 4.19 + live_dts 对照审计（本迭代主任务）——排除清单更新**：

| 项 | 审计结论 |
|---|---|
| tx_lane_enable | **v4 PHY 是空函数**（下游源码注释："v4 PHY does not have TX_LANE_ENABLE register"）——审计表最后候补正式出局 |
| ctrl_rx_linecfg | 仅 unipro 1.41 生效（本机 2.x）——不适用 |
| update_sec_cfg | 下游就是 `return 0` 桩——无操作 |
| probe 期 device reset | 下游也不做（detect_device 只在恢复路径；needs_reinit 仅强制 g4 场景）——Android 证明设备从 XBL 残留态可正常工作 |
| LCC 禁用 / hw_clk_gating / cfg_timers / QUNIPRO_SEL | 主线 7.3 均有等价实现 |
| freq-table ref/tx_lane0 | live 75M-300M vs 我们 0-0：**有意偏差**（gcc_ufs_phy_tx_symbol_0 是无父分支，clk_set_rate → -EINVAL → `ufshcd_set_clk_freq` 的 break 中断 scaling；0-0=enable 但不 rate，上游 sm8250 走 OPP 也从不 rate 这两个） |
| dev-reset pinctrl | **真缺口**：live 有 assert/deassert 两态（ufs_reset 特殊组，output-low/high），下游 ufshcd core 在 EH/恢复路径消费；主线 7.3 的 device reset 只走 gpiod（`reset-gpios`），而 ufs_reset 特殊组不在 gpio 范围——**补齐**（见下） |

**修改**：
1. ICCD 探针移除（bcm-voter.c + core.c 还原；sm7250.c 的 sync_state 移除
   保留 = 正式修复）；
2. `ufs-qcom.c/.h`：device reset 增加 **pinctrl 两态回退**——gpiod 缺席时
   `devm_pinctrl_get` + 按名取 "dev-reset-assert"/"dev-reset-deassert"
   （两态不全则整体弃用，维持上游可选语义）；`device_reset_ctrl`/vops
   `device_reset`/suspend 断言路径三条全部接入；
3. DT：tlmm 区新增 `ufs-dev-reset-assert/deassert` 两态（phandle 0x6f/0x70，
   live 实值：bias-pull-down + drive 8 + output 电平）；ufshc 补
   pinctrl-names/0/1（无 "default" 态 → 注册期不自动应用）；
4. freq-table 偏差注释入 DT。

**过程坑（记录在案）**：曾误判 7.3 pinctrl-msm 缺 PIN_CONFIG_OUTPUT 需补
框架支持——编译报 undeclared 后核实：7.3 的 generic pinconf 已把
output-high/low 映射为 **PIN_CONFIG_LEVEL**（pinconf-generic.c:198），
pinctrl-msm 原生支持该路径（io_reg 写电平 + oe，对 oe_bit=-1 特殊组的
位运算与下游 4.19 同型）。框架零改动。

**迭代 32 实判（fd2aca15，2026-10-14 深夜实机）**：❌ **UFS 全灭回归**——
probe 到 `link startup failed 1`（2.127s，DME 连败 ~450ms）→ probe -EIO，
无 LUN、无块设备（boot 反而 2.57s 到 init）。**根因链（新接线路径首次
实机即暴露）**：hw4 → `ufs_qcom_advertise_quirks` 打上
`UFSHCD_QUIRK_REINIT_AFTER_MAX_GEAR_SWITCH` → `ufshcd_probe_hba` 在链路
训练**前**先 `ufshcd_device_reset()`（vops）——迭代 ≤31 无 gpiod 时该
vops 返回 -EOPNOTSUPP = 空操作，迭代 32 接通 pinctrl 后变成真 ufs_reset
脉冲（assert 10µs→deassert，两态应用零报错=接线物理生效）→ 设备被冷
复位、固件启动远超后续流程窗口 → DME 连败。下游 4.19 从不在 probe 期
复位设备（只在 EH 恢复路径）——Android 从未踩过这条路。

### 迭代 33：清 REINIT quirk（回到下游行为）+ 保留 dev-reset 接线（2026-10-14 深夜）

**修复**：`ufs_qcom_init` 在 advertise_quirks 后清除
`UFSHCD_QUIRK_REINIT_AFTER_MAX_GEAR_SWITCH`（照上游 `ufs_qcom_fw_managed_init`
的既有清除模式，注释记录完整因果）。probe 期不再脉冲；dev-reset 的
pinctrl 接线**保留**给 EH/恢复路径（其审计定位的真正用途）。本机
gear3==max，reinit 流程本无增益。副作用：probe 少一轮 hba_stop/enable
重初始化（≤31 里该块曾以"空操作 reset"运行过——行为等价性由 fw_managed
路径背书）。

**迭代 32 的意外收获**：回归本身证明 ufs_reset 两态**物理生效**（无
select 报错、设备实际被复位）——EH 路径的接线无需再验证。

**判读预期**：回到迭代 31 形态（6 LUN 枚举、READ 30s 超时风暴、36s 到
init），看点仍是 EH 后恢复路径是否被 device reset 救回。

### 迭代 34：启动序列对照下游全量 diff + 两处缺失补齐（2026-10-15）

以 [ufs-live-dts-attr-map.md](ufs-live-dts-attr-map.md) 梳理的下游启动
逻辑为准，对主线 7.3 的 host+PHY 启动序列逐环节 diff（源码级）：

| 环节 | 下游 4.19 | 主线 7.3 | 判定 |
|---|---|---|---|
| PHY 上电 | assert BCR→1ms→写表（复位保持中）→deassert→1ms→start_serdes→PCS_READY→QUNIPRO_SEL | host_reset(200µs 脉冲)→phy_init→set_mode→power_on→calibrate（**assert→表→deassert**→SW_RESET→SERDES_START→PCS_READY）→QUNIPRO_SEL | ✅ 等价（时间常量略异） |
| lane sync 时钟 | hce PRE、PHY 就绪后 enable（rx0/tx0/rx1/tx1 可选） | hce PRE 同位置 | ✅ |
| check_hibern8 / hw_clk_gating | link_startup PRE | hce POST（仍在 link startup 前） | ✅ 等效位置 |
| cfg_timers | SYS1CLK_1US（qunipro 路径） | 同 | ✅ |
| core clk ctrl | max_freq_mode(150)+**PA_VS_CORE_CLK_40NS_CYCLES=6**（hw≥4） | set_core_clk_ctrl：1US 域+清 DIV_EN+**set_clk_40ns_cycles(150→6)**（1930 行） | ✅ 完整等价 |
| host TX LCC 禁用 | PA_LOCAL_TX_LCC_ENABLE=0 | ufshcd_disable_host_tx_lcc | ✅ |
| **RX LineCfg 使能** | link_startup PRE：清 PCS+0x148 bit1 | **无任何 linecfg 处理** | ❌→✅ 补齐① |
| **SAVECONFIGTIME_MODE** | link_startup PRE：PA_VS_CONFIG_REG1(0x9000) \|= 0x6000（QUNIPRO_CLK_GATING cap=hw≥3 默认开，picasso 生效） | 仅有 device-quirk 变体（bit12），host rmw 缺失 | ❌→✅ 补齐② |
| tx_lane_enable / AH8 / linecfg-disable | POST | 等价 / 不适用（v4 空函数/unipro 2.x/cap off） | ✅ |
| pwr_change PRE/POST | cfg_timers、ref_clk(PWM→HS)、adapt(hw4)、equalizer quirk / bus vote | 全有（ICC=迭代 30） | ✅ |

**修改**：
1. `phy-qcom-qmp-ufs.c`：`qmp_phy_cfg` 新增 `.linecfg_disable_off`；
   `qmp_ufs_phy_calibrate` 尾部（PCS_READY 后、link startup 前）幂等 RMW
   清 bit1（=下游 ctrl_rx_linecfg(true)，时序约束"link startup 前使能"
   仍满足）；sm7250 cfg 配 `SM7250_PCS_LINECFG_DISABLE 0x148`；
2. `ufs-qcom.c`：link_startup PRE 在 disable_host_tx_lcc 后补
   `ufshcd_dme_rmw(PA_VS_CONFIG_REG1, 0x6000, 0x6000)`（hw≥3 门，照
   下游 cap 路径；SAVECONFIGTIME_MODE_MASK 本地定义）。

**判读预期**：这两处是启动序列的"最后缺口"，但均为厂商时序旋钮——
READ(10) 楔死**未必**因此消失（诚实预期：更可能仍复现）。若仍挂，
启动序列方向正式穷尽，战场转向运行态仪器化（首挂 READ 与扫描命令的
逐 dword 对比，见 ufs-bringup.md §5）。

### 迭代 35~39：READ(10) 攻坚（2026-10-15）

详录（逐迭代实判/技术细节/review 记录）见 **ufs-bringup.md §5/§7/§8**，
此处只记走向：

- **35/36（bisect→真修复）**：35 关 34 两补齐复现 33 形态 → 34 无罪、
  32 的 pinctrl 接线实锤；36 给 `ufs_qcom_device_reset_ctrl` 加
  `link_started` 门（链路首次训练成功前禁止脉冲——真凶是主线
  `ufshcd_init` 的无条件设备复位）。✅ 回归修复确认，回到 31 形态
  （6 LUN + 36.9s init），READ 楔死原样——**启动序列/DT 属性两方向
  正式穷尽**。
- **37（仪器化+判读纠错）**：先离线解码 36 的 EH dump——自纠三处
  （CDB 字节序：挂死命令=4KB 分区扫描读非 1MB；0x60=UTRLRSR 非
  doorbell；hexdump LE 约定三次校准），新事实三件（本机 UFS hw
  **v4.2**；TRLUT/DESC_RAM 等控制器内部表全零；CFG1 QUNIPRO_SEL=0）。
  镜像三件套（提交/完成轨迹+doorbell 时间线+LU 激活后受控 READ
  矩阵），两轮 review+下游对照修 7 处（含 timeout 单位、outstanding
  幽灵位两个真 bug）。实判：**块层路径/命令构造/缓冲地址三假设全灭**
  （矩阵在保留 tag、bank1 coherent、逐字节完美 UPIU 下同样楔死），
  且楔死期间 ~50 条并发命令照常完成、abort TM <1ms 秒回——**逐命令
  楔死，控制器与设备命令处理器都活着**；数据相位 ≥64B 通、4KB 挂。
- **38（ICE 时钟假设→证伪）**：对照 live_dts 发现我方自 29 起 8 时钟
  形态漏了 core_clk_ice/core_clk_ice_hw_ctl 两对（live=10 对）→
  用户指示按 live 实频实现：gcc-sm7250 照下游 gcc-lito 移植缺失的
  `gcc_ufs_phy_ice_core_clk_src` RCG（主线只有孤儿分支，无 RCG 则
  probe 期 rate=-EINVAL 击穿）+ DT 补两时钟 + freq-table 75M-300M。
  实判：**证伪**——RCG 工作正常（零 rate 错）、ICE 实际使能 300M，
  READ 原样楔死。**时钟轴穷尽**（PWM✗fix10/ICE✗38/降频从未发生✗36）。
  确定遗产：gcc RCG + 10 时钟 DT + freq-table 与 live 逐对一致（保留）；
  "tx_lane0 有意偏差"确认为迭代 32 的对齐错位化石（live tx 本就 0-0）。
- **39（到达性判别矩阵 v2，待实机）**：收敛为一次分叉实验——
  **B=越界 LBA 读**（EDTL/PRDT 与基线全同形，设备应秒回 CHECK
  CONDITION、零数据相位）、C=未配置 LUN6：B/C 完成 → 命令确实到达
  设备、描述符路径全好 → 楔死锁定在数据相位/NAND（设备侧战场）；
  B/C 也楔死 → 命令根本没离开主机（取指引擎拒绝，"TRLUT 全零"复核
  为真）→ 主机侧战场。D=写 bMaxNumOfRTT=8+回读（两树驱动都不写
  RTT，主动探测新轴）。R3 review：query_attr 自持锁实证、D 初版
  写 2=与默认同值无效实验改写 8。quirk/tune 轴同轮审结
  （PA_TACTIVATE/HIBERN8TIME 调优仅 UniPro<1.6 生效=两树空转；
  VS_DEBUGSAVECONFIGTIME 只有 hisi 处理）。

### 迭代 41：供电模式修复 + QPHY 投票 + 诊断纠错（2026-10-16）

对照用户提供的 `uart_early.log`（#77/P39）、下游 4.19 和 live DTS，
发现 VCC/L5A/L9A 初始 LPM 且缺少 DRMS 权限，负载请求静默失效。
本轮将三轨固定 HPM，补齐 QPHY 的 RPMh QLINK 投票及 PHY 绑定；同时
修正 DWORD 调试地址、撤销基于错误 MDI 单位的 P40 写入实验，验证回到
正常分区扫描。完整因果链、文件清单、编译结果及实机判据见
[P41 记录](2026-10-16-ufs-power-fix.md)。

状态：macOS 侧先完成三个 AArch64 对象与 DTB 检查；随后 Linux 环境
完整构建并实机验证。

**实机更新（构建 #79，`boot_7.3.img` SHA256
`9b29d0b28cb074c1629ec16e0654abd28ca0c42ec0284da490bf56d03a1d0960`）**：
P41 判据全部命中——L5A/L9A/VCC 均为 HPM，
qphy.lvl 投票生效；首批 4KB READ(10) 全部 `ocs=00/result=0/residual=0`，
6 LUN 完整分区扫描，`Run /init` 2.176971s。用户 shell 确认
`/proc/partitions` 列出全部分区。UFS 主阻塞解除。

**清理版（同日 02:24）**：P41/迭代 29 的 UFS 调试与测试代码全部移除，
保留 HPM/QPHY 等功能修复；完整构建与 manifest 通过，`Image` 中调试
字符串归零。清理版 `boot_7.3.img` SHA256：
`e33ffec766cad31767a35e456d41f67ca7637ed1ff57a5146db85a79b4b74dee`，
待实机回归。后续待做 HPM/QPHY 贡献分拆，以及触摸 ESD workqueue 缺
`WQ_UNBOUND` 的 warning 修复。

### 产物 sha256 速查

| 迭代 | sha256（boot.img） | 状态 |
|---|---|---|
| 0 direct | df98411adf91da6924ca2c840c7a6a10292df94f951624b336e2d9b4a39e7399 | 作废 |
| 1 首版 shim | d122ab8a7930eb66f798a1a903db6c38cbca03b81eefa1fbd8e6051746d0e0a4 | 作废 |
| 2 EL+探针 | b138f1f436e8d5fc3c11b331d5096360d1646a6af498b9309878a9fee8c5d810 | 作废 |
| 3 串口内核 | 723e8eb93ff5c4e6fc0e10a2f178557c0e7ca493ea9a3336bdb2e8c5eb5c8bb4 | 作废 |
| 4 memory 节点 | 0fea5e1801f44c7e7457feaca4f1cb9fe0426e0dcf354ee710807e3b8269e955 | 作废 |
| 5 initcall_debug | 3de8edcb5114e9b68a70fff75a485ebdbacfed18523b412d3b8648c4f784c43c | 作废 |
| 6 打点 | d6d938e03c7b2a7927adc56c7e98c0276b5bafa36139f78f75b470c6548ec10e / 0306c3d19e1f1d8e5c1287248142d0ee6249d6b791471dad5d8dc57a9496421f | 作废 |
| 7 固件区保留 | 48b61b0e427ba232e3d9f5afc19f9145f8b9483d4d05aa23c8e8d1ac956e5a4c | 作废 |
| 8 wdt 驱动停狗 | 1fd9ba4c37dec2836845da6126d96263dcd3e1257635323b4558a18beee72c44 | 作废 |
| 9 三修复(地址bug) | 239bd26b458a9d6a98bc56ee93a1e927c085df11b4708ddae8e38615a2505bc1 | 作废 |
| 10 地址修正 | 8bef9b6b1ef3e1851c7e659cd096d20834bb7b6f4c8078d0d16675e756f17772 | 已被 11 覆盖 |
| 11 GNU ld 回退 | 98255e930e33a0a93c24f8f637c3d2ddfb92f1e44a44ff5a4d4b9c67e620eac4 | 已被 12 覆盖 |
| 12 DT 大修+时钟链 | d11f14ebb6d87157ea88d6caacc018e64d761b98956f90c43027321c38c31fc0 | 已被 13 覆盖 |
| 13 initramfs+pinctrl+rpmh 收缩 | f7871f727fd5aa73cc3ab5cf370751fca66b1e967cddbf2e7a6d55e44232324b | 已被 14 覆盖 |
| 14 GDSC 旗标+PLL 类型 | 3c202e0a3e3e75bc7b66dddc6ae02f85cb4c164020ee6071c6f437c67fd08371 | 已被 15 覆盖 |
| 15 GDSC 下线（bisect） | f274e75134cc06912227f09239defd45602c20cd8ae3a2945b7096d9e5378f2b | 已被 16 覆盖 |
| 16 逐时钟调试打印 | 4c4aa56c07408a8296e5618b0c4f49b7b6dbeec5f3cce6662d35406aac1b2712 | 已被 17 覆盖 |
| 17 移除 gpll7 幻影 PLL | 8018901910b49ad3b5744f916e1a3191154b5482a146199479c01404bf4fcbd5 | 已被 18 覆盖 |
| 18 clk_ignore_unused | b2979c79a18802835f9e33412c71d21def70e80ce9b84f05f1e7f157f2875586 | 已被 19 覆盖 |
| 19 禁 ICC + 修 gpio-ranges | 7bed931c15d9e78b932ef8c6d3e9e06e15d5a8a1fde402b5e085282cadef0605 | 已被 20 覆盖 |
| 20 显示 bring-up（dispcc/DPU/LCDB/SPMI） | a052b4c7dbcc5c60a88dc944ecf27d04aaa9ff506b50321d7b578a816295e56f | 首刷：mdss -16（gcc disp axi 幽灵偏移） |
| 20-fix gcc 全量审计修正 59 偏移 | 58196cdf32b3a70d2f23c77a6bdcd98b63bb3c539caece9d6a576a2cf12315a6 | 二刷：DPU✅，regs-131+qup ahb 投票位错 |
| 20-fix2 投票位/hwcg 36 处+vddio 改 l1c | 8defc7fc810298ba64d576f580349e5af3ddcff07e69ebfac14d9298f4852ad2 | 三刷：面板点亮白屏（LCDB✅），l2c 乌龙卡 regs-1 |
| 20-fix3 l2c 十六进制修正 0x117730 | a5b8058e571c04a97d581cbcee98589bf27b5192fa4ccb0dcf9ea446823d959f | 四刷：0x117730 仍错（=1144624） |
| 20-fix4 l2c=0x1174C0（python 复核全部数值） | a8d711a11a7e69be4586cbb38ff65615feb8debf8a86638a92b18ad1d6b2a822 | 五刷：🐧企鹅出现（simpledrm），pdc 未编致 spmi 链挂 |
| 20-fix5 CONFIG_QCOM_PDC=y | 7440376f3640e1e5e0cc3eb9f9ac59f2f12f7d8f0a1cebaf6b41d71ccb139479 | 六刷：spmi 通，pinctrl 假 tiles 挂死+lcdb/wled reg 错 |
| 20-fix6 pinctrl 4-tile 重写+SPMI reg 单 cell+wled 属性 | 66778ed4d24ec127b307c7b71e71b940c3b79e3bcd98e16fe55bb7a0ec4db4fc | 七刷：🐧后重启；wled/lcdb reg 段语义错 |
| 20-fix7 reg 段修正+refgen 等 5 config+tsens 隔离+打点 | d24643e17c4ee7fd1b04a1165e497dd43427745363b824b9790676672bf47451 | 八刷：🐧后重启；打点实锤 pinctrl 挂死，wled✅ |
| 20-fix8 pinctrl 三阶段打点+LCDB DT 直读 | eb2f684486283055307f8a93c6c3b4a40abb718c8c5fc70b7953abfe0e32612a | 九刷：打点二分=msm_gpio_init 挂死 |
| 20-fix9 gpiochip 前清中断+wakeup-parent 移除 | 132da978cc3ede5e557d37f4b6fb314468b27620e64cef97014e481b8d3b7b59 | 十刷：清循环在 gpio0(tile3) 写挂=east 段推断错 |
| 20-fix10 TLMM 单段+tile 并入偏移（下游算式） | 3b2867c1f9d2999718cd42b94fc9feafcf6bf21548ec4088b7262bb8e428e2e7 | 十一刷：gpio0 打点实锤鬼区；onewire 佐证公式正确 |
| 20-fix11 全量回滚临时物+对齐下游惰性行为 | 43eb0a3f391cf8696ba4c1f73d0b912ba88a9b899a352d1dc4536981b6f5a409 | 被 fix12 覆盖 |
| 20-fix12 UFS_RESET/SDC2 偏移修正（工作流验证） | adfd486bf238b60bea7fc6106584bcb4f1a1c957879229a5e89c80412f11735c | 十二刷：pinctrl 首probe即挂（gpiolib 逐脚 get_direction） |
| 20-fix13 soc init_valid_mask 清 EAST 死区 0-5 | 6831bfccd04135f4f50c69b896f71c173cc21ca4da163d726b5f9176ce9f276e | 十三刷：无效（挂错 girq 层，应为 chip 层） |
| 20-fix14 chip->init_valid_mask（gpio 层掩码，三环链核对） | b69356dd64575142845f7df3d7c26165f8d7245923a3f4d205ccbf560209135b | 十五刷：掩码生效，gpio59 ctl 读挂死（WEST 槽） |
| 20-fix15 下游审计 73 脚白名单 + qup02 组补脚 | 6c15200227e6a93fdbdfbd42c629c89bedee8570fad77849257a440e7a0e2f4e | 十六刷：gpio0 在白名单内被读挂（数据属性引用≠可访问） |
| 20-fix16 no_get_direction（probe 零引脚访问）+ 白名单剔 gpio0 | adbfee4a2b0f46c1822b3cf62a5a816ab228c66caa7b12ab0364c899f1fd684e | 十七刷：pinctrl通✅DPU起来✅；uart funcs表错WARN+modeset挂 |
| 20-fix17 下游函数表整体重生 + DBRG 显示链打点 | e637cd11b2257014c93ac4aa5250301837629463153f46d7c038d442c1ea09b6 | 十八刷：uart✅ttyMSM0；modeset 挂点在 DBRG 区之前 |
| 20-fix18 mdss 屏蔽转 simplefb（纯 DT） | 46e63829a44179de7ebdeab7d2405d57127c657528c25ac6e89b908c20f6cc08 | 十九刷：引导通✅simplefb注册✅；late-init 关电源致黑屏 |
| 20-fix19 pd/regulator_ignore_unused（keep-alive） | 2eac1b1f7c7cf7878721773de059dd2842ba2b0034c2eb1bae8c40c0ee5c67b6 | 二十刷：参数生效仍黑屏；转回 msm 路线+开 GPU |
| 20-fix20 GPU 链使能（gpucc 移植/gcc 父时钟/620.2 chipid）+盲区打点 | 7a6b5957ca924a311cc63a7ed33f3044891407b46c1ccb97ea5053114f8703af | 待实机验证（已被迭代 21 覆盖） |
| 21 uart_putc TX 序列修复（poll_put_char 对齐；根因来自 uart_test 项目） | 105660e06e2bc2b47f97771e87224599da995aeb716fc2b88f9ce17a1d6c98d1 | 待实机回归 |
| 22 USB gadget console（输入通道迁移，USB2-only） | 94cb7511d85b23c977cb67032e53444c3cd2ff9aac3c9aa9a16faa56c98a705b | 部分验证：phy✅，死于显示 |
| 23 mdss 暂禁（fix20 显示链实机暴露 modeset 挂死；给 USB console 让路） | 10a2ff955898a3160e29fe4b0cc3bcb177868b9749d766e805aa4841b16b869e | ✅ 存活到 shell；USB 全通，g_ether 抢 UDC |
| 23-fix 禁 USB_ETH（g_ether 抢 UDC，g_serial 绑不上） | 216a64a9442afc2f38b8a98370e534984b8aba49d103bf728b1702b3d3352638 | ✅ ttyACM+shell 打通 |
| 24 DT 统一到内核树（runtime dts 整体覆盖树内源，保留作参考；构建切换+无损验证） | 216a64a9442afc2f38b8a98370e534984b8aba49d103bf728b1702b3d3352638 | ✅ 实机验证（同 23-fix） |
| 25 GPU catalog 0x06020002 修复 + GMU/zap 固件（ROM 逐字一致）+ merge_3d 幽灵修复 + 显示链全量审计 | 3c1dffd2c611bd9fe5d21e3864484469d253e9321410b7345d036ead79916551 | 首刷：DPU/DSI/panel✅，GPU 被 fw_devlink 卡死（qfprom 缺编） |
| 25-fix1 CONFIG_NVMEM_QCOM_QFPROM=y（fw_devlink：gpu 的 nvmem supplier 缺编 → probe 前永久 defer） | 07b25f399a2ed04c8c47b2f11b2f9e2d90eea8b7211185f778a6aa099d6ec7a1 | 二刷：qfprom -22（cell 长度错，invalid bits） |
| 25-fix2 qfprom gpu-speed-bin reg 0x1→0x2（live 实测 2 字节，bits<5,8> 跨字节） | f7e8047b9509bc90816e94176f977cc75c154e8d653598764986f1505524aa08 | 未单刷 |
| 25-fix3 第二轮审计：adreno_smmu 补 clocks+dma-coherent（CX 域 off/on 后 AHB 缺失会挂）；gpucc/PHY/zap 全部对齐印证 | 938341d5d5ba34b8bcd322667983550fb52ea027dd4d64d4ccf6c51c12b31621 | 三刷：GPU 链贯通到 panel init；byte0 stuck + sqe fw -2 |
| 25-fix4 DSI 显式 reparent（v2.9 方案下沉 v2.4：可选 clk + power_on 时 byte/pixel src→PLL） | 79671db9ab4828c1e4b36ed797b0efb3373c1ce5f9b18726751508d8eac78e97 | 仍 -22（parent 表错位） |
| 25-dbg dsi_host dump + clk.c fill_parent_index dump | 6a1da5b8/c1ca6bed… | 实锤 fw_name 解析错位（→gpll0），dtb 无误 |
| 25-fix5 dispcc byte/pixel parent 改 .index | 9cd429f480a0660babf6aef0fe244a943f17ac51a6930ef92872ecd417b02377 | 实测运行时 index 系统性 -1 偏移 → 锁定 dts |
| 25-fix6 根因：dispcc 的 sleep_clk(#clock-cells=0) 引用误带 arg → 无效 phandle 条目挤位、后续索引+1；修复 `<0x29 0x00>`→`<0x29>` | 0c645147a188291b572e9ae4f3973fec98071c73bc7007493ba92d313fec6d73 | ✅ **屏幕点亮（2026-10-11）** |
| 27 GPU 全链（vrm.soc + gfx3d ROOT_EN 组合拳 + zap 4K 对齐动态分配，含全部 DBRG 探针） | 647dbc2b3513dd86567f6da870d7d25f6f638edc6b85d870f0c6df789fe95782 | ✅ **GPU 打通：fb0 msmdrmfb + fbcon 切换 + 零错误（2026-10-12）** |
| 27 清理版 v1（探针移除，quirk 漏了 CFG→XO 切换——组合拳在死源 PLL0 上无效） | 6275731dfc94de6ca37f8567f36c953aaec5e683dc52dd545b1312d0dca70621 | ✗ `gfx3d root clock did not turn on`（判决：PLL0 lock 位撒谎、输出实死——修正此前"PLL0 无罪"误判，ST6 判别缺组合拳） |
| 27 清理版 v2（成功序列精确复刻：gx_gmu + CFG→XO + UPDATE + ROOT_EN 轮询 + 分支重触发 + 组合 UPDATE；GPU 以 19.2MHz 运行） | beae4aadbfb81e765981eeb9b983651b445a136a81eb9d8c804cde8189ba4153 | 待实机回归 |
| 28 触摸 NT36672C 主线化移植（gpiod/proc_ops/stub 化 + SE7 SPI PIO + geniqup wrap1 启用；实机取证：lockdown 实测/gpio5 Edge irq） | 820b9ffd42710a6bbf9b60a8c2a2ed1c220b97ee8aa7a1b102426172cacd9d5a | ✅ probe 全链通（chip id/irq113/input1）；fw -2 + 无 evdev |
| 28-fix1 触摸固件入 initramfs（RAM fw 模型正道）+ INPUT_EVDEV + spi_device_id | a6292420815715ae5678492f845541081bf84ac29acacb2f8084d6eed41340aa | ✅ /dev/input/event1 触摸数据（2026-10-13） |
| 28-fix2 架构规范化：垫片删除，ts 直接挂 SPI 总线（spi_driver + touchscreen@0，详见 touch-bringup.md §8a） | c7a24f32…（产物已覆盖，仅存短前缀） | ✅ 回归通过；probe 提前到 0.96s，触摸变 input0/event0 |
| 28-fix3 spi_device_id 配对 + 垫片残留彻底清理（文件/Kconfig/Makefile 五处，详见 touch-bringup.md §10 教训） | 2ea56496…（产物已覆盖，仅存短前缀） | 功能与 fix2 等价；dmesg 应无 spi_device_id 警告（可选验证） |
| 29 UFS bring-up：sm7250 QMP-UFS-PHY 表移植入主线 qmp-ufs 驱动 + ufshc/phy 节点启用 | 92ab3382322125f1fbb593b942689fc1afcf3561438ea5a5f2bab6458a4fc6e4 | 首刷：PHY✅（PCS_READY），link startup -110（见 fix1） |
| 29-fix1 DT 补 freq-table-hz（下游 dtsi 实值 8 对，去 ICE）| 954841b97de6cbb5e80730be3460881e69dee9abe54994c1aee0a7f044f283b9 | 二刷：时钟链✅（200M/150M），首条 dev cmd（NOP）超时→EH 死锁（无 shell） |
| 29-fix2 插桩 v3-v6（scan=manual + UFSDBG 四点 + SMMUDBG SMR 残留转储 + DEBUG_FS/DEVMEM） | 83594ae9 / 334ba3e6 / 450af441 / e22217f1 | 判读见下：伪案全清，sid 实锤 |
| 29-fix3 对照审计 + sid=0x80（XBL SMR 残留 0x800f0080 + sm6350 上游双证；MIUI dump 的 0xe0 实为 USB）+ 删 reset-gpios（gpio46=qup13/qdss 引脚，真复位是专用 ufs_reset pin 153）+ 补 dev-ref-clk-freq | 29cd923b5c479c2c29aed4daec64fffb5b7c05e93a5379862e81cb9e4e05059b | 待实机验证 |
| 29-fix4 数据源重导出收口：gcc 补 aux_clk_src RCG（0x770a0，父 bi_tcxo 19.2M，下游对照）+ 上述全部 | 775c603e77244d2df2590abf19d92feaf436801756771c92af19ffdbc4242012 | sid 0x80 无变化（见 fix5 判读） |
| 29-fix5 终极插桩（doorbell 写后回读 + UTRLBA/AGG/CE 现场） | ac51b4a89920cee538b3e2e1ef92cf6aed650aa913f52dac617e46d90c544ea2 | 判读支撑 fix6 |
| 29-fix6 **删除 ufshc iommus**（下游 dtsi + live_dts 双权威均无此属性；三 sid 零 fault 零变化=UFS 抓取不经 apps_smmu，IOVA 无人翻译；去 iommus 走物理地址与 Android 同构） | 29688118b3ecd318a0f424db90ae0b70dc6af87d4deade496fe534407854f3cd | doorbell 锁存(DB=1)仍零完成 → 引出 fix7 |
| 29-fix7 **DMA mask 强制 32 位**（`ufs_qcom_set_dma_mask` vops；CAP 64AS 谎报，utrdl 落 4GB+（0x100fa1000）即死）+ `arm-smmu.disable_bypass=0` | 78be31bb565ea13a9c95148c8d07e16229e748ed29b4db37edd3215e90284b8a | ✅ **dev cmd/设备识别/shell 全通**（irq 0x1 首现、SKhynix 识别、Run /init 2.19s） |
| 29-fix8 撤 scan=manual（恢复自动扫盘） | ac51b4a8…（fix7+pmc 插桩）/ c5ba94766e1724ac40277411867de31094cd59141bf2d2b243680bc201cf547d | ✅ **sda 62.8GB+5 LUN 枚举**；但 READ(10) 全 30s 超时→offline→boot 36s |
| 29-fix9 关时钟门控（CLK_GATING/HIBERN8_WITH_CLK_GATING caps 删；空闲后 HW_H8 失步 2进1出→ocs=0xf） | c5ba9476（同上合并） | READ 挂死仍在（另见下） |
| 29-fix10 PWM 强制实验（dme_change 钳 SLOWAUTO） | 06c97558561693c899d995c0ff0a242c32dcd205ab59a79bcda1ae87adf1bd02 | PWM 同挂 → **HS/PHY 无罪** |
| 29-fix11 can_queue=1 并发实验（第一版只钳 11329 行，被 nutrs 路径 9433 行覆盖，实验无效——outstanding 仍 0x80a0000e 六位在途） | 41e949690f121895d6c50139e0da0653fccf2311d3961e7c76cc1c8adf085429 | **实判：并发假说证伪**——本次实机 outstanding=0x2（单命令在途）仍挂；且 EH 后 tag0 abort 风暴卡死 async_synchronize_full → 无 shell。TRD/UPIU/PRDT 逐字段全正确、OCS=0xF 未动、事后 link startup 连败 = 控制器/设备深度楔死。另破插桩截断假象：qc/rq 共用计数器恰 40、irq 上限恰 60，"打印消失"≠事件消失 |
| 29-fix12 can_queue=1 双点钳制 | 未单刷（fix11 已证伪并发，钳制保留仅为时间线干净） | — |
| 29-fix13 READ 挂死三假说单刷判别（首条 READ 三阶段救援：30ms 观察→纯重踢→PRDT 换探针缓冲重踢；probe 延迟 5s；SND 全命令时间线含 PRDT 落点；插桩计数器拆分+上限 400） | a9718cf63f5753ff5c6d74fe3c9122b1627d4aa98d9c2b8a85b9d831d5ffd9fc | **实判**：延迟 5s 仍挂→WA 排除；"重踢/换 PRDT 无效"判读修正为不构成 H2/H4 证伪（doorbell 重写对已锁存位是 no-op）；SND 时间线证明扫描 PRDT 与 READ PRDT 同窗紧邻；apps_smmu 全复位+SYS1CLK=200+hw4 实证→三假说排除 |
| 29-fix14 受控自测 T0-T3（关 IE、独占 tag、纯轮询手工构造 UTRD/UPIU/PRDT；移除 5s 延迟） | 9badc1bc20561de26e637e23da7e0a86ecff426bce019913e5f9bdb2af7239cb | **实判**：四 case 全 1ms DONE——命令通路+设备响应无罪；但自测 PRDT 偏移单位 bug（qcom 无 BYTE_GRAN quirk，字段=dword 单位）→数据相位未真正测到 |
| 29-fix15 自测修正（偏移×4 字节）+ 新增 T4（tag1 复刻块层命令） | 80939795… | **实判**：READ 在新/tag1 tag 均真实完成（响应 UPIU 有效）但全部 case data0=0；"挂点移到 INQUIRY"实为自测污染（钩子在 queuecommand 中段，T4 覆盖已备好的首条命令）；shell 慢启动成功（EH 未死锁） |
| 29-fix16 自测移 probe_hba 尾部（消污染）+ 响应 UPIU 16B 打印 + T5 | f25df2c2… | **实判**：干净自测全完成但 data0 仍全零；块层 READ 原样楔死（SND #164）；发现 I 位混淆（块层 0x15/自测 0x14）与 GFP_DMA32 不落 bank2 |
| 29-fix17 去混淆：自测补 I 位 + probe2 改 DT 预留 bank2（0xf0000000）+ 块层 ≥4KB PRDT 重定向实验 | 21c095ac… | **实判**：ioremap(Device) 上 memset 对齐错误 Oops 击落自测（黑屏）；崩溃前 T0（bank1+I=1+512B）仍零数据→I 位排除 |
| 29-fix18 probe2 改 memremap(WC) | 4032e7ca… | 未单刷（被 fix19 审计吸收） |
| 29-fix19 全面审计（用户指示）：回滚 fix10 PWM 强制遗留（8 轮判读环境失真）+ redirect 三重守卫 + 双探针 DT 确定化 + 删全部失效插桩 | 55ab51df… | **实判**：T0b（bank2 探针）无数据 + 重定向 bank1 的块层 READ 仍楔死——两条地址理论全灭；自测基线污染确认（跑在 add_lus 前，LU 未激活）→自测结论全部作废 |
| 29-fix20 **干净基线**（用户指示彻底清场）：ufshcd.c/scsi_error.c/arm-smmu.c 还原初始提交（99 行纯插桩零真修复）；仅留 ufs-qcom.c 两实证修复；DT 探针区删；权威源逐属性对齐确认 | 620077f3bc22ddd1…（完整 sha256 见 ufs-bringup.md §7） | 待实机 |
| 30 qnoc 裁表启用 interconnects（干净基线判读后既定方向）：删 6 幽灵 BCM（SN11/12/14/15+SH8/10，本机 cmd-db 无；aggre2/compute/system 三处数组+定义）；DT 11 个 interconnect 节点 disabled→okay（4 个被引用者补 phandle 0x6b-0x6e）；ufshc 补 interconnects 两条路径（a1noc MASTER_UFS_MEM→mc_virt EBI、gemnoc AMPSS_M0→cnoc UFS_MEM_CFG，上游 sm8250 同款） | 50effd3522183731e5cc1809b3a77d8209ba80d23e423e08b7f448b501fb0855 | **实判：1.779398s 整机死机重启**（UART 字符流中途损坏=fabric 断供指纹；早于分区扫描，READ 判据未测到）。根因=icc_sync_state 全局屏障清 init(INT_MAX) 票、非 ufshc 路径 BCM 显式禁用（详见迭代 30 实判节） |
| 31 sync_state 移除（sm7250.c 无 .sync_state，init 票恒 INT_MAX=第四个 ignore_unused）+ ICCD 探针（core.c 屏障见证 + bcm-voter 逐 BCM AMC 提交打印） | 1f44d3041ce3ac410dd050ffb9c258e889de06c3020d3d4b14d045cb75a21eef | ✅ **实判：死机消失、存活到 init（36.77s）——屏障因果确认**；READ 楔死原样（ICC 带宽假说证伪）；6 LUN 全枚举 |
| 32 ICCD 清场 + dev-reset pinctrl 补齐（ufs-qcom pinctrl 两态回退 + DT ufs_reset 组 0x6f/0x70）+ 下游审计收口（tx_lane_enable=v4 空函数出局；freq-table 有意偏差注释） | fd2aca1540212382575c8c6cb512db2db19518244091d409a19bdcf7cedb4e5f | ❌ **实判：UFS 全灭回归**——hw4 的 REINIT quirk 使 probe 期真脉冲 ufs_reset（≤31 是 -EOPNOTSUPP 空操作）→ 设备冷复位后 link startup 连败 → probe -EIO。副作用收获：两态物理生效实证 |
| 33 清 REINIT_AFTER_MAX_GEAR_SWITCH（照 fw_managed_init 既有模式；probe 不再脉冲，dev-reset 留给 EH 恢复路径） | 968fc6c6bc35ba6e2a9cf58e5ce31629a6401fdf5fb23cca5216092b4cf2be3f | 未单刷（被 34 累积包含） |
| 34 启动序列对齐：qmp-ufs 补 RX LineCfg 使能（PCS 0x148 bit1，cfg 门控）+ ufs-qcom link_startup PRE 补 PA_VS_CONFIG_REG1 SAVECONFIGTIME_MODE rmw（0x6000，hw≥3） | 260380a69f7b0a2b29dfb1a811413b89d223fd5e68d5add8b55a542a90895735 | 待实机（含 33；判据：回 31 形态后看 READ 是否仍挂——若仍挂，启动序列方向穷尽） |
| 35 二分轮：关闭 34 两补齐 = 精确 33 状态 | 933a694b6ad2e6f1183d5643875d5816680d16ef9e42a350cccbc013cd9a6cae | ❌ 实判：仍挂 → 34 补齐无罪、32 pinctrl 实锤 → 追凶到 ufshcd_init:11308 无条件设备复位 |
| 36 真修复：device_reset_ctrl 加 link_started 门（首次链路训练成功前不脉冲）+ 恢复 34 两补齐 | 46206affec43a6046059f5d71684af83d7f88e6ae0529109c80deb1799d456d8 | ✅ **实判：回归完全修复**（6 LUN+36.9s init）；READ 楔死原样——启动序列/DT 方向穷尽 |
| 37 READ 仪器化三件套：160 条提交/完成轨迹 + 首 3 READ doorbell/OCS 时间线 + LU 激活后受控 READ 矩阵（两轮 review+下游对照共修 7 处：done 门控/sg_set_buf 垃圾页/打印失真/级联标注/掩码/timeout 单位 msecs_to_jiffies/outstanding 幽灵位清位） | 4315a6a8…（完整 sha 见 ufs-bringup.md §7；9bbaa55e/e5aeb9e1 作废未刷） | ✅ **实判：三假设灭**（QUNIPRO 常态、bank2 全通、完美自制 READ+bank1 也挂）+ 新事实（abort <1ms 秒回、并发命令照常、≥64B 通/4KB 挂）→ ICE 时钟线索 |
| 38 ICE 时钟根因修复（用户指示 live 实频）：gcc-sm7250 照下游 gcc-lito 移植 gcc_ufs_phy_ice_core_clk_src RCG（0x7706c、37.5/75/150/300M、id 112）+ 分支接线 + DT 10 时钟形态 + freq-table live 逐对实频 | f0ff4f43…（0-0 中间版 7c05fd3e 作废未刷） | ❌ **实判：ICE 假设证伪**——probe 零 rate 错（RCG✅ ICE 300M 实使能）但 READ 原样楔死；**时钟轴穷尽**。遗产：freq-table 与 live 逐对一致（保留）、tx"有意偏差"确认为迭代 32 误读化石（已更正） |
| 39 到达性判别矩阵 v2（P39）：A 基线 / **B 越界 LBA**（同形 CDB，秒回 CHECK CONDITION=命令通路好）/ C 未配置 LUN6 / D 写 bMaxNumOfRTT=8+回读；R3 review 修 D 无效实验+实证 query_attr 自持锁 | 2590c6fc…（中间版 d5107e0e 作废未刷） | ⏳ 待实机。分叉判据：B/C 完成→楔死在数据相位/NAND（设备侧）；也挂→命令未离开主机（取指引擎）。quirk/tune 轴同轮审结（UniPro≥1.6 两树空转、VS_DEBUG 只 hisi） |

---

## 6. 内核侧改动清单

树：`sdm765_sm7250_linux-6.17.0/`（.config 原始备份：`.config.bak-*`）

### 6.1 配置变更（scripts/config）

```bash
# 串口（注意 QCOM_GENI_SE 是硬依赖，缺它 olddefconfig 会静默丢弃串口选项）
scripts/config --enable QCOM_GENI_SE --enable SERIAL_EARLYCON \
  --enable SERIAL_QCOM_GENI --enable SERIAL_QCOM_GENI_CONSOLE

# 内嵌 initramfs（文件名必须以 .cpio.gz 结尾，否则 usr/Makefile 按文件清单解析报错）
# 迭代 13 起：aarch64 busybox+musl（unpacked_boot_img/ramdisk 是 x86-64，ENOEXEC）
bash ../scripts/make-initramfs.sh
scripts/config --set-str INITRAMFS_SOURCE \
  "/home/yucong/Documents/redmi_k30_5g/builds/copydown/initramfs.cpio.gz"

# cmdline（显式 earlycon 地址 + FORCE；调试期再加 ignore_loglevel initcall_debug）
scripts/config --set-str CMDLINE \
  "earlycon=qcom_geni,0x888000 console=ttyMSM0,115200 rw ignore_loglevel initcall_debug"
scripts/config --enable CMDLINE_FORCE
```

### 6.2 源码改动

| 文件 | 改动 | 状态 |
|---|---|---|
| `drivers/clk/qcom/clk-rpmh.c` | 新增 `clk_rpmh_sm7250` 表 + `qcom,sm7250-rpmh-clk` match 项。迭代 13 收缩为仅 `bi_tcxo/bi_tcxo_ao`（"xo.lvl"）——lito cmd-db 比 kona 精简（qnoc 缺 SN11/SN14/SH10/SH8），lnbb/rf 资源不保证存在，而 clk-rpmh 缺资源是整体 -ENODEV；WiFi 阶段再按需增补 | 保留（后续可考虑提上游） |
| `drivers/pinctrl/qcom/pinctrl-sm7250.c` | 迭代 13：去掉照抄 sc7180 的 north/south 双 tile 模型（lito TLMM 是单一 0xf000000 区域；tiles 模式下 msm_pinctrl_probe 按名字找 reg，DT 无名 reg → "invalid resource (null)" -EINVAL）。pin group 全部 `.tile = 0`，走 index-0 单区域路径 | 保留 |
| `drivers/clk/qcom/gcc-sm7250.c` | 迭代 14：① GDSC 旗标清零（原 `VOTABLE\|ALWAYS_ON` 导致 probe 时强制给未上电的 MMU TBU 投票 → 静默复位）；② GPLL0/6/7 FABIA→LUCID + enable_reg 0x52000→0x52010 + postdiv lucid ops（原类型错导致 rate 计算错） | 保留（PLL 偏移/表仍需对照下游全量核对，gpll7@0x27000 未必存在） |
| `drivers/watchdog/qcom-wdt.c` | ① match 表加 `qcom,apss-wdt-sm7250/sm8250 → &match_data_apcs_tmr`（修复 kpss 回退的布局错误）；② probe 末尾无条件 `writel(0, WDT_EN)` + `pr_info("qcom_wdt: probed, watchdog disabled for bringup")` | 保留（bring-up 用，量产前应改为正常接管喂狗） |
| `kernel/dma/pool.c` | `atomic_pool_expand` 调试打点（gfp/order/page/prep/remap/decrypted/gen_pool 各步骤） | **调试用，确认稳定后删除** |
| `arch/arm64/mm/dma-mapping.c` | `arch_dma_prep_coherent` 调试打点（va/phys/size/done） | **调试用，确认稳定后删除** |

### 6.3 配置变更（迭代 12 新增）

```bash
# rpmh 时钟驱动（GCC 的 bi_tcxo 父时钟来源；缺了 GCC 永远 -517，串口无法 probe）
scripts/config --enable QCOM_CLK_RPMH
make ARCH=arm64 LLVM=1 olddefconfig
```

已确认编入的 sm7250 相关项：`SM_GCC_7250`、`PINCTRL_SM7250`、`QCOM_RPMH`、
`QCOM_RPMHPD`、`QCOM_CLK_RPMH`、`INTERCONNECT_QCOM_SM7250`、`QCOM_GENI_SE`、
`SERIAL_QCOM_GENI(_CONSOLE)`、`QCOM_SCM`、`QCOM_SMEM`。

### 6.4 构建命令

```bash
cd /home/yucong/Documents/redmi_k30_5g/linux-7.3-rc6
export PATH=$WORK/tools/lld-21/usr/lib/llvm-21/bin:/usr/lib/llvm-21/bin:$PATH
make ARCH=arm64 LLVM=1 -j16 Image
# 验证串口驱动真的进去了：
strings arch/arm64/boot/Image | grep -c "geni-debug-uart\|ttyMSM0\|qcom_geni"
```

### 6.5 已知问题（内核侧）

- ~~`100000.clock-controller`（GCC）一直 -517~~ → **迭代 12 已定位修复**：三层根因
  （`CONFIG_QCOM_CLK_RPMH` 未开 + clk-rpmh 无 sm7250 表 + 见 §7 的 DT 项），待实机验证；
- ~~`18200000.rsc:regulators-1 returned 19`（-ENODEV）~~ → **迭代 12 已修复**：PM8150L
  的 cmd-db 资源前缀是 "c" 不是 "l"（见 §5 迭代 12），待实机验证；
- 无 sm7250 QMP-UFS-phy 驱动（ufshc 暂禁用，rootfs 阶段需移植）；
- cpufreq-hw 无 sm7250 表（节点不生效，CPU 无调频，不影响 shell）。

---

## 7. runtime DTB 改动清单

文件：`builds/copydown/runtime-dtb-stdout.dtb`（由 `unpacked_boot_img/dtb`
经 dtc 往返 + 编辑生成，**编辑后的源 dts 固化为 `scripts/runtime-dtb-picasso.dts`**）。

| 改动 | 内容 |
|---|---|
| `/chosen/stdout-path` | 新增 `"serial0:115200n8"`（serial0 别名 = serial@888000） |
| `/memory@80000000` | 占位符 size=0 替换为三段真实布局（ABL RAM Partitions） |
| reserved-memory | 补 10 个固件 no-map 区（见 §5 迭代 7 表）；迭代 12 再补 3 处（xbl 前段/uefi log/reserved_pil），现共 **22 条** |
| `/soc/iommu@3da0000` | `status = "disabled"`（GPU CX GDSC 未上电读寄存器挂死） |
| 迭代 12：serial@888000 | compatible 改 `qcom,geni-debug-uart`（注册 ttyMSM0 console；引脚 gpio36/37/qup02 与 live dump 核对一致） |
| 迭代 12：RSC 子节点 | rpmh-clk 保持 `qcom,sm7250-rpmh-clk`（内核已加表）；rpmhpd 改 `qcom,sm7150-rpmhpd`（lito cmd-db 无 mmcx） |
| 迭代 12：RPMh regulators | PM8150L pmic-id `"l"`→`"c"`（cmd-db 实名 ldoc\*/smpc\*/bobc）；删 pm7250b 组（cmd-db 无 b 系资源）与虚构 pm8150a 组；l7c 并入 PM8150L |
| 迭代 12：禁用防挂死 | MDSS 整树、dispcc、gpucc、UFS+phy（无驱动/无 panel/域未上电，详见 §5 迭代 12） |

### 生成命令（复现）

```bash
cd /home/yucong/Documents/redmi_k30_5g
# 已固化的源 dts: scripts/runtime-dtb-picasso.dts（含全部 4 类修改），直接编译：
dtc -I dts -O dtb scripts/runtime-dtb-picasso.dts -o builds/copydown/runtime-dtb-stdout.dtb
# 验证
dtc -I dtb -O dts builds/copydown/runtime-dtb-stdout.dtb | \
  sed -n '/reserved-memory {/,/^	};/p' | grep -c no-map   # 应为 22
```

> 如需从零重建该 dts：`dtc -I dtb -O dts unpacked_boot_img/dtb` 反编译后，
> 依次做 4 类修改（① chosen 加 stdout-path；② memory@80000000 的 reg 替换为
> 三段真实布局；③ reserved-memory 补 10 个 no-map 区；④ iommu@3da0000 加
> `status = "disabled"`），再编译。dtc 往返 diff 验证：除手工新增行外与原始
> DTB 完全一致（phandle/别名无损）。

dtc 往返 diff 验证：除手工新增行外与原始 DTB 完全一致（phandle/别名无损）。

---

## 8. 复现手册（从零到 boot.img）

> **一键脚本**：`bash scripts/build-boot.sh`（自动完成 §8.1-8.4 全部步骤 + 校验），
> 详见 [build-boot-manual.md](build-boot-manual.md)。以下为分步命令（排查用）。

### 8.0 前置

> 2026-10-11 起默认内核树为 linux-7.3-rc6（6.17 树已移除；迭代 0-25 的
> 历史命令原记 6.17 路径，复现时按此替换）。

- 解锁的 picasso + fastboot；UART（115200n8，SE2）接到 0x888000 对应触点；
- 工作目录 `/home/yucong/Documents/redmi_k30_5g/`，内核树
  `sdm765_sm7250_linux-6.17.0/`（已完成 §6 全部改动）；
- 工具链：`arm-gnu-toolchain-15.2`（aarch64-none-elf-*）、clang 21 + ld.lld
  （`/tmp/lld-extract/usr/lib/llvm-21/bin/ld.lld`）、`/usr/bin/dtc`、
  `/usr/bin/mkbootimg`。

### 8.1 构建内核（改内核后）

```bash
cd /home/yucong/Documents/redmi_k30_5g/linux-7.3-rc6
export PATH=$WORK/tools/lld-21/usr/lib/llvm-21/bin:/usr/lib/llvm-21/bin:$PATH
make ARCH=arm64 LLVM=1 -j16 Image
strings arch/arm64/boot/Image | grep -c geni   # 必须 > 0
```

### 8.2 生成 runtime DTB（改 DTB 后）

见 §7 的命令序列。

### 8.3 打包 boot.img（每次修改后）

```bash
cd /home/yucong/Documents/redmi_k30_5g
LINUX_IMAGE=sdm765_sm7250_linux-6.17.0/arch/arm64/boot/Image \
RUNTIME_DTB=builds/copydown/runtime-dtb-stdout.dtb \
bash scripts/mkboot-linux-copydown-picasso.sh
```

### 8.4 shim 改动的强制校验

```bash
TC=/home/yucong/Application/arm-gnu-toolchain-15.2.rel1-x86_64-aarch64-none-elf/bin
$TC/aarch64-none-elf-objdump -d builds/copydown/copydown-shim.elf
# 逐项核对（教训：立即数地址必须看 objdump 反汇编结果，不能只看源文件）：
#  - shim_entry/alt_entry → copy_entry 的 adrp 目标
#  - copy_entry: x0=_start, x1=0x200000, x2=linux_image_end
#  - el_drop: msr hcr_el2/sctlr_el1/spsr_el2/elr_el2/eret
#  - WDT: mov x2,#0x17c10000 + str wzr,[x2,#8]/[x2,#64]
#  - uart_putc: 0x888000 构造对 + 0x40/0x610/0x604/0x618/0x270/0x600/0x700 偏移
#    （迭代 21 起为 poll_put_char 对齐版：等 CMD_DONE + CANCEL 超时恢复）
```

### 8.5 刷写测试

```bash
fastboot boot boot-linux-copydown-picasso.img
# 预期 UART: SCK → "Booting Linux on physical CPU..." → ... → Run /init
```

### 8.6 常用诊断开关

- shim 探针：'S'/'A'=入口与 EL 降级、'C'=拷贝完成、'K'=跳内核前；
- 内核：`initcall_debug`（定位挂死的 initcall）、`ignore_loglevel`；
- `atomic_pool_expand`/`arch_dma_prep_coherent` 打点（§6.2，稳定后删）。

---

## 9. 当前状态与遗留问题

### 9.1 已打通

```
ABL（overlay 校验通过） → copydown shim（EL2→EL1 降级、停狗、拷贝、缓存、x0=主线DTB）
→ 主线 7.3-rc6 内核（earlycon → 8 核 SMP、6GB、cmd-db → rpmh-clk → rpmhpd → GCC 全表
→ geniqup → ttyMSM0 全功能 console → clk_ignore_unused → of_platform_sync_state
→ Run /init → initramfs busybox shell → ✅ UART 交互命令行（2026-10-09 达成）
  迭代 25：GPU catalog 0x06020002 + GMU/zap 固件（YAAP ROM）+ merge_3d 基址修复
  + qfprom/nvmem-cell/smmu-clocks/DSI-reparent 修复 + dispcc sleep_clk 0-cell
  根因修复 → **DPU 6.1 → DSI → G7A DSC 面板 → fbcon 出图，屏幕点亮（2026-10-11）**
  迭代 27：GMU bw 表/PDC/vrm.soc + **gfx3d 根时钟内核补刀（ROOT_EN 组合拳）**
  + **zap 装载区 4K 对齐动态分配** → **GPU 全链打通：CP/SQE 执行 → zap 认证 →
  fb0: msmdrmfb → msm 渲染管线首次真实出图（2026-10-12）**
  迭代 28：NT36672C 触摸主线化移植（下游驱动 gpiod/proc_ops 化 + SE7 SPI
  PIO + 固件入 initramfs + EVDEV）→ **/dev/input/event1 触摸数据（2026-10-13）**
  迭代 29：sm7250 QMP-UFS-PHY 表移植入主线 qmp-ufs 驱动（下游 v4-lito
  值；TX/RX 尾部偏移 lito 专属；serdes/hs_b 复用 sm8150 逐值同）+
  ufshc/phy 节点启用 + iommus 双 sid → **UFS 枚举待实机验证（2026-10-13）**
```

### 9.2 遗留

| 项 | 说明 |
|---|---|
| 迭代 18 待实机验证 | `clk_ignore_unused` 后能否通过 clk_disable_unused → genpd/regulator late initcall → `Run /init` → busybox shell |
| 时钟表幻影排查 | ~~进行中~~ gpll7(0x27000) 已定位移除（迭代 17 实锤）；全表已通过注册 |
| clk_disable_unused 复位 | 迭代 18 用 `clk_ignore_unused` 绕过；根治 = 按上游惯例给 AHB 配置时钟补 CLK_IS_CRITICAL 后移除该参数 |
| qup s2 RCG WARNING | `rcg didn't update its configuration`——偏移存疑，波特率实际沿用 ABL 配置；改波特率前必须修 |
| pinctrl 永久 -517 | ~~原因未明~~ → 迭代 19 定位：gpio-ranges 自引用 defer 死循环，已删（tlmm + PM8150L gpio 两处），待实机验证 probe 是否成功 |
| qnoc 缺 BCM | SN11/12/14/15、SH8/SH10 不在本机 cmd-db（警告非致命）；将来按需裁表 |
| ~~GCC -517~~ | **迭代 12/13 已定位并修复**：CONFIG_QCOM_CLK_RPMH 未开 + clk-rpmh 无 sm7250 表（13 收缩为仅 bi_tcxo），待实机验证 |
| ~~regulators-1 -ENODEV~~ | **迭代 12 已修复**（PM8150L 的 cmd-db 前缀是 "c" 不是 "l"），待实机验证 |
| ~~initramfs ENOEXEC~~ | **迭代 13 已修复**（x86-64 busybox → aarch64 busybox+musl，加 /dev/console 节点；`scripts/make-initramfs.sh` 生成） |
| UFS | 迭代 29 移植 QMP-UFS-PHY 并打通 6 LUN 枚举；迭代 41 的 L7C/L5A/L9A HPM + qphy.lvl 投票实机修复 4KB READ(10)、分区扫描与数据读回。调试/测试代码已清理并本地构建通过；遗留：清理版实机回归、HPM/QPHY 贡献分拆、rootfs 挂载策略 |
| rpmhpd 专用表 | 目前借用 sm7150 集（无 EBI/无 MMCX）；后续给 rpmhpd.c 加 sm7250 描述符 |
| 显示（MDSS/DPU/panel） | 迭代 20 已全链 probe 通过（DPU hw rev 0x60010000、dsi bound）；**modeset 挂死待 GPU 链解锁**——迭代 25 修复 catalog chipid + GMU/zap 固件后恢复 mdss，待实机验证 |
| GPU（adreno/gpucc/SMMU） | 迭代 25 修复：catalog 加 0x06020002（fix20 手算 0x06020200 翻车）+ a650_gmu.bin/a650_sqe.fw/a620_zap 固件入 initramfs（YAAP ROM 提取）；gpucc/gmu DT 资源 fix20 已就绪，待实机验证 |
| cpufreq-hw | 无 sm7250 表，节点不生效（CPU 无调频，不影响 shell） |
| initramfs | 当前为最小 bring-up 版；rootfs 挂载策略待完善 |
| wdt 驱动 | 当前 bring-up 模式直接停狗，量产前应恢复"接管 + 喂狗"（apcs_tmr 布局已修正） |
| panel reset（PM8150L GPIO3） | 若出现早期白屏，需在 shim 实现 SPMI 写事务（SID=4、外设 0xc000） |
| 调试打点清理 | pool.c / dma-mapping.c 的 pr_info 确认稳定后删除 |

---

*文档生成于 picasso 主线 Linux bring-up 过程，随每次迭代更新。*

> **2026-10-11 起的工作约定**：6.17 内核树已彻底移除，默认树 =
> `linux-7.3-rc6`；构建产物统一命名 `boot_7.3.img`（不再使用
> boot-linux-copydown-picasso.img 及 builds/ 下的变体名）；构建入口
> `bash scripts/build-boot.sh`（无需环境变量）。

> **2026-10-13 起的数据源等级（用户指令，必须遵守）**：
> 1. 代码与设备树结构 → 以下游 4.19 内核/dtsi（用户提供的树）为第一权威；
> 2. 需要运行时/板级事实时 → **以用户提供的 `live_dts.dts` 为唯一权威**
>    （工作区根目录），不得以其他 adb 自取的运行时 dump 等替代；
> 3. **不擅自通过 adb 从实机获取数据**；确有必要时先向用户说明并获得同意。
> 教训（迭代 29 实锤）：`scripts/runtime-dtb-picasso.dts`（历史会话另一份
> 运行时 dump）引入了 ufshc 两个错误——sid 0xe0（实为 USB 的流）与
> reset-gpios=<tlmm 46>（实为 qup13/qdss 引脚）；而用户的 live_dts.dts
> 本来就没有这两个错误属性。**凡与 live_dts.dts 冲突的历史 dump 值一律
> 以 live_dts.dts 为准**；`runtime-dtb-picasso.dts` 仅作历史参考，不再
> 作为取值来源。

