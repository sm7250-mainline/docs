# picasso（SM7250/lito）显示链主线 bring-up 全记录

> 2026-10 上旬，迭代 18-25。从零移植到**屏幕点亮**（2026-10-11，
> `boot_7.3.img` 前身 6.17 版本 sha `0c645147…`）。本文记录完整技术路径：
> 每个组件首次如何编写（数据源与移植方法）、遇到的每一类问题、排查
> 手段与根因、以及被证伪的路线（simplefb）。配套迭代日志见
> `copydown-shim-port.md` 迭代 18-25（逐轮实录与 sha 表）；GPU 内部
> bring-up（GMU/gfx3d/zap）另见 `2026-10-12-gpu-bringup.md`，本文只覆盖
> 其与显示的交互点。

---

## 目录

1. [硬件与软件架构](#1-硬件与软件架构)
2. [阶段一：四大缺口的首次移植](#2-阶段一四大缺口的首次移植)
3. [阶段二：GCC 幽灵偏移大坑（59 个时钟审计）](#3-阶段二gcc-幽灵偏移大坑)
4. [阶段三：regulator 链（类型错配与命名陷阱）](#4-阶段三regulator-链)
5. [阶段四：SPMI/PDC 打通与 pinctrl 大战](#5-阶段四spmipdc-打通与-pinctrl-大战)
6. [阶段五：simplefb 弯路（被证伪的路线）](#6-阶段五simplefb-弯路)
7. [阶段六：GPU 依赖与首次 modeset 挂死](#7-阶段六gpu-依赖与首次-modeset-挂死)
8. [阶段七：冲刺（qfprom/nvmem/smmu/reparent/sleep_clk 终极根因）](#8-阶段七冲刺)
9. [最终形态：保留的代码修改](#9-最终形态保留的代码修改)
10. [验证方法](#10-验证方法)
11. [方法论与教训](#11-方法论与教训)
12. [遗留与展望](#12-遗留与展望)

---

## 1. 硬件与软件架构

### 1.1 显示链路总览

```
 kernel 侧                          硬件侧
 ─────────                          ─────────
 dispcc-sm7250 ──45 时钟+MDSS_GDSC─▶ DISP_CC @0xaf00000
    │  (byte/pixel clk_src)            ├ PLL0 (LUCID @0x0, 1035MHz)
 gcc-sm7250 ──disp axi/ahb/gpll0──▶    └ MDSS_CORE_GDSC (gdscr 0x3000)
    │
 msm DRM (display-subsystem@ae00000)
    ├ mdss (mdss_core GDSC 消费者)
    ├ DPU 6.1 (mdp@ae01000, GIC 83)  →  MDP 扫描引擎（SSPP→LM→DSPP→PP→INTF）
    │    catalog: dpu_6_1_sm7250.h       merge_3d×2 / DSC×2（G7A 双 slice 必经）
    ├ DSI0 ctrl (qcom,mdss-dsi-ctrl v2.4 @ae94000, mdss intr 4)
    │    └ sdm845_dsi_cfg 组（io 精确命中）
    ├ DSI PHY 7nm v4.1 (@ae94400)
    │    └ DSI PLL (@ae94900) → byte/pixel clk 反馈回 dispcc RCG parent
    └ panel G7A 37 02 0A（DSC video）
         │ 时序 hfp40/hs12/hbp40、vfp32/vs2/vbp30
         │ DSC 1.1：540×20 slice × 2、2 slice-pkt、8bpc/8bpp、block-pred
         │
 供电链（全部 PM8150L，SPMI SID 4/5）：
    vddio = l1c 1.896V (pldo_lv, 8mV 步进精确命中)
    avdd/avee = LCDB @pmic@5 ec00（+BST 联动 +150mV headroom）
    reset = pm8150l gpio3
    背光 = WLED @pmic@5 d800（3 strings + cabc）
    vdda/PHY = pm8150a-系 l5a 0.88V + refgen @0xff1000
```

### 1.2 与参考平台的差异（排障先验）

| 维度 | kona/sm8250（上游有） | **lito/sm7250（本文）** | 影响 |
|---|---|---|---|
| DPU | catalog 上游无 6.1 条目 | SDE_HW_VER_610，块偏移=live-0x1000 | 自建 catalog |
| DSI ctrl | v2.9 起显式 reparent | **v2.4**，靠 assigned-clock-parents（不可靠，见 §8） | 移植 v2.9 方案 |
| refgen | 0x88e7000（kona 地址） | **0xff1000**（live 实测） | 旧 DT 是错的 |
| 电源域 | MMCX 可用 | sm7150 域集（**无 MMCX**） | mdss/dsi/dispcc 域全改 CX |
| PMIC 资源前缀 | "l"（kona 惯例） | PM8150L 实为 **"c"**（ldoc1-11/smpc2,7,8/bobc1） | regs 组 probe |
| gcc 时钟偏移 | 多处与 lito 不同（disp axi/qup/sdcc/ufs/usb） | 以 gcc-lito.c 为唯一权威 | §3 大坑 |
| interconnect | qnoc 可用 | cmd-db 缺 SN11/12/14/15、SH8/SH10 BCM | qnoc 禁用（无 ICC 也能显示） |

### 1.3 数据源纪律（三源对齐）

- **下游 4.19 内核**（`Xiaomi_Kernel_OpenSource-picasso-r-oss/`）：行为权威——
  驱动源码（dispcc-lito.c、techpack/display、qpnp-lcdb-regulator.c）、
  dtsi（lito.dtsi、lito-gdsc.dtsi）；
- **live_dts**（Android 运行时导出的设备树，104 万字符）：实测真值——
  节点地址、pinctrl 状态、启用/禁用清单；
- **上游主线**（linux-6.17/7.3 原始包）：API 形态与参考平台（sm8250/sm6350）
  的结构模板。

每个数值落码前问一句"哪个源实锤"——本文反复出现的"幽灵偏移"（kona
地址混入）全是违反这条纪律的代价。

---

## 2. 阶段一：四大缺口的首次移植

起点（迭代 19 末）：UART shell 已通。对照 live dts 启用节点清单，主线
7.3 对 picasso 显示链的支持为**零**，四大缺口按下表移植（迭代 20）。

### 2.1 dispcc-sm7250.c（← 下游 drivers/clk/qcom/dispcc-lito.c）

**移植方法**：结构照上游 `dispcc-sm8250.c`（同代），数据（45 个时钟的
寄存器偏移/freq 表/parent 关系）原样取自 `dispcc-lito.c`。API 主线化要点：

- 下游 `vdd_class`（techpack 时钟投票）→ 主线 GDSC 框架：
  MDSS GDSC 折入 `gdscr = 0x3000` + `HW_CTRL` 旗标（hw-trigger 域，
  live dts `qcom,gdsc-hw-ctrl` 实证）；
- PLL0：LUCID @0x0，L=0x35（1035MHz），`clk_alpha_pll_fixed_lucid_ops`；
- parent_data 用 **`.index` 引用**而非 fw_name——这是后来 DSI PLL
  reparent 排障的伏笔（§8.4：fw_name 版本在 6.17 实测解析错位，.index
  才暴露出真正的 dts 根因）；
- MDSS_CORE_BCR（block reset）一并注册。

**注册**：Kconfig `SM_DISPCC_7250` + Makefile + dt-binding 头
（`qcom,dispcc-sm7250.h`，45 时钟 id）。

### 2.2 DPU 6.1 catalog（← 下游 techpack/display）

lito 是 **SDE_HW_VER_610**（下游 sde_hw_catalog.c 实锤）。新建
`drivers/gpu/drm/msm/disp/dpu1/catalog/dpu_6_1_sm7250.h`：

- 拓扑：4 ctl、2 vig + 3 dma SSPP、4 lm、2 dspp、4 pp（merge_3d@0x83000/
  0x83100）、2 dsc、4 intf、wb、cdm；
- **偏移全部 = live dts 的 `qcom,sde-*-off` − 0x1000**（live 的 mdp 基址
  0xae00000 是 MDSS 父级，主线 mdp@ae01000 映射，故全部减 0x1000）；
  逐块核对表见迭代 25 的审计矩阵；
- `dpu_hw_catalog.c` include + extern、`msm_mdss.c` 加 `qcom,sm7250-mdss`
  匹配条目。

**代价最大的教训**：merge_3d 原写 0x3000/0x3100（老 catalog 残留，非
live 实测），直到迭代 25 全量审计才修正为 0x83000/0x83100——G7A 是
DSC 双 slice 面板（1080 = 2×540），DSI 单 port 必走 PP0+PP1→merge_3d，
基址错 = modeset 写未映射区域，**疑为历史 modeset 静默挂死的直接元凶**
（与 GPU 缺失叠加，见 §7）。

### 2.3 LCDB 调节器（← 下游 qpnp-lcdb-regulator.c，2323 行裁剪）

PM8150L LCD bias（avdd +5.5V/avee −5.5V）。裁剪版 `qcom-lcdb-regulator.c`
只保留三块功能：

- 模块使能：0x46 bit7 + 0x10 轮询 ready；
- ldo/ncp 电压设定：0x71/0x81；
- BST 联动：0x41（输出 = 设定 + 150mV headroom）。

裁掉：PM660L workaround、TTW、休眠控制、SC IRQ（均 lito 不用或非必需）。
**探测接口踩坑**：SPMI 孙节点在 `#size-cells=<0>` 下 platform resource
机制不可靠（同结构 WLED 能取到 reg 而 LCDB 取不到，机制之谜未解），
最终用 `of_property_read_u32(np, "reg")` DT 直读绕过。

### 2.4 G7A 面板驱动（树内生成驱动补全）

主线树内的 `panel-novatek-nt36672c.c` 实为小米 G7A 生成驱动（文件名
误导，CONFIG_DRM_PANEL_NOVATEK_NT36672C）。补两块：

- regulator_bulk（vddio/avdd/avee）+ 上电时序：vddio →1ms→ avdd/avee
  →10ms → reset（照下游 `dsi_panel_pwr_supply_avdd` 序列）；
- mode/DSC 参数与 live 逐项核对（§1.1 表）全部一致。

### 2.5 DT 与 config 的首轮落码

DT（当时为 runtime dts，迭代 24 起统一树内 dts，内容一致）：

| 节点 | 关键修正 |
|---|---|
| refgen | 0x88e7000 → **0xff1000**（live 实测；旧值 kona 地址） |
| mdss_mdp/dsi/dispcc 域 | MMCX → **CX**（sm7150 域集无 MMCX） |
| WLED/LCDB | 从 spmi 总线平级移入 **pmic@5**（下游实测 USID 5；挂错 USID 写入无响应） |
| 面板 vddio | l1a(0.752V 错) → **l1c@1.896V**（见 §4） |
| dispcc 输入 | sm8250 式 + gcc 补 `gcc_disp_gpll0_clk_src`（AON 0x52000 bit20，ID 160） |
| DSI PHY compatible | 改通用（`qcom,dsi-phy-7nm` 走版本表 v4.1） |

config 新发现（迭代 20 首刷前的重大盲区）：**`CONFIG_SPMI` 整链未开**——
PM8150L 子树（gpio/wled/lcdb）此前从未 probe 过。补：

```
CONFIG_SPMI=y  CONFIG_SPMI_MSM_PMIC_ARB=y  CONFIG_MFD_SPMI_PMIC=y
CONFIG_PINCTRL_QCOM_SPMI_PMIC=y  CONFIG_SM_DISPCC_7250=y
CONFIG_REGULATOR_QCOM_LCDB=y  CONFIG_DRM_MSM=y
CONFIG_DRM_PANEL_NOVATEK_NT36672C=y  CONFIG_BACKLIGHT_QCOM_WLED=y
```

---

## 3. 阶段二：GCC 幽灵偏移大坑

### 3.1 现象（迭代 20 首刷）

LCDB/WLED/dispcc/mdss_gdsc 全部 probe 成功，但 `mdss_runtime_resume`
**无限 -EBUSY 重试 21 次**：

```
gcc_disp_hf_axi_clk status stuck at 'off'
Failed to enable clk 'bus': -16
```

→ mdss probe 永远失败，DSI/panel 全被拖死。现象：屏幕一闪（GDSC 上电
瞬间）后黑屏、背光常亮。

### 3.2 排查与根因

我们的 gcc-sm7250.c 是自 kona 版（gcc-sm8250.c）改的。对照下游
`gcc-lito.c` 同名时钟逐一比对——`gcc_disp_hf/sf_axi_clk` 用了 kona 偏移
**0xb020/0xb024，lito 实际 0xb030/0xb034**：写入无效地址，halt 位永不变
on。

**升级为全量审计**（用户指示）：72 个同名时钟逐一对 gcc-lito.c 比对，
**59 个偏移不符**。全部按 lito 修正。受影响面远超显示：

| 时钟族 | 后果 |
|---|---|
| 全部 QUP wrap0/1 SE RCG | **解释了迭代 17 的历史谜题** `rcg didn't update its configuration`——UART RCG cmd_rcgr 0x17290→0x17270 一直指错（波特率靠 ABL 残留配置才正常） |
| SDCC1/2 | 整套在错误区域 0x750xx/0x130xx |
| UFS 全家、USB 全家 | 后续 bring-up 直接受益 |
| prng/video/gpu/npu 分支 | 同批修正 |

### 3.3 第二轮审计（投票位与 hwcg）

偏移对了还不够——二刷暴露 `gcc_qupv3_wrap_0_s_ahb_clk status stuck at
'off'` → -16。根因两层：

1. **AON 投票位错**：wrap0 m/s_ahb 的投票位写了 kona 的 bit2/bit3，
   lito 实为 **bit6/bit7**（写错位 = 给别的时钟投票，目标时钟无人理）；
2. **缺 hwcg 字段**：主线 clk-branch 在 HW gating 模式下靠
   hwcg_reg/hwcg_bit 跳过 halt 轮询，不配则永远等不到 on。

第二轮全面审计（halt_check/enable_mask/hwcg）修正 **36 处**（19 处
hwcg + HALT_SKIP/VOTED/DELAY 类型）。至此 GCC 显示路径时钟全部就绪。

### 3.4 方法沉淀

> **自上游同代平台改的驱动，每个寄存器偏移都必须对下游真值全量审计，
> 不能只对显示用到的几个**——这是本项目"gpll7 幽灵 PLL"（迭代 17，
> 读 0x27000 无响应区挂死整机）之后的第二次全量审计，此后固化为纪律。
> 下一例见 §8 的 merge_3d（catalog 侧同病）。

---

## 4. 阶段三：regulator 链

### 4.1 现象（迭代 20 二刷）

DPU probe 成功（`ae01000.display-controller returned 0`，历史首次），
但 rpmh-regulator 两组注册失败 -131：

```
vreg_l2c_3p0 / vreg_l1a_1p896: failed to get the current voltage: -ENOTRECOVERABLE
→ regs-0/1 整组失败 → l9a/l5a/l1a 全缺失 → DSI/PHY/panel 全 -517
```

### 4.2 根因 A：PMIC 类型错配

主线 rpmh-regulator 的类型表：**ldoa1 = pmic5_nldo（上限 1.328V）装不下
1.896V；ldoc2 = nldo 装不下 3.008V**。machine_constraints 的 apply_uV
路径 set 后 get 映射不中线性范围 → -ENOTRECOVERABLE → 整组拒绝注册。

### 4.3 根因 B：命名陷阱（本项目最隐蔽的一类）

live_dts 里 1.896V 的轨名字叫 **"pm8150a_l1"**，但它的父块其实是
`rpmh-regulator-ldoc1`——小米 R 树命名误导（4.19 picasso overlay 实际
改的正是 `&L1C`，双重佐证）。**面板 vddio 真身 = PM8150L 的 ldo1**：
主线 pldo_lv 类型（1.504-2.0V @8mV 步进），1896000µV 精确命中。

修复：vddio 改挂 l1c@1.896V；l1a 复原 0.752V；l2c 改 1.144V（nldo
范围内，本机无消费者）。

### 4.4 十六进制乌龙 ×2（三刷、四刷）

三刷 l2c 仍 -131：手写 `0x1179D0` = 1,145,296 ≠ 1,144,000（十六进制
换算错）。四刷改 `0x117730` = 1,144,624 **还是错**。连续两次人肉换算
翻车后，**全部手写数值改用 python 换算并全面复核**：

```
l2c → 0x1174C0 = 1,144,000 ✓
l1c → 0x1cee40 = 1,896,000 ✓（pldo_lv 8mV 步进点）
```

五刷：regs-0/1 全通。此纪律后来扩展为"立即数地址一律工具换算 +
objdump 核对"（shim 停狗地址、a6xx chipid 同病，见 GPU 文档）。

---

## 5. 阶段四：SPMI/PDC 打通与 pinctrl 大战

> 六刷到十七刷，共 12 轮。显示链最后的硬骨头不在显示本身，而在它的
> 供电链（SPMI）与引脚子系统（TLMM pinctrl）。

### 5.1 PDC：一根 CONFIG 引发的连环案

五刷后 dsi/spmi/pinctrl/tsens 仍在 defer。终极根因：
**`CONFIG_QCOM_PDC is not set`**——pdc 驱动根本没编进内核。PDC 是
wakeup 中断域（spmi 的 periph_irq、pinctrl 的 wakeup-parent、tsens 的
中断都挂它下面），域不存在 → `of_irq_get()` 的 `irq_find_host()` 返回
NULL → -EPROBE_DEFER 永久循环。**这也是历史遗留"pinctrl 永久 -517"
的真因另一半**（前一半是 gpio-ranges 自引用，迭代 19 已修）。

`+CONFIG_QCOM_PDC=y` 后六刷：**spmi 全链通**（pmic@4/@5、gpio@c000
全部 probe 0），但随即暴露三个新问题（pinctrl 挂死、LCDB/WLED reg
错——§2.3/§2.5 已述后者）。

### 5.2 pinctrl 挂死：从假说证伪到行为差异定位

**九刷**（打点二分）：`regs mapped ✓ → registering pinctrl ✓ → calling
msm_gpio_init → 死`。首轮假说"ABL 遗留 GPIO pending 在汇总中断使能
瞬间引发风暴"→ 主动清全部 146 脚中断 → **十刷在 gpio0 写挂**
（intr_cfg 写无响应）→ 十一刷每脚打点实锤 gpio0 槽位死区。

**关键法证**（live_dts）：onewire 节点 `cfg-addr=0xf942000` ==
EAST(0x900000) + 0x1000×66——**tile 布局公式完全正确**，死的是 gpio0-3
个别槽位（0/3/4/5 为 PDC direct-connect 唤醒脚，疑 TZ/PDC 接管）。
Android 从不访问故无感。**用户指示定调：对齐下游惰性——probe 不碰
任何引脚寄存器，停止纠结该区为何不可写。**

**tlmm-verify 工作流**（3 agent 并行取证，结论归档
`docs/tlmm-wakeirq-map.md`）：

- 146 脚 PINGROUP 的 index==id 全量核对 **0 错位**；
- 抓到 2 个真实偏移错：UFS_RESET 0x9f000→**WEST+0xbc000**、SDC2
  0x98000→**SOUTH+0xb7000**（下游绝对地址相减实锤）；
- 提取 87 条 gpio→PDC wakeirq 官方映射归档备用；
- 参照项目 other_works 实机点亮但**完全没有 TLMM**（引脚全靠 ABL
  预配置）——反向印证"不碰无主引脚"路线。

**十二刷突变**：wakeup-parent 删除后 pinctrl 第一次 probe 即真跑、当场
挂死（此前各轮先 -517 defer）。真凶浮出：**6.17 gpiolib 的
`gpiochip_add_data()` 对每个 valid 引脚调用 `gc->get_direction()`（读
ctl_reg）初始化 FLAG_IS_OUT——4.19 没有这个逐脚循环**。gpio0 第一脚
即读死。

### 5.3 修复演进：白名单 → no_get_direction

| 轮次 | 修复 | 判决 |
|---|---|---|
| 十三刷 | 掩码挂在 `girq->init_valid_mask`（irq 层） | ✗ 无效——gpiolib 查的是 `gpiodev->valid_mask`（gpio 层），**两套独立开关**，教训：修复必须核到目标代码行的完整调用链 |
| 十四刷 | `chip->init_valid_mask`（gpio 层，三环链逐环核对） | △ 掩码生效、gpio6-58 全读通，**gpio59 又挂**（WEST 槽位） |
| 十五刷 | 下游审计得 73 脚白名单（live 启用消费者引用） | ✗ gpio0 在白名单内被读挂——"启用的数据属性消费者 ⇒ 槽位可读"被证伪（gpio0 是 PDC direct-connect 脚，疑 TZ 接管） |
| 十六刷 | **`soc_data.no_get_direction`**：置 NULL → 6.17 注册循环走软件分支 → **probe 零引脚寄存器访问**（与 4.19 注册行为完全一致） | ✅ **彻底打通** |

配套修复：pinctrl 函数表整体重生（python 从 pinctrl-lito.c 提取全部
146 脚 funcs + 函数组；原表是机械拼的——qup04 错含 gpio37、mdp_vsync
缺 12/70/71 等 25+ 处不一致，曾致 uart 状态应用 WARN、serial probe -22）。

**十六刷战果**：pinctrl probe 0 → dsi/panel 解除 defer → **DPU hw
revision 0x60010000、dsi bound、Initialized msm 1.13.0——显示链全部
probe 成功**。转战 modeset（§6-7）。

---

## 6. 阶段五：simplefb 弯路（被证伪的路线）

### 6.1 动机

十六刷后首次 modeset 静默挂死（§7），临时点不亮。决策：屏蔽 msm
（display-subsystem disabled），改走 simplefb 寄生 ABL 留下的
cont-splash 管道（`framebuffer@a0000000` + CONFIG_FB_SIMPLE）。

### 6.2 实测过程

- **十九刷**：引导全通、simplefb 注册、fbcon 接管 135×150——**屏幕黑**。
  根因：mdss 屏蔽后显示资源无消费者，late-init 全关（日志实锤
  `PM: genpd: Disabling unused power domains` + `regulator_init_complete`）。
- **fix19**：CMDLINE 追加 `pd_ignore_unused regulator_ignore_unused`（与
  既有 `clk_ignore_unused` 同族）。
- **二十刷**：参数生效（`Not disabling unused power domains`）**仍黑屏**。

### 6.3 机理性结论（为什么救不回）

simplefb 的前提是 **ABL 配置的显示管道在内核期继续跑**（DPU 扫描固定
帧缓冲 → DSI 出流 → DSC 解码 → panel）。但：

1. fastboot boot 场景下 cont-splash 管道在 ABL `ExitBootServices` 前后
   即停止（UEFI 把显示资源交还）——**停止的扫描与视频流，保活参数
   只能留时钟/电源域，救不回来**；
2. G7A 是 **DSC 视频模式面板**：流断即黑（不像 command mode 面板有
   self-refresh 窗口）；"企鹅一闪"只是 simpledrm 挂帧缓冲的残余窗口。

**结论：simplefb/cont-splash 寄生路线在本场景不可依赖，主线 msm 自建
链是唯一正道。** 此弯路的价值：确证了 late-init 电源治理行为，且
keep-alive 三参数（clk/pd/regulator_ignore_unused）从此常驻 cmdline
（对其他子系统同样有益）。

---

## 7. 阶段六：GPU 依赖与首次 modeset 挂死

### 7.1 现象与定位

十六刷显示链 probe 全通后，首次 modeset（fbdev 首开触发）静默挂死：
日志止于 `adreno_load_gpu` 的 **"no GPU device was found"**，之后
panel prepare/DSI 上电/PHY 全无打印。为定位在 modeset 链加了 20+ 个
DBRG 打点（dpu_encoder atomic_enable / dsi pre_enable / dsi power_on 六段
/ phy enable / panel prepare 各阶段）——**十八刷一个都没触发**，证明
挂点在打点覆盖之前的 msm fbdev/modeset 内部（dpu_crtc enable 区）。

### 7.2 GPU 为什么挡显示的路

主线 msm DRM 的 fbdev 路径要求 GPU 在位（adreno_load_gpu 成功）才走
完整 modeset；无 GPU 时 fbdev 首开的路径在本机直接挂死（叠加
merge_3d 幽灵基址——写未映射区，两因素叠加，§2.2）。因此显示的最终
点亮以 GPU 链就绪为前置：迭代 20-fix20 起 GPU 整体使能（gpucc 移植/
a6xx catalog/gcc gpu_gpll0/DT 四节点），迭代 23 实机暴露 catalog
chipid 不匹配（`Unknown GPU revision: 6.2.0.2`），mdss 一度再暂禁给
USB console 让路（迭代 23），直到迭代 25 GPU 链修复完成（细节见
`2026-10-12-gpu-bringup.md` 阶段 1-3）后显示恢复。

---

## 8. 阶段七：冲刺

> GPU 链就绪、mdss 恢复 okay（迭代 25）后的六轮，从"GPU 被 fw_devlink
> 卡死"到"屏幕点亮"。

### 8.1 第一刷：fw_devlink 陷阱（qfprom）

DPU/DSI/panel **历史首次全部 probe 0**（`ae94000.dsi returned 0`），但
GPU 反复 `returned -517 after 8 usecs`——**8µs、零驱动打印**是指纹。

根因：gpu 节点的 `nvmem-cells = <qfprom gpu_speed_bin>` 使 fw_devlink
把 qfprom 登记为 gpu 的 **supplier**；`CONFIG_NVMEM_QCOM_QFPROM` 缺编 →
supplier 永不 ready → gpu 的 really_probe **在驱动调用之前**被打回
-517 → gpu 不 component_add → msm DRM master 永不 bind。

修复：`+CONFIG_NVMEM_QCOM_QFPROM=y`（顺带发现 config 静默回归两例：
QCOM_SMEM/HW_RANDOM_QCOM 被丢——config 大改后抽查的纪律由此固化）。

### 8.2 第二刷：nvmem cell 约束

qfprom 编入后 probe **-22**：`nvmem: invalid bits on gpu-speed-bin@1ea`。
cell 写 `reg=<0x1ea 0x01>` 而 `bits=<5,8>` **跨字节需要 2 字节**（nvmem
core 校验 bit 偏移+宽度 ≤ reg 字节数）。live 实测 `reg=<0x1ea 0x02>`；
kona 是完全不同的 `@19b<5,3>` 单字节布局——**不可跨平台照抄 fuse cell**。

### 8.3 第三刷：GPU 链贯通到 panel init + 两个新卡点

qfprom ✓ → gpu bind ✓ → `dpu hardware revision 0x60010000` →
**`Initialized msm 1.13.0`** → fbdev 1080×2400 stolen fb → DSI→PHY→host
power_on→panel 供电/复位全通（DBRG 全触发）。用户现象："企鹅一闪 →
背光关 → 背光开 → 黑屏" = simpledrm 企鹅被 msm 顶掉 + panel off/on
循环（modeset 在跑）。剩两卡点：

1. `disp_cc_mdss_byte0_clk status stuck at 'off'` → link clk -16 →
   host power_on 失败 → panel DCS -22；
2. `a650_sqe.fw` 加载 -2（GPU 微码缺失——此悬案两个月后在迭代 26 破案：
   initramfs spec 缺 `dir /lib/firmware` 父目录条目，内核解包静默丢弃。
   与显示无直接关系，但记录在此因为它当时消耗了大量排查带宽）。

### 8.4 byte0 卡点：显式 reparent → 两级 debug → 终极根因

**修复尝试一（fix4）**：DSI 显式 reparent。上游 v2.9 平台的方案下沉到
v2.4——`dsi_clk_init_6g_v2()` 尾部可选 get 四个 clk（byte_src/
dsi_pll_byte/pixel_src/dsi_pll_pixel，devm_clk_get_optional，老平台行为
不变）；`dsi_link_clk_set_rate_6g()`（power_on 阶段，PHY 已 enable）显式
reparent byte/pixel src → DSI PLL。源码注释给了机理：

> "DSI PHY PLLs have to be enabled to allow reparenting to them, so
> cannot use assigned-clock-parents"

（v2.4 靠 assigned-clock-parents 在 probe 时一次性 of_clk_set_defaults，
失败后无人重试 → byte0/pclk0_clk_src 悬在死 parent）。**实机：仍 -22。**

**两级 debug 定位法**（此案的方法论核心）：

- 第一级（consumer 侧 dump，dsi_host.c reparent 失败时）：
  `byte0_clk_src parents = {bi_tcxo, gcc_disp_gpll0_clk_src, (null)}`——
  **DSI PLL 的位置被 gpll0 占据**，`clk_fetch_parent_index` 找不到 → -22；
- 第二级（框架侧 dump，drivers/clk/clk.c `clk_core_fill_parent_index`
  条件打印）：
  `fw=dsi0_phy_pll_out_byteclk idx=0 -> gcc_disp_gpll0_clk_src err=0`——
  **fw_name 字符串正确却解析到 clocks[2]**。

而 dtb 三重验证（本地 sha=manifest=boot.img 内嵌、fdtget 原始属性）
clock-names/clocks 全部正确 5 项——**数据与代码都对，运行结果错位**。

**修复尝试二（fix5）**：parent_data 改 `.index`（按 clocks 数组位置直引）。
实机数据点：`idx=3 -> gpll0(clocks[2])`、`idx=5 -> dsiclk(clocks[4])`、
`idx=0 -> bi_tcxo`——**运行时对 index≥1 系统性 -1 偏移**。fw_name 时代
全部数据点同规律吻合 → 锁定**条目边界问题**而非字符串匹配问题。

**终极根因（fix6）**：dispcc 节点的 **sleep_clk 引用写 `<0x29 0x00>`，
而 0x29 是 `#clock-cells = <0>` 的 fixed-clock**——多余的 arg u32 在
运行时被 of_phandle_iterator 当作下一个条目的 phandle（无效 phandle 0，
占据一个条目位），此后全部条目索引 +1：

```
[0]bi_tcxo [1]sleep_clk [2]无效-0 [3]gpll0 [4]dsi0byte [5]dsi0dsi
                                ↑ idx=3→gpll0、idx=5→dsiclk 与实测逐点吻合
```

修复：`<0x29 0x00>` → `<0x29>`（全树唯一一处）。条目表恢复 5 项后
`.index` 值随即可用。

**实机（0c645147）：屏幕点亮（2026-10-11）。**

> **工具盲区警告**：dtc 不校验 arg 数与 #clock-cells 匹配、fdtget 打印
> 原始 u32 也不暴露——这类错误只有内核侧两级 debug 对出规律后才能锁死。
> 审计手写 dts 必须逐一核对每个 provider 的 #clock-cells 与引用 arg 数。

---

## 9. 最终形态：保留的代码修改

| 文件 | 修改 | 性质 |
|---|---|---|
| `drivers/clk/qcom/dispcc-sm7250.c` | 自下游 dispcc-lito.c 移植（45 时钟 + MDSS GDSC + PLL0 LUCID）；byte/pixel parent 用 `.index` 引用 | 新驱动 |
| `drivers/clk/qcom/gcc-sm7250.c` | 两轮审计修正 59 偏移 + 36 投票位/hwcg；补 gcc_disp_gpll0_clk_src | 修正+补全 |
| `drivers/gpu/drm/msm/disp/dpu1/catalog/dpu_6_1_sm7250.h` | DPU 6.1 catalog（偏移=live-0x1000；merge_3d **0x83000/0x83100**） | 新增 |
| `drivers/gpu/drm/msm/msm_mdss.c` + `dpu_hw_catalog.c` | sm7250-mdss 匹配条目 / catalog 注册 | 注册 |
| `drivers/gpu/drm/panel/panel-novatek-nt36672c.c` | G7A 驱动补 regulator_bulk + 上电时序 | 补全 |
| `drivers/regulator/qcom-lcdb-regulator.c` | 下游 2323 行裁剪版（DT 直读 reg） | 新驱动 |
| `drivers/gpu/drm/msm/dsi/dsi_host.c` | **v2.9 显式 reparent 方案下沉 v2.4**（可选 clk 四件套 + power_on 时 reparent） | 功能回移 |
| `drivers/pinctrl/qcom/pinctrl-sm7250.c` | 4-tile 偏移并入算式（=下游 4.19 同一访问式）+ 函数表从 pinctrl-lito.c 整体重生 + **no_get_direction** + 72 脚白名单 | 新驱动+机制 |
| `drivers/pinctrl/qcom/pinctrl-msm.c/.h` | `soc->init_valid_mask`/`no_get_direction` 字段与挂载（gpio 层掩码，非 girq 层） | 机制 |
| `drivers/soc/qcom/ubwc_config.c` | `qcom,sm7250` 条目（ubwc 3.0/hbb15/swizzle6——无条目 DPU probe -EINVAL） | 注册 |
| `arch/arm64/boot/dts/qcom/sm7250-xiaomi-picasso.dts` | 显示全树节点 + 修正（refgen/CX 域/pmic@5 子树/l1c/qfprom cell/sleep_clk 0-cell） | DT |
| `.config` | SPMI 链四件套/QCOM_PDC/DISPCC/LCDB/DRM_MSM/PANEL/WLED/REFGEN/QFPROM | 配置 |

---

## 10. 验证方法

```sh
# ① 启动日志组合判据（必要条件）
dmesg | grep -E "dispcc|mdss|dsi|dpu hardware|Initialized msm|fbcon|frame buffer"
#   成功链 = dispcc probe 0 → mdss gdsc → "dpu hardware revision 0x60010000"
#   → "Initialized msm 1.13.0" → "Console: switching to colour frame buffer"
#   → "fb0: msmdrmfb frame buffer device"

# ② 分层探针（失败时按链路二分）
dmesg | grep -E "returned (-517|-131|-22|-16)"   # probe 错误码分层：
#   -517=supplier 未就绪（查 fw_devlink）  -131=regulator 类型/电压超界
#   -22=参数/资源（reg 格式、cell bits、reparent）  -16=时钟 stuck（偏移/投票位）

# ③ 供电链实测（SPMI 侧）
cat /sys/class/regulator/*/name | grep -E "l1c|lcdb"    # vddio/偏压在位

# ④ 背光/面板状态
cat /sys/class/backlight/*/brightness
```

屏幕出图 ≠ 渲染管线通（fbcon 走 DPU 扫描、CPU 写帧缓冲；GPU 死着也亮）。
GPU 侧判据另见 GPU 文档 §8。

---

## 11. 方法论与教训

1. **三源纪律**：下游内核=行为权威、live_dts=实测真值、上游=结构模板。
   每个数值落码前问"哪个源实锤"。本文四大类翻车（gcc 59 偏移、refgen/
   PMIC 前缀、merge_3d、fuse cell）全是"照抄了错误平台的值"。
2. **自同代平台改的驱动必须全量审计寄存器偏移**——只审显示用到的
   时钟时，qup/usb/ufs 的幽灵留给了未来（迭代 17/20 各付一次学费）。
3. **两级 debug 对比法**：consumer 侧 dump（parent 表）+ 框架侧 dump
   （fill_parent_index）一次对比锁定错位层；fw_name 与 .index **一致错位**
   → 条目边界问题；**错位规律一致**（系统性 -1）→ 数据问题而非代码问题。
4. **手写 dts 的 0-cell provider 引用绝不能带 arg**（挤位效应），dtc/
   fdtget 都不查——只能靠 #clock-cells 逐一核对或内核侧 debug。
5. **fw_devlink 指纹**：`returned -517 after ~8 usecs` + 零驱动打印 =
   supplier 缺编（驱动 probe 前被打回），逐个 grep DT 引用的 supplier。
6. **fuse cell/nvmem 布局每平台不同**，bits 跨字节时 reg 长度必须覆盖。
7. **静默失败路径优先怀疑**：initramfs 父目录缺失（sqe -2 两个月）、
   SPMI 孙节点 resource 机制、TZ 拒绝只回 -22。
8. **掩码类修复必须核到目标代码行完整调用链**（girq 层 vs gpio 层两套
   独立开关的教训）；**行为差异类 bug 先读上游演进**（6.17 gpiolib 逐脚
   get_direction 是 4.19 没有的新行为——"下游能跑"不等于"主线同路径能跑"）。
9. **不可访问区不纠结**：gpio0-3 死槽按下游惰性路线绕行（no_get_direction
   对齐 4.19 注册行为），取证工作流只用于排除"是不是我们把地址算错了"。
10. **立即数/十六进制一律工具换算**（l2c 两次翻车后固化；同一坑后来在
    shim 停狗地址与 a6xx chipid 三度出现）。
11. **寄生路线（simplefb/cont-splash）在 fastboot boot 场景不可依赖**：
    UEFI 收回显示资源后流已断，保活参数救不回 DSC 视频面板。
12. **config 静默回归要抽查**：QCOM_SMEM/HW_RANDOM_QCOM/QCOM_PDC 都被
    olddefconfig 静默丢过；大迭代后逐项 grep。

---

## 12. 遗留与展望

| 项 | 说明 | 优先级 |
|---|---|---|
| DFPS | 面板 120/90/60Hz 动态刷新率未启用（mode 表只有 60Hz 档） | 中 |
| ESD irq | gpio55（面板 ESD 保护中断）未接 | 低（保护功能） |
| TE 同步 | tlmm10 TE 走 DSI 中断 vs TE gpio 的取舍 | 低（vsync 已可用） |
| interconnect | qnoc 禁用中（缺 BCM 会冲 RPMh 复位）；显示无 ICC 可跑但无带宽投票 | UFS/USB 阶段一起修 |
| sde_rscc | 上游 msm 无 display RSC 驱动（仅功耗管理） | 可接受 |
| 触摸 | 迭代 28 打通（`2026-10-13`），本文范围外 | 已完成 |

---

*成文于 2026-10-13（触摸打通之日，应约回顾整理）。逐轮排障实录与全部
产物 sha256 见 `copydown-shim-port.md` 迭代 18-25；当日总结见
`2026-10-11.md`；GPU 侧见 `2026-10-12-gpu-bringup.md`。*
