# build-boot.sh 使用手册（一键生成 boot.img）

> 配套技术文档见 [copydown-shim-port.md](copydown-shim-port.md)。
> 本手册面向"改完东西只想重新打包"的场景：脚本自动完成内核构建、
> DTB 编译（树内源）、shim 构建、boot.img 打包与全量校验，任何依赖缺失
> 都会给出明确的错误提示和修复命令。

---

## 1. 一键用法

```bash
cd /home/yucong/Documents/redmi_k30_5g
bash scripts/build-boot.sh              # 完整构建（默认）
bash scripts/build-boot.sh --skip-kernel  # 只改了 DTB/shim/打包参数时用（秒级）
```

环境变量：

| 变量 | 默认 | 作用 |
|---|---|---|
| `KERNEL_JOBS` | 16 | 内核 make 并行度（机器内存小可调低） |

产物：`boot_7.3.img`（工作目录根下）。

刷写测试：

```bash
fastboot boot /home/yucong/Documents/redmi_k30_5g/boot_7.3.img
# 预期 UART: SCK → Booting Linux on physical CPU → … → Run /init
# 预期 USB: 电脑 /dev/ttyACM0 出现（CDC-ACM）→ picocom /dev/ttyACM0 交互 shell
```

---

## 2. 脚本流程（5 步）

```
步骤 0  依赖检查（缺什么报什么，含修复命令）
步骤 1  构建内核  make ARCH=arm64 LLVM=1 -j16 Image（增量，未改动秒级）
         → 自动验证产物里串口驱动字符串 > 0（历史大坑：静默无输出）
步骤 2  编译 DTB（树内源 sm7250-xiaomi-picasso.dts → builds/copydown/，迭代 24 起唯一源；dtc 直编）
         → 自动验证 no-map 保留区 = 22 条（少了内核会踩 TZ/固件内存挂死）
步骤 3  构建 shim + 打包（内部调用 mkboot-linux-copydown-picasso.sh）
步骤 4  校验 manifest（全部断言必须 True）
         → 反汇编抽查 shim 含 0x17c10000 停狗代码
步骤 5  打印 sha256 + 刷写命令
```

---

## 3. 依赖文件清单

### 3.1 输入文件（脚本会逐一检查，缺失即报错）

| 文件 | 路径（相对工作目录） | 作用 | 来源 |
|---|---|---|---|
| 原厂 boot.img | `boot_factory.img` | dtb 字段提取源（offset 0x21ae000，1863938 B，12-DTB blob；ABL overlay 校验必需） | 设备原厂镜像，直接可用 |
| initramfs 源 | `builds/copydown/initramfs.cpio.gz` | 内核内嵌 initramfs（shim 换 DTB 后外部 ramdisk 不可达，必须内嵌）。**aarch64 busybox+musl + FastRPC sensor bootstrap + GPU/ADSP/触摸固件**，由 `scripts/make-initramfs.sh` 生成（勿再用 `unpacked_boot_img/ramdisk`——那是 x86-64 busybox，在 aarch64 上 ENOEXEC(-8)）。`/bin/fastrpc-sensor-attach` 是静态 AArch64 helper，用于复刻下游 `sscrpcd sensorspd` 的 create/attach 契约 | `bash scripts/make-initramfs.sh` 生成；helper 源码在 `scripts/fastrpc-sensor-attach.c` |
| ADSP 分段固件 | `scripts/initramfs/firmware/qcom/sm7250/adsp.{mdt,b00-b32}` | 音频 ADSP PAS 固件；`make-initramfs.sh` 自动打包到 `/lib/firmware/qcom/sm7250/` | `android_firmware_dump/firmware_mnt/image/` 或 `scripts/extract-adsp-firmware.sh modem.img` |
| shim 模板 | `scripts/copydown-shim.S.in` | copydown shim 汇编（含 EL2 降级、停狗、UART 探针） | 本项目 |
| DT 源（唯一） | `linux-7.3-rc6/arch/arm64/boot/dts/qcom/sm7250-xiaomi-picasso.dts` | 迭代 24 起：runtime dts 整体覆盖而来的自包含权威源（迭代 0-23 实机形态，显式 phandle）。`scripts/runtime-dtb-picasso.dts` 已退役为**对照参考**，不参与构建；原 dtsi 结构在 `sm7250-xiaomi-picasso.dts.dtsi-based.bak` | 本项目 |
| 打包脚本 | `scripts/mkboot-linux-copydown-picasso.sh` | shim 构建 + mkbootimg + manifest | 本项目 |
| 内核树 | `linux-7.3-rc6/Makefile`、`.config` | 主线 7.3-rc6 内核（**默认树，6.17 已于 2026-10-11 移除**；.config 必须已含 §3.3 的配置项） | 用户内核树 |
| initramfs 副本 | `builds/copydown/initramfs.cpio.gz` | `CONFIG_INITRAMFS_SOURCE` 指向的文件（文件名必须以 `.cpio.gz` 结尾，否则内核构建按文件清单解析报错） | 脚本自动从 ramdisk 复制，被清理后需手动恢复 |

### 3.2 工具链（路径固定，缺失时脚本给出获取方法）

| 工具 | 路径/命令 | 用途 | 获取方法 |
|---|---|---|---|
| aarch64-none-elf-gcc / ld / objcopy | `/home/yucong/Application/arm-gnu-toolchain-15.2.rel1-x86_64-aarch64-none-elf/bin/` | shim 汇编/链接/取裸二进制（不能用宿主 clang 直接 objcopy：MC 对 adrp 同类符号也发重定位） | ARM GNU 工具链 |
| clang | PATH 中 `clang` | 内核构建（LLVM=1） | 系统包 |
| ld.lld（仅 7.x host 工具需要） | `tools/lld-21/usr/lib/llvm-21/bin/ld.lld` | **目标端内核链接一律 `aarch64-none-elf-ld`（用户规则，脚本内置）**。ld.lld 仅供内核**自身默认的 HOSTCC**（fixdep 等 host 工具强制 `-fuse-ld=lld`，内核 Makefile 行为，不对抗）——存在时脚本自动加入 PATH；缺失时 HOSTCC 环节构建失败（由用户决定提供方式）。⚠️ `/tmp` 会被系统清理，固定放 `tools/lld-21` | `cd /tmp && apt-get download lld-21 && mkdir -p tools/lld-21 && dpkg-deb -x lld-21*.deb tools/lld-21`（免 root） |
| llvm-objcopy | `/usr/lib/llvm-21/bin/llvm-objcopy` | 内核 LLVM 工具链 | 系统包 |
| dtc | PATH 中 `dtc` | runtime DTB 编译 | 系统包 |
| mkbootimg | PATH 中 `mkbootimg` | boot image 打包（支持 header v2） | Ubuntu 包 |
| python3 / make | PATH 中 | 脚本与构建 | 系统自带 |

### 3.3 内核 .config 必需配置（脚本会在步骤 0 检查）

```
CONFIG_SERIAL_QCOM_GENI=y            # 串口驱动（缺依赖会被 olddefconfig 静默丢弃！）
CONFIG_QCOM_GENI_SE=y                # ↑ 的硬依赖
CONFIG_SERIAL_EARLYCON=y
CONFIG_SERIAL_QCOM_GENI_CONSOLE=y
CONFIG_QCOM_CLK_RPMH=y               # rpmh 时钟驱动（GCC 的 bi_tcxo 父时钟来源；缺了 GCC 永远 -517，ttyMSM0 无法 probe）
CONFIG_MAILBOX=y                     # AOSS QMP mailbox 框架（A620 ACD 必需）
CONFIG_QCOM_IPCC=y                   # SM7250 AOSS QMP mailbox 控制器
CONFIG_QCOM_AOSS_QMP=y               # qmp_get()/ACD 状态通知；缺了 GPU bind 连锁失败
CONFIG_DEVFREQ_THERMAL=y             # GPU devfreq cooling；缺了 cooling 注册必然 -EINVAL
CONFIG_SND=y                         # ALSA 基础
CONFIG_SND_SOC=y
CONFIG_SND_SOC_QCOM=y
CONFIG_SND_SOC_SM8250=y              # Picasso 复用 SM8250 QDSP6 machine 行为
CONFIG_SND_SOC_QDSP6=y
CONFIG_SND_SOC_WCD938X_SDW=y         # WCD9385 RX/TX SoundWire codec
CONFIG_SND_SOC_LPASS_RX_MACRO=y
CONFIG_SND_SOC_LPASS_TX_MACRO=y
CONFIG_SND_SOC_LPASS_VA_MACRO=y
CONFIG_SOUNDWIRE=y
CONFIG_SOUNDWIRE_QCOM=y
CONFIG_REGMAP_SOUNDWIRE=y
CONFIG_PINCTRL_LPASS_LPI=y
CONFIG_PINCTRL_SM8250_LPASS_LPI=y
CONFIG_REMOTEPROC=y
CONFIG_QCOM_Q6V5_PAS=y
CONFIG_RPMSG_QCOM_GLINK_SMEM=y
CONFIG_QCOM_APR=y
CONFIG_QCOM_PD_MAPPER=y
CONFIG_QCOM_ADSP_SENSOR_PD=y       # ADSP sensor_process 的 PDR ACK keepalive
CONFIG_QCOM_FASTRPC=y              # ADSP fastrpcglink-apps-dsp；live cb10..cb12 对应主线 reg/SID 3..5
CONFIG_QCOM_SMP2P_SLEEPSTATE=y     # ADSP sleepstate/sleepstate_see 与 apps-awake bit12
CONFIG_QRTR=y
CONFIG_QRTR_SMD=y                  # ADSP “IPCRTR” glink 通道，PDR/QMI 必需
CONFIG_CMDLINE_FORCE=y
CONFIG_CMDLINE="earlycon=qcom_geni,0x888000 console=ttyMSM0,115200 rw ignore_loglevel initcall_debug clk_ignore_unused"
CONFIG_INITRAMFS_SOURCE="<工作目录>/builds/copydown/initramfs.cpio.gz"
```

迭代 20 起显示链路新增（缺一则 PM8150L 子树/显示不工作）：

```
CONFIG_SPMI=y                        # SPMI 总线
CONFIG_QCOM_PDC=y                    # PDC wakeup 中断域（spmi/pinctrl/tsens 的中断父；缺了整链永久 -517）
CONFIG_SPMI_MSM_PMIC_ARB=y           # 高通 PMIC arbiter（spmi@c440000）
CONFIG_MFD_SPMI_PMIC=y               # qcom,spmi-pmic（pmic@4/@5 设备）
CONFIG_PINCTRL_QCOM_SPMI_PMIC=y      # pm8150l gpio（面板 reset-gpios 消费者）
CONFIG_SM_DISPCC_7250=y              # dispcc-sm7250（下游 dispcc-lito 移植）
CONFIG_REGULATOR_QCOM_LCDB=y         # PM8150L LCD bias（面板 avdd/avee）
CONFIG_DRM_MSM=y                     # 显示栈（含 DPU/DSI/7nm PHY/fbdev）
CONFIG_DRM_PANEL_NOVATEK_NT36672C=y  # G7A 面板驱动（文件名 nt36672c，实为 G7A DSC）
CONFIG_BACKLIGHT_QCOM_WLED=y         # PM8150L WLED 背光
```

迭代 22-23 起 USB gadget console（启动早期输入通道，电脑 /dev/ttyACM0）：

```
CONFIG_USB_G_SERIAL=y                # CDC-ACM 串口 gadget → /dev/ttyGS0
CONFIG_PHY_QCOM_USB_SNPS_FEMTO_V2=y  # USB2 HS phy（注意符号带 USB_ 前缀）
# CONFIG_USB_ETH 禁用                # legacy g_ether 会抢 UDC（单 UDC 单 gadget，
                                     #   抢先绑定后 g_serial 只能 couldn't find UDC）
```

### 3.4 生成产物

| 产物 | 路径 |
|---|---|
| **最终 boot.img** | `boot_7.3.img` |
| 内核 Image | `linux-7.3-rc6/arch/arm64/boot/Image` |
| runtime DTB | `builds/copydown/runtime-dtb-stdout.dtb`（由树内 dts 编出） |
| 原厂 dtb 字段 | `builds/copydown/stock-dtb-field.img` |
| shim 中间件 | `builds/copydown/copydown-shim.{S,o,elf,bin}` |
| manifest | `builds/copydown/boot_7.3.manifest` |

---

## 4. 常见错误对照

| 报错 | 原因 | 解决 |
|---|---|---|
| `缺少输入文件: boot_factory.img` | 原厂 boot.img 不在工作目录 | 从设备/官方线刷包取回原厂 boot.img |
| `缺少 ld.lld（内核链接器）` | lld 未提取且 aarch64-none-elf-ld 也不可用 | 按脚本提示的 apt-get download + dpkg-deb -x 两步执行；或确认 ARM GNU 工具链路径（正常情况下会自动回退，不报此错） |
| `.config 未设置 CONFIG_INITRAMFS_SOURCE` | 内核未内嵌 initramfs | 按脚本提示的 cp + scripts/config 命令执行 |
| `CONFIG_INITRAMFS_SOURCE 指向的文件不存在` | initramfs 副本被清理 | `cp unpacked_boot_img/ramdisk builds/copydown/initramfs.cpio.gz` |
| `缺少 CONFIG_SERIAL_QCOM_GENI=y` | 串口选项被 olddefconfig 丢弃（缺 QCOM_GENI_SE 依赖时必现） | 按脚本提示的 scripts/config 三连 + olddefconfig |
| `构建产物里没有串口驱动` | 同上，但发生在旧 Image 上 | 重新构建内核（不带 --skip-kernel） |
| `no-map 保留区应为 22 条，实际 N 条` | 树内 sm7250-xiaomi-picasso.dts 被改动 | 对照 copydown-shim-port.md §5 迭代 7（10 个固件区）+ 迭代 12（xbl 前段/uefi log/reserved_pil）补齐 |
| `manifest 检查未通过: …=False` | shim 布局断言失败（如内嵌 Image 不在 2MB 对齐处） | 检查模板 .balign 0x200000 是否被改动 |
| `shim 里没找到 0x17c10000 停狗代码` | shim 模板的 el_drop 被改动 | 恢复模板（`movz x2, #0x17c1, lsl #16` + 两处 `str wzr`） |

---

## 5. 注意事项

1. **改 shim 模板后必须看反汇编**：立即数地址类 bug 源文件看不出来（历史教训：
   `movz #0xc100`+`movk #0x17` 拼出的是 0x17c100 而非 0x17c10000，直接把 shim
   挂死在未映射 MMIO 上）。脚本步骤 4 已内置 0x17c10000 抽查，其他地址改动
   请自行 `objdump -d builds/copydown/copydown-shim.elf` 核对。
2. **改内核 config 后必须跑完整构建**（不带 --skip-kernel）：`olddefconfig`
   会静默丢弃依赖不满足的选项（串口选项被丢过两次）。
3. **构建是确定性的**：相同输入 + **同一链接器**产生相同 sha256，可据此判断"是否真的变了"
   （注意：ld.lld 与 aarch64-none-elf-ld 的产物不同，见 §3.2 回退说明）。
4. 内核源码里的调试打点（`kernel/dma/pool.c`、`arch/arm64/mm/dma-mapping.c`）
   和 wdt 停狗 patch 是 bring-up 状态，稳定后按 copydown-shim-port.md §9 清理。
