# picasso TLMM 布局与 PDC wakeirq 官方数据（live_dts 实测提取）

来源：live_dts.txt 的 `pinctrl@f000000` 节点（Android 运行时导出）+ 下游
`pinctrl-lito.c`。由 tlmm-verify 工作流（3 agent 并行取证）交叉验证。

## TLMM 布局（已实机闭环验证）

- 基址 **0x0f000000**，单段 16MB（`reg = <0xf000000 0x1000000>`），汇总中断
  GIC SPI 208 电平。
- **4 tile 交错**（寄存器偏移 = tile + 0x1000×全局号）：NORTH +0x0 /
  SOUTH +0x100000 / WEST +0x500000 / EAST +0x900000。gpio 号→tile 分配
  无规律（如 0-5 EAST、6-10 WEST、75-102 大段 SOUTH、112-145 WEST）。
- **实证锚点**：gpio66(EAST) cfg = 0xf942000 == 小米 onewire 驱动硬编码
  （`mi,onewire-gpio-cfg-addr`，绕过 pinctrl 框架直写）。
- **gpio0-5 中 gpio0/3/4/5 是 PDC direct-connect 唤醒脚**（PINGROUP 尾参
  0xB4000 bit0-3）；gpio0 的 TLMM intr 寄存器写挂（疑 PDC/TZ 接管）。
  **主线 6.17 完全惰性（handler 只扫 enabled_irqs 位表）不会触碰**——勿加
  任何"遍历全部引脚"的代码。
- 特殊组（绝对偏移）：SDC1 = WEST+0xb8000、**SDC2 = SOUTH+0xb7000**、
  **UFS_RESET = WEST+0xbc000**。
- 146 脚 PINGROUP 的 index==id 全量核对通过（0 错位）。

## gpio→PDC wakeirq 官方映射（87 条，live_dts `irqdomain-map` 提取）

主线 6.17 **无 irqdomain-map binding**（4.19 通用解析），当前 DT 已摘除
wakeup-parent（gpio 中断直走 GIC 可用）。将来恢复深度睡眠唤醒时，按此表
构造 `sm7250_pdc_map[]`（或用主线 wakeup-parent + wakeirq_map 机制）：

```
gpio 0 -> pdc 58    gpio 3 -> pdc 44    gpio 4 -> pdc 80    gpio 5 -> pdc 89
gpio 6 -> pdc 52    gpio 9 -> pdc 104   gpio 10 -> pdc 54   gpio 11 -> pdc 62
gpio 22 -> pdc 64   gpio 24 -> pdc 82   gpio 26 -> pdc 56   gpio 30 -> pdc 84
gpio 31 -> pdc 43   gpio 32 -> pdc 79   gpio 33 -> pdc 66   gpio 34 -> pdc 53
gpio 36 -> pdc 92   gpio 37 -> pdc 63   gpio 38 -> pdc 73   gpio 39 -> pdc 76
gpio 41 -> pdc 81   gpio 42 -> pdc 94   gpio 43 -> pdc 55   gpio 45 -> pdc 83
gpio 46 -> pdc 57   gpio 47 -> pdc 86   gpio 48 -> pdc 121  gpio 49 -> pdc 87
gpio 50 -> pdc 90   gpio 52 -> pdc 72   gpio 53 -> pdc 96   gpio 55 -> pdc 91
gpio 56 -> pdc 135  gpio 57 -> pdc 137  gpio 58 -> pdc 93   gpio 59 -> pdc 136
gpio 62 -> pdc 97   gpio 64 -> pdc 65   gpio 65 -> pdc 75   gpio 66 -> pdc 98
gpio 67 -> pdc 99   gpio 68 -> pdc 100  gpio 69 -> pdc 46   gpio 70 -> pdc 85
gpio 72 -> pdc 50   gpio 73 -> pdc 45   gpio 74 -> pdc 101  gpio 78 -> pdc 42
gpio 82 -> pdc 88   gpio 83 -> pdc 51   gpio 84 -> pdc 102  gpio 85 -> pdc 113
gpio 86 -> pdc 95   gpio 87 -> pdc 103  gpio 88 -> pdc 114  gpio 90 -> pdc 105
gpio 97 -> pdc 70   gpio 98 -> pdc 106  gpio 100 -> pdc 59  gpio 103 -> pdc 60
gpio 105 -> pdc 71  gpio 107 -> pdc 78  gpio 108 -> pdc 61  gpio 109 -> pdc 134
gpio 110 -> pdc 48  gpio 111 -> pdc 67  gpio 112 -> pdc 68  gpio 113 -> pdc 77
gpio 114 -> pdc 69  gpio 115 -> pdc 133 gpio 116 -> pdc 109 gpio 117 -> pdc 120
gpio 119 -> pdc 110 gpio 121 -> pdc 119 gpio 122 -> pdc 112 gpio 124 -> pdc 107
gpio 126 -> pdc 111 gpio 127 -> pdc 116 gpio 128 -> pdc 132 gpio 133 -> pdc 131
gpio 135 -> pdc 130 gpio 138 -> pdc 129 gpio 139 -> pdc 128 gpio 140 -> pdc 127
gpio 141 -> pdc 126 gpio 142 -> pdc 125 gpio 144 -> pdc 122
```

PDC 段映射（qcom,pdc-ranges，line→GIC SPI）：0-41→480、42-69→612、
70→63、71-85→640、86-137→522。

## other_works 参照（实机点亮 simplefb/UFS/USB 的移植）

**完全没有 TLMM/pinctrl**（dtsi 无节点、无驱动）——引脚全靠 XBL/ABL
预配置，内核不碰。这是"能启动"的最保守基线，反向印证：**TLMM 惰性使用
（不碰无主引脚）是正确路线**。
