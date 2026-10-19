# picasso（SM7250/lito）Adreno 620 GPU 主线拉起全记录

> 2026-10-09 ~ 2026-10-12。从零移植到 **GPU 全链打通**（GMU 固件 → GX 域上电
> → gfx3d 时钟 → CP/SQE 执行 → zap 安全认证 → msm 渲染管线首次真实出图）。
> 本文记录完整技术路径：每个故障的现象、排查方法、根因、修复，以及被证伪的
> 假说清单。配套迭代日志见 `copydown-shim-port.md` 迭代 20/25/27，产物 sha 表
> 亦在该文。

---

## 目录

1. [硬件与软件架构](#1-硬件与软件架构)
2. [阶段一：基础设施移植（gpucc/GDSC/catalog/DT）](#2-阶段一基础设施移植)
3. [阶段二：固件链（ROM 解包 / initramfs / SQE 版本）](#3-阶段二固件链)
4. [阶段三：GMU 上电（bw 表 / PDC / vrm.soc）](#4-阶段三gmu-上电)
5. [阶段四：GX 域 gfx3d 根时钟（主侦探战）](#5-阶段四gx-域-gfx3d-根时钟)
6. [阶段五：zap 安全认证（TZ 拒绝对齐不齐的装载区）](#6-阶段五zap-安全认证)
7. [最终形态：保留的代码修改](#7-最终形态保留的代码修改)
8. [验证方法（如何确认 GPU 在工作）](#8-验证方法)
9. [方法论与教训](#9-方法论与教训)
10. [遗留与展望](#10-遗留与展望)

---

## 1. 硬件与软件架构

### 1.1 上电链总览

picasso（Redmi K30 5G，SM7250 "lito"）的 GPU 是 **Adreno 620**（chipid
`0x06020002`，GEN3/a650 家族寄存器规格，GMEM 512KB，最高 750MHz）。它不是
"给时钟给电就能跑"的旧式 GPU，而是一条由 **GMU（Graphics Management Unit，
Cortex-M3 微控制器）** 管理的电源域机器：

```
                 ┌─────────────────────────── SoC ───────────────────────────┐
                 │                                                            │
  内核侧         │  GMU fw 侧                硬件侧                           │
  ────────       │  ─────────              ─────────                         │
  gpucc 驱动     │  a650_gmu.bin           GPU_CC（0x3d90000）              │
   PLL1 配置     │   (v2.1.8, CM3)          ├ PLL0 (fw 写, 750 档源)         │
   CX/GX GDSC   │      │ HFI 消息           ├ gfx3d RCG  @0x101c ──┐        │
  gcc            │      │ (perf/bw 表)       ├ gfx3d CBCR @0x1054   │根时钟 │
   gpll0 分支    │      ▼                    ├ gx_gmu CBCR@0x1064  │        │
  PDC/TCS 编程 ─┼─▶ RSC（GPU 专属）─▶ RPMh   └ ...                 ▼        │
   (睡眠/唤醒    │   轨道投票 gfx.lvl 等                          GPU 核心   │
    微码装载)    │                                               (0x3d00000) │
                 └────────────────────────────────────────────────────────────┘
```

关键概念：

| 概念 | 说明 |
|---|---|
| **CX/GX GDSC** | GPU 两个电源域。CX（GMU+外设）常驻，GX（GPU 核心）按需塌缩。GX 由 GMU fw 经 RSC→RPMh 投票 `gfx.lvl` 轨道后抬起 |
| **RSC / TCS / PDC** | GPU 专属 RSC（Resource State Coordinator）通过 TCS（Trigger Command Set）向 RPMh 发寄存器写；PDC 序列微码管睡眠/唤醒路径；**lito 的 PDC 由内核编程**（kona/a650 是 AOP 代编——这是重要平台差异） |
| **HFI** | 主机↔GMU 固件消息协议（队列共享内存）。perf/bw 表经 HFI 下发，DCVS 由 fw 自主执行 |
| **OOB** | Out-of-Band 请求（`GMU_OOB_GPU_SET` 等），经 HOST2GMU 中断位握手，让 fw 执行电源动作并 ack |
| **gfx3d 时钟树** | GPU 核心时钟：PLL0/PLL1/gpll0 源 → gfx3d RCG（选源/分频）→ 分支（CBCR）→ GPU 核心。**fw 自主切换**，内核不建模（kona/lito 下游 gpucc 均不注册 gfx3d） |
| **SQE/zap** | SQE = CP 微码（`a650_sqe.fw`）；zap = TZ 认证的安全着色器（PAS id 13），hw_init 末段装载 |

### 1.2 与参考平台的差异（决定排障方向的先验）

| | kona/sm8250（a650，主线工作） | **lito/sm7250（a620，本文）** | lagoon/sm6350/sm7225（a619） |
|---|---|---|---|
| GMU fw | a650_gmu.bin v2.1.8（**与我们逐字节相同**） | 同左 | a619_gmu.bin（另一代） |
| PDC | **AOP 代编**（内核跳过） | **内核必须编** | 内核编 |
| gpucc 内核建模 | 无 gfx3d（fw 自管） | 无 gfx3d（同） | **注册 gfx3d 树**（fw 代差） |
| adreno_smmu | 0x3da0000 | 0x3da0000（同） | 0x3d40000（不同管芯） |
| vrm.soc TCS 命令 | 无 | **a620 独有** | — |

> lito 与 kona 用**同一颗 GMU 固件**但行为不同——排障时"kona 主线能跑"只能
> 证明固件本身没问题，不能证明内核侧编程集相同。

---

## 2. 阶段一：基础设施移植

（2026-10-09~11，迭代 20/25；此处收录与 GPU 直接相关的部分，完整过程见
迭代日志。）

### 2.1 gpucc-sm7250.c（自下游 gpucc-lito.c 移植）

- 13 个时钟：gmu_clk_src RCG@0x1120（19.2/200/500MHz 三档）、ahb@0x1078
  （CRITICAL）、cx_gmu@0x1098、gx_gmu@0x1064、cxo/cxo_aon、hlos1_vote 等；
- LUCID PLL1@0x100（l=0x1A≈500MHz，gmu 高档源）；
- CX GDSC（gdscr 0x106c，hw_ctrl 0x1540，VOTABLE）/ GX GDSC（0x100c，
  clamp_io 0x1508，`gdsc_gx_do_nothing_enable`——GX 归 fw 管，内核"使能"
  只做引用计数）；
- probe 的 CX_GMU CBCR WAKE/SLEEP 域写 0xF（下游推荐值）。

**审计纪律**：与下游 gpucc-lito.c 逐项比对（偏移/投票位/halt 类型），确认
**lito 下游也不注册 gfx3d/PLL0**——它们完全归 fw。这决定了后续"内核不建模
gfx3d"的正确性边界。

### 2.2 catalog 条目与 chipid 陷阱

**问题**：首次手写 catalog 条目用了 `0x06020200`，实机打印
`Unknown GPU revision: 6.2.0.2`——不匹配。

**排查**：`find_chipid()`（adreno_device.c）解析 compatible
`qcom,adreno-620.2` 的规则是 core=6 / major=2 / **minor=0** / patch=2 →
**0x06020002**。手算把 minor/patch 位放反了。

**修复**：正确条目 `ADRENO_CHIP_IDS(0x06020002)`，family=GEN3，firmware
名与 ROM 逐字一致（sqe=`a650_sqe.fw`、gmu=`a650_gmu.bin`、zap=`a620_zap.mdt`），
speedbins {0→0, 132→1, 115→2}（下游 lito-gpu.dtsi 三个 SKU）。

**教训**：chipid 必须从解析代码逆推，不能手拼十六进制。（同类错误历史上
出现过两次：l2c 电压十六进制、shim 立即数地址——一律改用 python 换算。）

### 2.3 DT 节点与配套修复

- gmu 节点：4 段 reg（gmu/rscc@0x3de0000/gmu_pdc@0xb290000/gmu_pdc_seq@
  0xb490000，均下游实证）、5 时钟（ahb/gmu/cxo/axi/memnoc）、cx+gx 双
  power-domains、iommus<5>；
- gpu 节点：qcom,gmu 引用、nvmem speed_bin、zap-shader 子节点、
  275–750MHz OPP 表；
- **adreno_smmu 补 clocks（ahb/bus/iface）+ dma-coherent**：缺 AHB 时钟时
  probe 靠 ABL 残留态能过，但 CX 域第一次 off/on 循环后寄存器访问会挂；
- **qfprom cell**：`gpu-speed-bin@1ea` 的 `bits=<5,8>` 跨字节，`reg` 长度
  必须 2（写 1 → nvmem "invalid bits" → qfprom probe -22 → fw_devlink 把
  gpu 永久 -517，特征是 `returned -517 after ~8 usecs` 且无任何驱动打印）。

---

## 3. 阶段二：固件链

### 3.1 固件选型（数据源：YAAP ROM）

ROM vendor 分区（EROFS，解包工具链见迭代日志）内 GPU 固件全集：

| ROM 文件 | 用途 | 决定 |
|---|---|---|
| **a650_gmu.bin** | GMU fw v2.1.8 | **用**（下游 gpulist 写死；与 linux-firmware 逐字节相同） |
| **a650_sqe.fw** | CP 微码 v93 | 先用 ROM 版（门槛放宽）→ 终态换 **linux-firmware 版 v95**（门槛回滚上游 0x095） |
| **a620_zap.mdt+b00/01/02** | 安全着色器 | **用**（ROM 原名原内容） |
| a630_sqe.fw / a650_zap.* | 老平台/kona 残留 | 不用 |

**SQE 版本反复的教训**：曾按"固件必须与 ROM 一致"原则用 ROM v93 + 放宽
门槛；用户否决"改代码绕过检测"的思路后，回到 linux-firmware 新版 SQE +
上游原生门槛——GMU fw 与 SQE 无强配套约束（kona 主线同组合 v95+2.1.8 工
作），版本门槛是上游安全修复而非兼容性门槛。

### 3.2 initramfs 丢固件悬案（两个月 -ENOENT 之谜）

**现象**：`adreno_request_fw` 请求 `a650_sqe.fw` 恒 -2（-ENOENT），但构建
三环节（cpio.gz、usr/initramfs_inc_data、boot.img 内嵌）全部含该文件。

**排查**（耗时最长的一案）：
1. 静态逐层切片 cmp —— 打包层清白；
2. 实机 `dmesg` 无任何解包错误；`/lib/` 只有 musl 两文件——**"干净地"在
   cpio ~90% 处截断**（丢的恰为尾部固件段）；
3. `do_name` 逐条目探针 —— 内核**完整处理了全部条目**（12 固件+TRAILER，
   name/mode/body 全对）但文件没落地；
4. 对照 `do_name` 源码破案：`filp_open(O_CREAT)` 失败 **`return 0` 静默
   丢弃**、`init_mkdir` 返回值忽略——而 **spec 从第一天起就缺
   `dir /lib/firmware` 父目录条目**（gen_init_cpio 不自动建父目录）→
   `mkdir(qcom)` ENOENT 静默 → 12 个固件 open 全 ENOENT 静默。

**修复**：spec 补 `dir /lib/firmware`。次日实机
`loaded qcom/a650_sqe.fw from new location` —— 悬案终结。

**教训**：文件"在 cpio 里"≠"能落地"；内核 initramfs 解包的静默失败路径
极多，探针要打在 `do_name` 而不是打包侧。

---

## 4. 阶段三：GMU 上电

固件能加载后，GMU 依次暴露三个问题，每个都表现为"HFI/OOB 超时"但根因
各异——**同一症状三层根因**是本阶段的主题。

### 4.1 GMU graceful death：bw 表 fallback 链

**现象**：GMU fw 加载启动成功（`Loaded GMU firmware v2.1.8`），HFI 前 4 条
（INIT/FW_VERSION/PERF_TABLE/BW_TABLE）全部成功，随后 **HFI VOTE（id 5）
超时** → GMU 停滞 → 后续所有 OOB 连锁超时。DBRG-OOB 探针读
`cm3_status=0x900`（bit8 = graceful death 标志）。

**排查**：
- ARC votes 构建探针（DBRG-VOTES）：`ret=0`、5 档、`gx_arc[max]=0x1800606`
  （TURBO 384，解码合法）→ **votes 机制排除**；
- HFI 正常（消息有来有回）→ GMU 活着，是 fw **主动**趴了；
- 对照 `a6xx_hfi.c` 的 bw 表构建 fallback 链：`adreno_is_a650()` 是**精确
  chipid 匹配**（0x06010500）而非家族判断——我们的 0x06020002 **不中任何
  分支**，落到 `a6xx_build_bw_table`（a630 时代的 cmd-db 地址）→ fw 按错误
  地址执行 DDR/CNOC 投票 → fault → graceful death。

**修复**：

```c
else if (adreno_is_a650(adreno_gpu) ||
        (adreno_gpu->chip_id & 0xfffff000) == 0x06020000)
        /* picasso/lito A620：GMU 2.0/fw 2.1.8 与 kona a650 同代，
         * 用 a650 的 "single off" bw 表 */
        a650_build_bw_table(msg);
```

### 4.2 OOB GPU_SET 超时：PDC 归属误判

**现象**：bw 表修复后 VOTE 不再超时，但 **OOB GPU_SET 独立超时**（fw 对
"切 GX 域"类请求不应答）。HFI 依旧正常 → 电源路径问题。

**排查**：寄存器偏移主线 vs 下游逐一对证全部一致后，在下游
`adreno-gpulist.h` 找到决定性差异：**kona a650 的 `pdc_in_aop=true`（AOP
代编 PDC），lito a620 无此标志（内核必须编 PDC）**。主线按
`adreno_is_a650_family()` 硬判 `pdc_in_aop=true` → 跳过 PDC 序列/TCS 装载
→ lito 的 PDC 空 → GMU/RSC 无法执行电源切换。

**修复**（a6xx_gmu.c `a6xx_gmu_rpmh_init`）：

```c
bool lito_a620 = (adreno_gpu->chip_id & 0xfffff000) == 0x06020000;
if ((adreno_is_a650_family(adreno_gpu) && !lito_a620) || adreno_is_a7xx(...))
        pdc_in_aop = true;
...
else if (lito_a620)
        pdc_address_offset = cmd_db_read_addr("xo.lvl") ?: 0x30080;
```

（TCS1 的 mx/cx 地址同样动态化——主线写死的是 kona 值。）

### 4.3 vrm.soc：a620 独有的 PDC 编程

系统对照下游 `adreno_a6xx_gmu.c` 全部 `adreno_is_a620()` 分支时发现，
lito 还有一条主线完全没有的 PDC 编程——**TCS1/TCS3 各加第 4 条命令写
`vrm.soc+4`**（RPMh Voltage Regulator Manager 握手）：

```c
if (vrm_resource_addr && adreno_is_a620(adreno_dev)) {
        /* TCS1: vrm.soc+4 = 0x0；TCS3: vrm.soc+4 = 0x1 */
}
```

实测本机 cmd-db 有该资源（addr=0x43400）。已对齐移植（保留）。后文会看到
它在冷启动路径不生效（PDC 序列只在睡眠↔唤醒切换时执行），但它是下游实证
的正确行为，且是 IFPC/深度睡眠路径的必要条件。

### 4.4 附带修复：750MHz 档消失

OPP 表 750MHz 档 `opp-supported-hw=0x03`，fuse 读出的 speedbin 落 0x04 时
最高档被滤掉（freq 表只剩 625MHz）。扩为 `0x07`。另：opp-bandwidth 实验
（opp-peak/avg-kBps）因 GPU 节点无 interconnect 路径导致 OPP 解析 -22 →
GPU bind 失败连带 DSI/panel 全挂——回滚（上游 sm8250 GPU 节点本就不挂 ICC，
带宽投票走 GMU bw 表）。

**阶段三战果**：VOTE/OOB 全部不再超时、750MHz 档恢复、zap 请求可达——
GMU 层打通，战线推进到 GPU 核心层。

---

## 5. 阶段四：GX 域 gfx3d 根时钟

> 2026-10-12 主战役。GMU 全通后 hw_init 卡在 ringbuffer 不 drain：
> `rptr=0/wptr=9`——CP 不消费命令。这是整个 GPU bring-up 最深的一案，
> 共 9 轮刷机、7 个假说证伪。

### 5.1 现象固定（CP1-CP5 探针）

在 `a6xx_cp_init` 的 flush 后加全现场探针（一次加全，避免反复刷机）：

```
DBRG-CP1: sqe_cntl=0x0 rb_base=0x0 rb_cntl=0x0 gx_on=0     ← GPU 寄存器读回全零
DBRG-CP2: rbbm_status=0x0                                    ← 同上
DBRG-CP5: sptprac_status=0x80 gx_gdscr=0xf8222800 rscc_seq_busy=0x0
```

`SPTPRAC_PWR_CLK_STATUS`（0x1f8d0）位定义（下游 a6xx_reg.h 同义）：

| bit | 含义 | 实测 |
|---|---|---|
| 6 | GX_GDSC_POWER_OFF | **0**（电源域已上！） |
| 7 | GX_HM_CLK_OFF | **1**（核心时钟死） |

即：**GX 轨已由 fw 抬起，唯独 gfx3d 时钟不启动，GPU 核心寄存器读写全灭**
（GX 时钟死时写丢弃、读恒 0——这同时解释了 CP 探针全零与 ringbuffer 不 drain）。

### 5.2 假说清单与证伪过程

| # | 假说 | 判别实验 | 判决 |
|---|---|---|---|
| 1 | RCG 低频档源（gcc gpll0 分支）无消费者无人 enable | gpucc probe 保活两分支（`devm_clk_get`+`clk_prepare_enable`） | ✗ CP5 逐位不变（已回滚） |
| 2 | OOB ack 是假象（fw 用旧位应答） | 对照下游 OOB 位定义：a620 set=BIT(30)/ack=BIT(31) 与主线一致 | ✗ ack 为真 |
| 3 | fw 根本没执行上电动作 | **CP0 探针**（OOB 前后状态）：`pre 0xc0 → post 0x80` | ✗ **fw 真的抬了 GDSC**（GDSCR 从 SW_COLLAPSE 0x222001 → 0xf8222800） |
| 4 | CPU 访问被 GMU AHB fence 拒绝 | 读 `AHB_FENCE_STATUS/AO_AHB_FENCE_CTRL`：全 0 | ✗ fence 不在拒绝态 |
| 5 | 内核 gpucc 移植有偏差 | 与下游 gpucc-lito.c 逐项 diff | ✗ 逐字一致 |
| 6 | PLL0 输出死 | RCG 切 XO（19.2MHz 必活）+ 仅发 UPDATE | ✗ 根仍 off → **当时误判"PLL0 无罪"**（见 5.4 翻案） |
| 7 | cx_gfx3d 分支（lagoon 有）没开 | 写 0x10a4/0x10a8 | ✗ **写不粘**——寄存器不存在（lagoon 独有；下游 snapshot 名单佐证） |
| 8 | RCG 缺 UPDATE | 内核补发 `CMD bit0` | ✗ UPDATE 被接受（自清）但无效 |
| 9 | RCG 缺根使能 | `CMD_ROOT_EN(bit1)` 单独写 | △ ROOT_OFF 清了但分支/SP 状态不跟——**不充分** |

### 5.3 决定性证据：GPU CC 全量 dump 与 GMU fw log

**GPU CC 寄存器全量 dump**（gmu_read 走 GPU 基址相对 dword 偏移，
`GMU_BYTE_OFFSET` 自动折算）——fw 的上电足迹完整无缺：

```
PLL0  mode=0xd9000005  L=0x4e   ← bit0=EN, bit27=LUCID_PCAL_DONE, bit31=PLL_LOCK_DET：报"已锁定"
gfx3d RCG  CMD @0x101c = 0x80000000 (ROOT_OFF)   CFG @0x1020 = 0x101 (src1=PLL0, /1)
gfx3d CBCR @0x1054      = 0x80004221 (enable=1, CLK_OFF=1)
gx_cxo   @0x1060        = 0x1      ← fw 开成功了（活源分支能开！）
ACD clk  @0x1164/0x1168 = 0x1      ← 同上
DOMAIN_MISC(夹钳) 0x1508 bit0=0    ← 夹钳已放行
```

**GMU fw log**（16KB 环形缓冲，4 字/条：`id, timestamp, seq, data`）：

```
(0x23, ts, 1, 0xdeadba55)
(0x23, ts, 2, 0xdeadba55)
(0x23, ts, 3, 0xdeadba55)
...（共 58 条，seq 到 0x3a）
(0x21, ts, 2, 0x00103b04)   ← 失败后的状态转储
(0x21, ts, 3, 0x0010304)
(0x21, ts, 2, 0x0020a003)
```

**fw 自己在重试某个操作 58 次后放弃**——与"命令到位、根不起"完全吻合。

### 5.4 破案：组合拳 + 活源（矩阵与翻案）

**逐项补完实验（ST1-ST8）**最终点亮：

```
DBRG-ST8 combo: cmd=0x2 cbcr=0x1 sp=0x0 rbbm=0x1   ← GPU 复活！
DBRG-CP1: sqe_cntl=0x1 rb_base=0x1000 gx_on=1
DBRG-CP3: sqe_instr=0x9000                          ← SQE 真的取指执行了
```

完整判别矩阵（含后续清理版回归的补测）：

| gfx3d 源 | 组合拳* | 结果 |
|---|---|---|
| PLL0（fw 配置，mode 报 LOCKED） | ✗ | ✗ |
| PLL0 | ✓ | ✗（清理版 v1 实测：`gfx3d root clock did not turn on`） |
| XO | ✗ | ✗ |
| **XO** | **✓** | **✓ 全链通过** |

\* 组合拳 = `CMD_ROOT_EN` 写+轮询 → gfx3d 分支 0→1 重触发 → `CMD=ROOT_EN|UPDATE`。

**根因（两层）**：
1. **fw 的上电序列无法独自点亮 gfx3d 根时钟**（重试 58 次自曝）——需要内核
   补 `ROOT_EN` 强制 + 分支重触发 + UPDATE 组合；
2. **PLL0 的 lock 位在撒谎**——mode 位报 LOCKED/PCAL_DONE 但输出实际死，
   组合拳只对真正运行的源有效。此前"PLL0 无罪"是**单变量判别的冤案**：
   ST6（切 XO 只发 UPDATE）缺组合拳，把"源死"和"缺使能"两个变量混在
   了一起。清理版 v1 丢掉 CFG→XO 切换直接翻车，才逼出完整矩阵。

### 5.5 修复固化（hw_init 中，OOB GPU_SET 之后）

```c
if ((adreno_gpu->chip_id & 0xfffff000) == 0x06020000 &&
    (gmu_read(gmu, REG_A6XX_GMU_SPTPRAC_PWR_CLK_STATUS) & 0x80)) {
        /* gx_gmu 分支使能（成功路径实证步骤） */
        gmu_write(gmu, 0x24419 /* GX_GMU_CBCR 0x1064 */, 0x1);
        udelay(20);
        /* RCG 切 XO（src0，必活）+ UPDATE */
        gmu_write(gmu, 0x24408 /* gfx3d RCG CFG 0x1020 */, 0x001);
        gmu_write(gmu, 0x24407 /* CMD 0x101c */, 0x1);
        udelay(100);
        /* ROOT_EN 强制根使能（主线 clk_rcg2_set_force_enable 原语） */
        gmu_write(gmu, 0x24407, 0x2);
        for (w = 0; w < 500; w++) { /* 轮询 ROOT_OFF 清零 */ }
        /* 分支 0→1 重触发 + 组合 UPDATE */
        gmu_write(gmu, 0x24415 /* GX_GFX3D_CBCR 0x1054 */, 0x0);
        udelay(20);
        gmu_write(gmu, 0x24415, 0x1);
        gmu_write(gmu, 0x24407, 0x3 /* ROOT_EN|UPDATE */);
        udelay(100);
        /* 失败告警 */
}
```

幂等门（SPTPRAC bit7 已清则整段跳过）保证 fw 未来版本自行修复时不冲突。
**代价：GPU 目前以 XO 19.2MHz 运行**（功能全通、性能受限，高频档见 §10）。

---

## 6. 阶段五：zap 安全认证

### 6.1 现象

gfx3d 点亮后战线自然推进，暴露下一关（此前各轮都死在 drain、从未真正跑到
zap）：

```
loaded qcom/a620_zap.mdt from new location
adreno 3d00000.gpu: error -22 setting up firmware qcom/a620_zap.mdt
```

### 6.2 定位错误来源

zap 加载链：`zap_shader_load_mdt → qcom_mdt_load → __qcom_mdt_pas_init`。
三个可能报错点打印不同：

- `error %d reading firmware metadata`——metadata 读取；
- `error %d initializing firmware`——**PAS_INIT_IMAGE**（TZ 验签）；
- `error %d setting up firmware`——**PAS_MEM_SETUP**（TZ 登记装载区）。

实机打印是第三种 → **签名验证已通过**（固件没错！），是**装载区被 TZ 拒**。
调用参数（由 mdt 程序头计算）：

```
PAS_MEM_SETUP(pas_id=13, mem_phys=0x8bc15400, size=0x1000)
```

a620_zap.mdt 程序头（实测解析）：唯一 PT_LOAD `paddr=0x5000, memsz=0x68c,
flags=0x08000007`（bit27=RELOCATABLE）→ span=ALIGN(0x68c)-0x5000=0x1000。

### 6.3 根因：装载区未 4K 对齐

主线 zap 路径把 DT 固定保留区 `pil-gpu-mem@8bc15400` 直传 TZ——该地址
**0x400 未对齐**，本机 TZ 的 PAS_MEM_SETUP 拒绝之（-EINVAL）。

法证关键：对照下游 zap 的真实装载路径——下游根本不走 GPU 驱动直载，而是
`subsystem_get("a620_zap")` → PIL 框架（`pil-tz-generic`）→
`peripheral-loader.c:pil_alloc_region()`：

```c
/* 下游 4.19：relocatable 镜像的装载区是 dma_alloc_attrs 动态分配的，
 * 尺寸按镜像 span 向上对齐（4K/1M/4M），与 DT 保留区无关 */
region = dma_alloc_attrs(priv->desc->dev, aligned_size, &priv->region_start, ...);
```

即 **Android 上从没有人把 0x8bc15400 传给 TZ**——下游 zap 装载区是内核
动态分配的 4K 对齐内存。（kona 主线用同样未对齐的固定区能过，属该平台
TZ 校验宽松，不能类推 lito。）

### 6.4 修复（adreno_gpu.c，lito 分支）

```c
if ((to_adreno_gpu(gpu)->chip_id & 0xfffff000) == 0x06020000) {
        dma_addr_t zap_phys;
        void *zap_mem = dmam_alloc_coherent(dev, SZ_4K, &zap_phys, GFP_KERNEL);
        ...
        ret = qcom_mdt_load(dev, fw, loadname, pasid, zap_mem, zap_phys, SZ_4K, NULL);
        if (!ret)
                ret = qcom_pas_auth_and_reset(pasid);
        ...
}
```

### 6.5 终局证据

```
[1.0255] loaded qcom/a620_zap.mdt from new location     ← 后面再无错误行
[1.3680] Console: switching to colour frame buffer device 135x150
[1.3851] [drm] fb0: msmdrmfb frame buffer device         ← adreno_load_gpu 成功
[1.4237] Run /init → shell（1.02-1.45s 零错误）
```

**GPU 显示渲染链全通。** 最终清理版 `boot_7.3.img` sha256
`beae4aad…`（探针全移除、quirk 收编正式代码）。

---

## 7. 最终形态：保留的代码修改

| 文件 | 修改 | 性质 |
|---|---|---|
| `drivers/gpu/drm/msm/adreno/a6xx_catalog.c` | 0x06020002 条目（fw 名/speedbins/.bcms/gmu_cgc_mode 等） | 平台支持 |
| `drivers/gpu/drm/msm/adreno/a6xx_hfi.c` | a620 走 a650 "single off" bw 表 | 平台支持 |
| `drivers/gpu/drm/msm/adreno/a6xx_gmu.c` | ① lito 排除出 pdc_in_aop 组 + cmd-db 动态 TCS 地址；② vrm.soc TCS1/TCS3 第 4 条命令 | 平台支持 |
| `drivers/gpu/drm/msm/adreno/a6xx_gpu.c` | hw_init 的 gfx3d 根时钟 quirk（§5.5，幂等） | 平台 quirk |
| `drivers/gpu/drm/msm/adreno/adreno_gpu.c` | zap 装载区 lito 分支动态分配 | 平台 quirk |
| `drivers/clk/qcom/gpucc-sm7250.c` | 自下游 gpucc-lito.c 移植（13 时钟+PLL1+双 GDSC） | 新驱动 |
| `drivers/clk/qcom/gcc-sm7250.c` | gpu_gpll0 两分支（0x52000 bit15/16） | 时钟补全 |
| `arch/arm64/boot/dts/qcom/sm7250-xiaomi-picasso.dts` | gpu/gmu/gpucc/adreno_smmu 节点 + OPP + qfprom cell + zap 保留区 | DT |
| initramfs 固件 | a650_sqe.fw(lfw) / a650_gmu.bin / a620_zap.mdt+b00-02 | 固件 |

---

## 8. 验证方法

### 8.1 大陷阱：屏幕有画面 ≠ GPU 在工作

`fb0: msmdrmfb` 与 fbcon 出图走 **DPU 显示管线**（CPU 写帧缓冲），GPU 死了
照样亮屏（清理版 v1 失败那次屏幕就有画面）。

### 8.2 判据分层

```sh
# ① 启动日志组合判据（必要条件，一条命令）
dmesg | grep -E "gpu hw init|ringbuffer|gfx3d root|zap|Initialized msm"
#   成功 = Initialized msm ✓ + zap 行后无 error + 三个否定词全无

# ② devfreq sysfs（绑定层，弱判据——hw_init 失败它也在）
ls /sys/class/devfreq/3d00000.gpu/ && cat .../cur_freq

# ③ /proc/interrupts（gpu-irq 计数 >0 需要真实 3D 提交，mesa 阶段才有意义；
#    hfi/gmu 计数为 0 属正常——此代驱动应答走轮询路径）
cat /proc/interrupts | grep -E "gpu|hfi|gmu"

# ④ 终极：CONFIG_DEBUG_FS=y 后
cat /sys/kernel/debug/dri/0/gpu        # 实时 GPU 寄存器快照
#   或 rootfs + mesa 后跑真实 3D 负载
```

---

## 9. 方法论与教训

1. **对照三源**：主线参考平台（kona，同固件）证明"固件没错"；lito 下游内核
   是行为权威；lagoon（sm6350/sm7225）是表亲不是同管芯（smmu 地址/rscc
   布局不同，其 gpucc 注册 gfx3d 是 fw 代差行为，不可照抄）。每轮"主线
   sm8250 vs sm6350 diff → 推 sm7250 错位"（用户指示）正是靠这个框架。
2. **单变量判别必须含完整组合**：ST6"切 XO 单独无效"导致"PLL0 无罪"的
   冤案——源死与缺使能是两个变量，判别实验里组合拳缺席就把它们混住了。
   清理版 v1 的回归才补全矩阵。
3. **状态位会撒谎**：PLL0 mode 的 LOCKED/PCAL_DONE 全置位但输出死。状态
   位只有配合"下游动作可见"（分支 CLK_OFF 清零、寄存器读回非零）才能采信。
4. **fw 有自己的证词**：GMU log 环形缓冲（4 字/条）是 fw 视角的第一手
   证据——0xdeadba55 ×58 直接指明"fw 在重试且失败"。
5. **症状相同根因多层**：HFI/OOB"超时"先后对应 bw 表错、PDC 缺编、根时钟
   缺使能三层；"寄存器读零"先后对应 GX 时钟死、（被证伪的）fence 拒绝。
   每层修复后症状推移，要跟着战线重新取证。
6. **静默失败路径优先怀疑**：initramfs 父目录（return 0 丢文件）、zap 装载
   区（TZ 拒绝只回 -22）。
7. **一次刷机加全探针**（用户纪律）：CP0/CP1-5/CP5G 全量 dump/GMU log
   32 字/ST1-8 全部一次入镜像，每轮日志榨干信息量。

---

## 10. 遗留与展望

| 项 | 说明 | 优先级 |
|---|---|---|
| **GPU 高频档** | 现跑 XO 19.2MHz。候选：内核侧用主线 lucid ops 真正起 PLL0（fw 的使能序列不完整）；或切 gpll0 600MHz（需回加 gcc gpu_gpll0 分支保活 + RCG src5） | 高 |
| `CONFIG_DEBUG_FS=y` | 拿到 `/sys/kernel/debug/dri/0/gpu` 寄存器级实时判据 | 低成本高收益 |
| DCVS 与 quirk 的长期相容 | devfreq 降档会触发 fw DCVS 重写时钟树——当前无真实 3D 负载未暴露问题，mesa 阶段需回归验证 quirk 幂等门是否足够 | 中 |
| 触摸 bring-up | I2C_GENI 已开，gpio4/5 在白名单 | 高（交互） |
| UFS rootfs | 解锁完整用户态（mesa/真 3D） | 高 |
| watchdog 恢复接管 | 当前 bring-up 模式停狗 | 量产前 |
| IFPC/深度睡眠 | 下游 a620 特性（含 vrm.soc 的真正用武之地），冷启动不需要 | 功耗阶段 |

---

*成文于 2026-10-12 GPU 全链打通之日。排障过程逐轮实录见
`copydown-shim-port.md` 迭代 20/25/27 与 sha256 速查表。*
