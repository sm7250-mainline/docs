# picasso（SM7250/lito）NT36672C 触摸主线 bring-up 全记录

> 2026-10-12 ~ 10-13，迭代 28。从零移植到**触摸事件流**（2026-10-13，
> `/dev/input/event1` 出真实触摸数据，产物 sha `a6292420…`）。本文记录
> 完整技术路径：实机取证如何定型方案、下游驱动如何主线化（每个文件
> 首次如何编写）、两轮刷机各暴露什么问题、如何排查与解决。配套迭代
> 日志见 `copydown-shim-port.md` 迭代 28（逐轮实录）；显示/GPU 侧另见
> [display-bringup.md](display-bringup.md) / [2026-10-12-gpu-bringup.md](2026-10-12-gpu-bringup.md)。

---

## 目录

1. [硬件与软件架构](#1-硬件与软件架构)
2. [阶段一：三源侦查与实机取证（方案定型）](#2-阶段一三源侦查与实机取证)
3. [阶段二：固件来源（ROM 解包工具链重建 + ihex 转换）](#3-阶段二固件来源)
4. [阶段三：驱动主线化移植（首次编写）](#4-阶段三驱动主线化移植)
5. [阶段四：DT 与 config 落码](#5-阶段四dt-与-config-落码)
6. [阶段五：第一刷——probe 全链一次通过 + 两个收尾问题](#6-阶段五第一刷)
7. [阶段六：第二刷——修复与达成](#7-阶段六第二刷修复与达成)
8. [最终形态：保留的代码修改](#8-最终形态保留的代码修改)
9. [验证方法（含 busybox 工具坑）](#9-验证方法)
10. [方法论与教训](#10-方法论与教训)
11. [遗留与展望](#11-遗留与展望)

---

## 1. 硬件与软件架构

### 1.1 触摸链路总览

picasso 的触摸是 **TDDI**（Touch and Display Driver Integration，触控
与面板驱动合一）方案——触摸控制器物理上与 G7A 面板驱动融合，但走
**SPI** 总线而非 I2C：

```
 内核侧                                硬件侧
 ────────                              ────────
 geniqup@9c0000 (wrapper1, okay)       QUPv3 wrapper1 @0x9c0000
  └ spi@984000 (qcom,geni-spi, SE7) ──▶ SE7 @0x984000（GIC SPI 354）
     │ clocks: GCC_QUPV3_WRAP1_S1_CLK(53)；PIO/FIFO 模式（无 dmas）
     │ pinctrl: gpio6/7/8 = qup11（MOSI/MISO/CLK）
     │           gpio9   = qup11（CS，由 ts 驱动的 active 组切换）
     │
     └ touch_spi@0（"xiaomi,spi-for-tp"）    ← spi-xiaomi-tp 垫片设备
          │ spi_device 转交
          ▼
 ts_novatek（/soc 下 platform 节点，"novatek,NVT-ts-spi"）── NT36672C
     │ late_initcall probe                    （TDDI，与 G7A 同体）
     │ reset = TLMM gpio4（gpiod，OUT）
     │ irq   = TLMM gpio5（gpiod→irq，Edge rising，TLMM 汇总 GIC 208）
     │ SPI: 5MHz（probe/固件）、10MHz（垫片上限）
     │
     └ input/evdev → /dev/input/event1 → 触摸事件流
```

### 1.2 架构的两个特殊点（排障先验）

**① ts_novatek 不是 SPI 子设备**。下游把触摸驱动做成 `/soc` 下的
platform driver（`late_initcall` + `platform_driver_register`），probe 里
经 `tmp_hold_ts_xsfer()/get_ts_xsfer()` 从垫片（`drivers/misc/spi-xiaomi-tp.c`，
98 行，匹配 `"xiaomi,spi-for-tp"` 的 SPI driver，把 spi_device 存起来
转交）取得 SPI 通道。**因此 fw_devlink 不会跟踪 ts_novatek 与 SPI
控制器/垫片的 probe 顺序**——驱动侧必须自行容错（见 §4 的 defer 化）。

**② 固件是 RAM fw 模型**。NT36672C 的固件由 host 在启动后 ~14s
（`Boot_Update_Firmware` delayed work）经 SPI 下载进 IC RAM；面板匹配
（选哪一份 fw）依赖 8 字节 lockdown 信息——下游由 **DSI 读面板**得来
（`dsi_panel_lockdown_info_read`），主线 msm DRM 无此函数。IC 的 flash
里另有出厂固件常驻（本次实机取证的关键事实，见 §2）。

### 1.3 引脚与供电（live dts 实测）

| 引脚 | 功能 | pinctrl active | pinctrl suspend |
|---|---|---|---|
| TLMM gpio4 | TP reset | gpio, drive16, pull-up | gpio, drive16, bias-disable |
| TLMM gpio5 | TP irq | gpio, drive16, **pull-down** | gpio, drive16, bias-disable, input |
| TLMM gpio6/7/8 | SPI MOSI/MISO/CLK | qup11, drive6, bias-disable | gpio, drive6, bias-disable |
| TLMM gpio9 | SPI CS | **qup11**（ts 驱动切换）， drive6 | gpio, drive6, pull-down, output-low |
| PM8150L gpio3 | reset-tddi | 驱动**未使用**（下游 dts 有属性、驱动代码 grep 零引用，面板侧管理） | — |

供电无独立 regulator（TP 由面板 vddio 1.896V 域覆盖，`regulator_ignore_unused`
常开）。中断触发：驱动硬编码 `INT_TRIGGER_TYPE = IRQ_TYPE_EDGE_RISING`。

---

## 2. 阶段一：三源侦查与实机取证

### 2.1 下游源码侦查

```
Xiaomi_Kernel_OpenSource-picasso-r-oss/
├── drivers/input/touchscreen/nt36672c/        ← picasso 专用版（本次移植源）
│   ├── nt36xxx.c             3345 行  主驱动（platform + SPI 传输 + input）
│   ├── nt36xxx_fw_update.c   1135 行  RAM fw 下载流程
│   ├── nt36xxx_ext_proc.c     741 行  /proc 接口（fw 版本/lockdown/raw…）
│   ├── nt36xxx_mp_ctrlram*.c 3100+ 行 工厂 MP 校准（未移植）
│   ├── nt36xxx.h / nt36xxx_mem_map.h
├── drivers/misc/spi-xiaomi-tp.c                ← SPI 垫片（98 行）
├── include/linux/spi-xiaomi-tp.h
└── firmware/novatek_nt36672c_g7a_{fw,mp}0{1,2,3}.bin.ihex + novatek_ts_fw.bin.ihex
```

注意 `nt36xxx/`（通用版，`CONFIG_TOUCHSCREEN_NT36XXX`）与 `nt36672c/`
（picasso 版，`CONFIG_TOUCHSCREEN_NT36xxx_HOSTDL_SPI`）并存——**原厂
factory config 两者都 not set**（触摸驱动在 Android 侧是 vendor 模块或
ROM 自编译内核内建），我们自建内核直接 =y 移植 picasso 版。

### 2.2 实机取证（本轮最关键的决策输入）

实机当时正跑 **YAAP Android 16**（getprop 显示的 MIUI V13 fingerprint
是 ROM 保留的兼容标识，用户指正），adb + root 可用。三条取证直接定型
方案：

| # | 取证 | 命令/来源 | 结论 → 决策 |
|---|---|---|---|
| 1 | **系统无任何 TP 固件文件** | `su -c 'ls /vendor/firmware /odm/firmware …'`（GPU 固件齐全，无 novatek）；input 设备表 `NVTCapacitiveTouchScreen` 活跃、`/proc/nvt_fw_version` = fw_ver 20 (0x14) | IC flash 常驻 fw 自启动，request_firmware 失败不影响功能 → **第一刷不带固件**（照抄实机行为，变量最少） |
| 2 | **触摸 irq 走 TLMM 汇总** | `/proc/interrupts`: `365: msmgpio 5 Edge NVT-ts-spi`（计数 219） | TLMM gpio5→GIC 汇总中断路径实证可用（主线同架构、首次真实消费者）；Edge 触发与驱动宏一致 |
| 3 | **lockdown 实测值** | `cat /proc/tp_lockdown_info` = `0x41,0x37,0x32,0x01,0x47,0x7b,0x31,0x00`；x_num=18/y_num=36 | tp-vendor 0x41 + glass 0x00 → 匹配 cfg_0（fw01）；`dsi_panel_lockdown_info_read` stub 直接填此值，**面板匹配不再依赖 DSI 读** |

> 取证 1 的结论在第一刷后被部分推翻（见 §6.3）——"实机无固件可工作"
> 对**实机自己的内核**成立，对自建内核的推论缺了一环：实机的 fw 来源
> 在内核之外不可见。这正是"照抄实机行为"与"照抄实机环境"的差别。

### 2.3 白名单核查

TLMM 的 72 脚白名单（迭代 20-fix15 建立）里 **gpio4/5/9（触摸 reset/
irq/cs）与 6/7/8（SE7 数据线）本就全部在位**（当时按 live 消费者清单
预置），无需改动——这是白名单机制"为后续 bring-up 预留"设计的首次
兑现。

---

## 3. 阶段二：固件来源

### 3.1 ROM 解包工具链重建（免 root，固化 tools/rom-extract/）

迭代 25 解包 ROM 的临时产物已被清理，重建并**持久化**（吸取 lld 放
/tmp 两次被清理的教训）：

```
apt-get download brotli libbrotli1 erofs-utils   → dpkg-deb -x → tools/rom-extract/tools/
brotli -d vendor.new.dat.br → vendor.new.dat（772MB）
sdat2img.py（自写）→ vendor.img（EROFS）
fsck.erofs --path=/firmware --extract → /firmware 目录
```

**sdat2img v4 格式踩坑**：transfer.list 的 `new <argc>,<start>,<end>` 行，
首位是**后续参数个数**（本包 argc=2 即单范围对 [start,end)），不是块数。
第一版按"块数"理解导致断言失败；修正后 185 行范围对精确覆盖
188523/188523 块。

### 3.2 结果与转向

`/firmware`（GPU 固件齐全）与 odm 分区**均无 novatek 固件**——YAAP
自定义 ROM 不带 TP 固件。转向**小米官方 oss 树的 .ihex 发布件**：
自写 `ihex2bin.py`（Intel HEX 记录类型 00/01/02/04 解析 + 校验和）转换
4 个文件，各 **139264 字节**，头字段合理（bin header 0x00 0x01 magic、
各 fw 头部区分位正常）。固化 `tools/touch-fw/`，第一刷暂不打包（§2
决策），第二刷启用。

---

## 4. 阶段三：驱动主线化移植

### 4.1 移植范围决策

| 组件 | 决定 | 理由 |
|---|---|---|
| nt36xxx.c / nt36xxx.h / nt36xxx_mem_map.h | 移植+主线化 | 核心链路 |
| nt36xxx_fw_update.c | 移植+主线化 | RAM fw 模型必需（后证实缺它 IC 进异常态） |
| nt36xxx_ext_proc.c | 移植+主线化 | 无外部依赖，/proc 调试接口（lockdown/fw 版本读取即用它） |
| nt36xxx_mp_ctrlram*.c | **不移植**（NVT_TOUCH_MP=0） | 工厂 MP 校准，3100 行纯产线工具；连带 `#define NVT_SAVE_TESTDATA_IN_FILE 1` 跳过 testdata 内存接口 |
| spi-xiaomi-tp.c/.h | 原样移植+微修 | 垫片，98 行 |

### 4.2 主线化修改全表（7.3 API 演进适配）

**GPIO 层（最大改动面）**：7.3 已删除 `of_get_named_gpio_flags`/
`gpio_request_one`/`gpio_set_value`/`gpio_to_irq` 等 legacy API——整体
**gpiod 化**：

```c
/* 下游（int 引脚号到处传） → 主线（描述符） */
ts->reset_gpio = of_get_named_gpio_flags(np, "novatek,reset-gpio", …);
gpio_request_one(ts->reset_gpio, GPIOF_OUT_INIT_LOW, …);
gpio_set_value(ts->reset_gpio, 1);
gpio_to_irq(ts->irq_gpio);
/* → */
ts->reset_gpio = devm_gpiod_get(dev, "reset", GPIOD_OUT_LOW);
gpiod_set_value_cansleep(ts->reset_gpio, 1);
gpiod_to_irq(ts->irq_gpio);
```

DT 属性名同步从 `novatek,reset-gpio`/`novatek,irq-gpio`（带 vendor
前缀，gpiod 的 con_id 匹配不上）改为主线惯例 `reset-gpios`/`irq-gpios`。
struct 字段 `int32_t irq_gpio` → `struct gpio_desc *`。

**接口与 API 演进**：

| 下游 | 7.3 适配 | 备注 |
|---|---|---|
| `struct file_operations` + proc_create | `struct proc_ops`（`.proc_open` 等带点成员名） | **debugfs 的两个 fops 保持 file_operations 不变**（一次性正则差点误伤，见 §4.3） |
| `strlcpy` | `strscpy` | |
| `del_timer_sync` | `timer_delete_sync` | |
| `do_gettimeofday` + `struct timeval` | `ktime_to_ms(ktime_get())` + `s64` ms（`ktime_get_ms()` 在 7.3 不存在，timekeeping.h 也没有——第一反应的函数名要再核对） | |
| platform_driver `.remove` int32_t | void | 7.3 签名变更（垫片同） |
| `client->master` / `SPI_MASTER_HALF_DUPLEX` | `client->controller` / `SPI_CONTROLLER_HALF_DUPLEX` | |
| 无点 designated initializer（`\t\tproc_open = …`） | 全部补前导点 | 新 clang 对 GNU 无点语法的报错位置在成员行，易误判为"成员不存在" |

**依赖裁剪与 stub（下游 techpack/平台依赖主线无对应物）**：

```c
static int dsi_panel_lockdown_info_read_stub(unsigned char *p)
{
        /* 实机取证值（§2.2）：tp-vendor 0x41 / glass 0x00 → cfg_0 */
        static const u8 lockdown[8] = {0x41,0x37,0x32,0x01,0x47,0x7b,0x31,0x00};
        memcpy(p, lockdown, 8); return 0;
}
static inline void dsi_panel_doubleclick_enable(bool on) { }  /* 手势双击 */
static inline void touch_irq_boost(void) { }                   /* irq boost */
static inline void lpm_disable_for_input(bool on) { }          /* LPM 投票 */
```

- **fb notifier 整体移除**：7.3 `fb.h` 已无 `FB_EVENT_BLANK` 事件枚举
  （注册/回调/unregister 全链删除）——bring-up 阶段触摸常开可接受，
  上游化时接 DRM client notifier；
- `CONFIG_TOUCHSCREEN_XIAOMI_TOUCHFEATURE` 段（touch mode 大表/palm/
  xiaomi_touch_interfaces）整体不编；其内的 `nvt_rf_resist_enable`/
  `nvt_rf_timer_timeout` 被 probe 无条件区 `INIT_WORK`/`timer_setup`
  引用 → 段外补空 stub；
- `MI_DRM_NOTIFIER`/`CONFIG_HAS_EARLYSUSPEND` 不定义（#else 的 fb 路径
  又被上一条删了，即挂起通知彻底不接）。

**probe 顺序容错（主线化增强）**：

```c
} else if (ret == -EINVAL) {      /* ts_xsfer not exist（垫片未 probe） */
        ret = -EPROBE_DEFER;      /* 下游：直接 exit nvt probe */
```

理由：ts_novatek 是 /soc 下独立 platform 设备，与 spi@984000/垫片之间
**无 fw_devlink 关联**（fw_devlink 只跟踪 OF 引用关系），late_initcall
probe 时若 SPI 侧尚未就绪，下游行为是一次性失败——主线 deferred probe
机制可以正确重试。

### 4.3 移植过程本身的坑（工程细节，供复刻参考）

1. **dts 注释里的 `*/`**：`/* qupv3_se7_spi_*/ts_* 实测参数 */` 中间
   的 `*/` 提前终止块注释 → dtc 语法错误。注释含通配符时改措辞；
2. **一次性正则误伤**：file_operations→proc_ops 的全局替换把
   `CONFIG_TOUCHSCREEN_NVT_DEBUG_FS` 段的 **debugfs** fops（要用
   file_operations）也改了；且 `\t.open\t=` 单 tab 形态没匹配全（双 tab
   行漏改）。修复：debugfs 两处还原 + 成员统一补点；教训：跨文件
   语义替换要按结构体边界，不能全文字符串；
3. **相对路径 cp 落错目录**：cwd 在 `drivers/input/touchscreen` 时
   `cp … ../misc/spi-xiaomi-tp.c` 落到 `drivers/input/misc/`——症状是
   Kbuild "没有规则可制作目标 …spi-xiaomi-tp.o"且与 auto.cfg 过期症状
   混淆，多花一轮排查；
4. **auto.cfg 过期**：改 Kconfig/Makefile 后必须 `make olddefconfig`
   （或 syncconfig）再编子目录目标，否则同样报"没有规则"——两个症状
   同型，先查文件位置再查 config 同步。

### 4.4 注册与 config

```
drivers/input/touchscreen/novatek-nt36672c/{Kconfig,Makefile}
  TOUCHSCREEN_NT36672C：depends on SPI && SPI_XIAOMI_TP
drivers/misc/Kconfig：SPI_XIAOMI_TP（照下游形态）
+obj 行各一（注意 Kconfig source 行只挂一次——脚本插两遍会 repeated
 inclusion error）
CONFIG_INPUT_TOUCHSCREEN=y  CONFIG_SPI_QCOM_GENI=y
CONFIG_SPI_XIAOMI_TP=y      CONFIG_TOUCHSCREEN_NT36672C=y
```

单目录编译验证命令（树根执行，路径不能从子目录传）：

```bash
make ARCH=arm64 LLVM=1 drivers/input/touchscreen/novatek-nt36672c/ drivers/misc/spi-xiaomi-tp.o
```

---

## 5. 阶段四：DT 与 config 落码

### 5.1 节点（自包含 dts，phandle 显式编号）

现有最大 phandle 0x62 → 新增 **0x63-0x6a** 八组（dtc 编译 + 反编译
双重验证 108 个 phandle 无重复）：

```dts
geniqup@9c0000 { status = "okay"; };        /* wrap1：仅 SE7 消费，其余 SE 节点保持 disabled */

spi@984000 {
        /* +pinctrl-0=<0x63>（qup11 on 6/7/8）/pinctrl-1=<0x64>（sleep） */
        /* -dmas/-dma-names：PIO 模式（见 §5.2） */
        status = "okay";
        touch_spi@0 {
                compatible = "xiaomi,spi-for-tp";
                reg = <0x00>;
                spi-max-frequency = <0x989680>;      /* 10MHz，live 实测 */
        };
};

ts_novatek {                                  /* /soc 下，照 live */
        compatible = "novatek,NVT-ts-spi";
        spi-max-frequency = <0x4c4b40>;       /* 5MHz */
        reset-gpios = <&tlmm 4 0>;            /* 主线惯例属性名（gpiod） */
        irq-gpios   = <&tlmm 5 0>;
        pinctrl-names = "pmx_ts_active", "pmx_ts_suspend";
        pinctrl-0 = <ts_int_active ts_reset_active ts_cs_active>;   /* 0x65 0x66 0x67 */
        pinctrl-1 = <ts_int_suspend ts_reset_suspend ts_cs_suspend>;/* 0x69 0x6a 0x68 */
        novatek,swrst-n8-addr = <0x3f0fe>;
        novatek,spi-rd-fast-addr = <0x3f310>;
        novatek,config-array-size = <0x03>;
        novatek,cfg_0/1/2 { … };              /* 三组面板固件选择表，照 live */
};
```

pinctrl 组 8 个的参数全部照 live 实测（§1.3 表）；CS 的 active/suspend
切换归 ts 驱动管（active 组含 gpio9→qup11），控制器自己的 default 组
只管 6/7/8——**CS 由驱动 probe 时应用 active 组后才切到 qup11**，早于
首次 SPI 传输（chip 识别），时序自洽。

### 5.2 关键决策：PIO 而非 GPI DMA

live dts 的 spi@984000 带 `dmas`（gpi_dma1，走 apps_smmu 0x36）。主线
7.3 的 spi-geni-qcom 两条路都支持，**首刷选 PIO（删 dmas 属性）**：

- 触摸报文小（事件 65B、fw 下载 139KB），FIFO 模式绰绰有余；
- 免去 gpi_dma1 驱动 + apps_smmu 启用 + interconnect 三个新变量
  （apps_smmu 当前 disabled、qnoc 全禁）；
- 实机验证 5MHz PIO 完美驱动 chip 识别、lockdown 读取与 131072B 固件
  下载。

下游走 GPI 是为了吞吐（触摸 + 大数据量产测），bring-up 阶段 PIO 是
更小的攻击面。后续接 MP 工厂测试或高负载时再评估回加。

---

## 6. 阶段五：第一刷

产物 `820b9ffd…`（同时叠加 GPU 清理版 v2 回归——判据独立可分）。

### 6.1 判读：probe 全链一次通过 ✅

```
[0.77] platform 9c0000.geniqup: Adding to iommu group 1     ← wrap1 启用
[0.82] SPI driver touch_xsfer has no spi_device_id …        ← 垫片（警告，后补）
[1.39] nvt_driver_init: start → nvt_ts_probe: start
       reset-gpios/irq-gpios acquired；SWRST_N8_ADDR=0x3F0FE；3 组 cfg 解析 ✓
       mode=0, max_speed_hz=5000000 → pinctrl_init → Request GPIO
[1.41] buf[1]=0x0C, buf[2..3]=0x00, buf[4]=0x72, buf[5]=0x66, buf[6]=0x03
[1.41] This is NVT touch IC                    ← ★ SPI 首读即中（NT36675 map 条目）
[1.41] input: NVTCapacitiveTouchScreen as input1
[1.41] request irq 113 succeed                 ← ★ TLMM gpio5 中断主线首次实证
[1.93] /proc/NVTSPI、nvt_fw_version、tp_lockdown_info… 全部创建；probe end
[6.49] Lockdown:0x41,0x37,0x32,0x01,0x47,0x7b,0x31,0x00    ← stub 实测值
[15.4] match panle type → fw01；（fw request -2 ×3 → 放弃）
```

无任何挂死/复位；GPU 清理版 v2 无回归。SPI/中断/输入三大链路一轮成型，
预判的四个风险点（SE 时钟 RCG、wrap1 首探、gpio5 中断、CS 时序）全部
未兑现——前序迭代（gcc 偏移审计、投票位/hwcg、白名单）的工程红利。

### 6.2 暴露问题一：`FW info is broken`

```
[15.44] firmware load failed, ret=-2 ×3 → nvt_update_firmware 失败
[15.45] FW info is broken! fw_ver=0x50, ~fw_ver=0xC5   ×4 retry
[15.45] Set default fw_ver=0, x_num=18, y_num=32 …     ← 用了 fallback 默认值
```

**排查**：

1. `~fw_ver` 校验失败（0x50+0xC5 ≠ 0xFF）= IC 的 fw info 区读到无意义
   数据，即 **IC 不在正常 fw 运行态**；
2. 读代码：`update_firmware_request` 失败路径只 `update_firmware_release()`
   + 释放 bin_map，**没有把 IC 复位回正常模式**——下载流程前段的
   bootloader reset 已把 IC 拉离正常态，下载未发生、恢复也未发生；
3. **决定性旁证**：fallback 默认 `y_num=32`，而实机 y_num=36——
   下游这段 fallback 代码从未在实机走对过，说明 **Android 上
   `nvt_get_fw_info` 从不 broken，即 Android 的 fw 请求从不失败**（其
   fw 来源在内核之外，我们看不到）。

**结论**：RAM fw 模型下 IC 正常态依赖 fw 下载成功；"不带固件照抄实机"
的判断错在把实机的**内核行为**（可以无文件）错当成实机的**环境**（它
其实有 fw 来源）。修复 = 固件入 initramfs（§7）。

### 6.3 暴露问题二：`Handlers=kbd`（无 event 节点）

`/proc/bus/input/devices` 里触摸设备 `H: Handlers=kbd`——没有 event
handler，`/dev/input/event*` 不存在（devtmpfs 正常挂着，USB shell 的
ttyGS0 即其节点；`.config` 里 **`CONFIG_INPUT_EVDEV` 未开**）。
`+CONFIG_INPUT_EVDEV=y`。

顺带修复：垫片补 `spi_device_id` 表，消 `SPI driver touch_xsfer has no
spi_device_id for xiaomi,spi-for-tp` 警告。

---

## 7. 阶段六：第二刷：修复与达成

### 7.1 修复内容

1. **固件 4 个入 initramfs**：固化 `scripts/initramfs/firmware/touch/`，
   `make-initramfs.sh` 打包到 `/lib/firmware/` **裸路径**（驱动的
   `request_firmware` 用无前缀文件名，与 GPU 固件的 qcom/ 前缀双路径
   不同）。按"父目录必须显式声明"的既有纪律验证：cpio -t 列出全部
   4 个文件、`fw01` 解包 cmp **逐字节一致**；
2. `+CONFIG_INPUT_EVDEV=y`；
3. spi_device_id 表。

产物 `a6292420…`（cpio 64 条目）。

### 7.2 实机结果：✅ 触摸打通（2026-10-13）

```
[15.44] match panle type, fw is [novatek_nt36672c_g7a_fw01.bin]
[15.44] fw_need_write_size = 131072(0x20000), NVT end flag
[15.44] nvt_bin_header_parser: ilm_dlm_num = 2 …          ← 固件解析正确、下载启动
```

**`/dev/input/event1` 出真实触摸数据**。UART 日志窗口截于下载中途
（用户抓取时点），但事件流是最硬判据——SPI（PIO）全速工作、irq 链
分发正常、input/evdev 层完整。

### 7.3 验证工具坑：busybox hexdump 的 EINVAL

实机 `hexdump -C /dev/input/event1` 报 `Invalid argument`——**不是
驱动问题**：`evdev_read()` 要求 `count >= sizeof(struct input_event)`
（24 字节），busybox hexdump 内部按 16 字节行缓冲、一次 `read()` 只
请求 16 字节被拒。绕过（dd 请求 24 的倍数，hexdump 从管道读不受限）：

```sh
dd if=/dev/input/event1 bs=96 count=5 2>/dev/null | hexdump -C
```

---

## 8. 最终形态：保留的代码修改

> **§8a 规范化重构（2026-10-13，功能打通后）**：垫片架构（§1.2①）
> 是下游 vendor 多 TP 共存的运行期所有权转交设计，单一 TP 的主线树
> 不应保留。重构为标准形态：
>
> | 项 | 垫片架构（旧） | **spi_driver 架构（现行）** |
> |---|---|---|
> | DT | ts_novatek 挂 /soc + touch_spi@0 垫片设备 | **touchscreen@0 直接挂 spi@984000 总线下**（compatible 改主线小写惯例 `novatek,nt36672c`，match 表保留下游旧串兼容） |
> | 驱动 | platform_driver + `late_initcall` + tmp_hold/get/put 所有权转交 | **`spi_driver` + `module_spi_driver()`**——probe 时机由设备模型管理（host probe 时枚举、host 未就绪走 deferred probe），垫片/所有权/手动重试全部删除 |
> | probe 签名 | `(struct platform_device *)`，内部再取 spi_device | `(struct spi_device *)` 直达 |
> | 文件 | + drivers/misc/spi-xiaomi-tp.c、include/linux/spi-xiaomi-tp.h、SPI_XIAOMI_TP Kconfig/Makefile | 全部移除（Image 内 touch_xsfer/spi-for-tp 字符串归零） |
>
> 副产品：probe 时机从 late_initcall 提前到设备枚举期；§4.2 的
> defer 化补丁随之自然消亡（设备模型本来就管）。产物 `c7a24f32…`。

| 文件 | 修改 | 性质 |
|---|---|---|
| `drivers/input/touchscreen/novatek-nt36672c/`（5 文件） | 下游 nt36672c 主线化（§4.2 全表）+ §8a 总线规范化；mp_ctrlram 未移植 | 新驱动 |
| `drivers/input/touchscreen/{Kconfig,Makefile}` | TOUCHSCREEN_NT36672C 注册（depends SPI && SPI_MASTER） | 注册 |
| `arch/arm64/boot/dts/qcom/sm7250-xiaomi-picasso.dts` | geniqup wrap1 okay；spi@984000 okay+PIO；8 组 ts/se7 pinctrl（0x63-0x6a）；touchscreen@0 挂 SPI 总线下（3 cfg，§8a） | DT |
| `scripts/make-initramfs.sh` + `scripts/initramfs/firmware/touch/` | 4 固件（.ihex 转换）→ /lib/firmware/ 裸路径 | 固件 |
| `.config` | INPUT_TOUCHSCREEN / SPI_QCOM_GENI / TOUCHSCREEN_NT36672C / **INPUT_EVDEV**（SPI_XIAOMI_TP 随 §8a 移除） | 配置 |
| `tools/rom-extract/` | sdat2img.py / ihex2bin.py + brotli/erofs 工具（持久化） | 工具 |
| `tools/touch-fw/` | 转换后固件源（139264B × 4，cmp 验证基准） | 数据 |

---

## 9. 验证方法

```sh
# ① 启动日志（NVT_DEBUG=1 → NVT_LOG 全 pr_err 常显）
dmesg | grep -E "NVT|984000"
#   成功链 = 984000 probe 0 → nvt_ts_probe start → "This is NVT touch IC"
#   → "request irq NN succeed" → probe end → 15s 后 fw01 下载（不再 -2）

# ② 输入设备与 handler
cat /proc/bus/input/devices | grep -A4 NVT
#   Handlers 必须含 event1（只有 kbd = EVDEV 没编）

# ③ 中断计数（触摸时增长）
cat /proc/interrupts | grep -i nvt

# ④ 事件流（busybox 坑见 §7.3）
dd if=/dev/input/event1 bs=96 count=5 2>/dev/null | hexdump -C
#   每条 24B：time(16) type(2) code(2) value(4)
#   type=0003 code=0035/0036 坐标 X/Y；0039 tracking-id（-1=抬起）
#   type=0000 EV_SYN 帧同步；type=0001 code=014a BTN_TOUCH

# ⑤ fw/面板状态
cat /proc/nvt_fw_version        # 应 x=18 y=36 fw_ver 合法
cat /proc/tp_lockdown_info      # 0x41,0x37,…,0x00
```

---

## 10. 方法论与教训

1. **实机取证是方案定型最快路径，但要分清"行为"与"环境"**：取证
   "无固件文件也能工作"正确，推论"自建内核不带固件也能工作"错误——
   实机的 fw 来源在内核之外（不可见）。同类推论须列出全部隐含前提。
2. **下游 fallback 代码是路径考古的证据**：fallback y_num=32 与实机
   36 不符 → 该路径从未被实机走对 → 实机从不进此分支 → 逆向推出
   "Android 的 fw 请求从不失败"，比抓 Android 日志更快。
3. **非标准拓扑的 probe 顺序要自行容错**：ts_novatek 与 SPI 设备无
   fw_devlink 关联（垫片是运行期转交，非 OF 引用），-EPROBE_DEFER
   化是主线化时应主动补的（下游 late_initcall 掩盖了此问题）。
4. **跨内核大版本移植的 API 演进面**：gpio→gpiod、file_operations→
   proc_ops（注意 debugfs 不变）、时间 API、remove 签名、spi controller
   改名——逐类清单化处理，别编译错一个修一个（本轮 13 个编译错误分
   五类一次清完）。
5. **语义替换要按结构体边界**：全局正则替换 file_operations→proc_ops
   误伤 debugfs fops；替换后 grep 全部同型符号复核。
6. **PIO 优先于 DMA 作为 bring-up 攻击面**：报文小的外设，FIFO 模式
   砍掉 DMA+IOMMU+ICC 三个依赖链，验证后再升级。
7. **busybox 工具的读取粒度会撞内核合法校验**：evdev 要求 read ≥
   24B，busybox hexdump 一次读 16B → EINVAL。工具报错先分辨是内核
   拒绝还是数据问题（EINVAL 来自 read 即工具侧）。
8. **既有工程纪律的复利**：TLMM 白名单（fix15 预置 4/5/9）、gcc 偏移
   全量审计（wrap1 时钟一次过）、phandle 显式编号+dtc 双验证——本轮
   零排障轮次不是运气。
9. **工具链/脚本产物持久化到 tools/**（/tmp 会被清理，同一坑不踩
   第三次）；解包脚本写对格式注记（sdat2img 的 argc 语义）防复用再踩。

---

## 11. 遗留与展望

| 项 | 说明 | 优先级 |
|---|---|---|
| fw 下载完成态定案 | UART 窗口截于下载中途；下次刷机 `cat /proc/nvt_fw_version` 确认 x=18/y=36/fw_ver 合法 | 低成本收尾 |
| 挂起通知 | fb notifier 7.3 无事件，灭屏触摸不挂起（bring-up 无碍）；上游化接 DRM client notifier | 中 |
| fw 更新失败路径 | 无 IC 复位（下游同款），带固件后不走；可考虑补 nvt_sw_reset_idle 恢复 | 低 |
| 手势唤醒 | WAKEUP_GESTURE 编译在（doubleclick stub 空实现）；需 suspend 链 + wakeirq（87 条 PDC 映射已归档）后启用 | 功耗阶段 |
| MP 工厂校准 | nt36xxx_mp_ctrlram 未移植；需要时按 ext_proc 同法适配 | 按需 |
| GPI DMA | 现 PIO；高吞吐场景（MP 测试/大数据）再评估 gpi_dma1+apps_smmu | 按需 |

---

*成文于 2026-10-13 触摸打通之日。逐轮实录与产物 sha256 见
`copydown-shim-port.md` 迭代 28；TLMM 白名单/wakeirq 背景见
`tlmm-wakeirq-map.md`。*
