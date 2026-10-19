# docs
sm7250 platform hardware info

Xiaomi Redmi K30 5G[(picasso)](picasso.md)

Bring-up 专题文档：
- [A620 完整支持 review 修复](2026-10-18-gpu-review-fixes.md)（当前定案：#82 已恢复 GMU/显示/触摸；#83 补 GPU cooling，本机 speed-bin 132，待真实 3D 专项）
- [A620 完整 GPU 支持（2026-10-19 #82 实机修订）](2026-10-16-gpu-full-support.md)（频率/DCVS、独立带宽、ACD、LM、preemption、IFPC、thermal）
- [Review 复核与修复记录](2026-10-16-review-fixes.md)（13 项确认问题已修改，本地验证通过，待完整构建和实机）
- [Review 复核与修复记录](2026-10-16-review-fixes.md)（13 项确认问题已修改；P41 镜像基础实机回归通过，解绑/模块等专项待测）
- [Linux 7.3 适配提交 review](2026-10-16-kernel-commit-review.md)（5 个适配提交、13 项问题及证据）
- [UFS P41 供电修复与验证基线](2026-10-16-ufs-power-fix.md)（最新：实机验证成功；调试代码已清理，清理版待回归）
- [音频链 bring-up](audio-bringup.md)（#89 实机已注册 WCD9385 ALSA card 并避免 SSR 重启；播放路径与 sensor_process 待处理）
- [ADSP sensor_process/PDR 修复](2026-10-19-adsp-sensor-pd.md)（#93 SID3 FastRPC ATTACH_SNS 已成功但 41 秒狗咬仍存在；#94 补 SMP2P sleepstate/sleepstate_see 与 apps-awake bit12，待实机回归）
- [2026-10-20 ADSP sensor_process 排查全记录](2026-10-20-adsp-sensor-process-debug-record.md)（#89→#94 完整时间线、逆向证据、失败路径、SID 根因与产物哈希）
- [显示链 bring-up 全记录](display-bringup.md)（迭代 18-25，屏幕点亮）
- [GPU bring-up 全记录](2026-10-12-gpu-bringup.md)（迭代 27，渲染管线）
- [触摸 bring-up 全记录](touch-bringup.md)（迭代 28，NT36672C SPI）
- [UFS bring-up 全记录](ufs-bringup.md)（迭代 29+，READ(10) 已由 P41 打通）
- [live_dts UFS 逐属性溯源](ufs-live-dts-attr-map.md)（对照下游 4.19 的属性作用全表）
- [迭代日志](copydown-shim-port.md)（全部轮次实录，先读再动）
- [移植文件清单](porting-filelist.md) / [构建手册](build-boot-manual.md)
- [TLMM 布局与 PDC wakeirq 映射](tlmm-wakeirq-map.md)
