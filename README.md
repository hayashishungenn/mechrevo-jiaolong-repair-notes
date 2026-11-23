# 机械革命蛟龙（Ryzen 9 9955HX）蓝牙消失与音频爆音维修记录

> 记录日期：2026-11-23 · Windows 11 24H2/25H2 · 状态：已修复
>
> 同一台机器上同时存在两颗问题芯片（MediaTek MT7922 无线网卡 + Realtek RTL8125 有线网卡），
> 蓝牙消失、音频爆音、喇叭掉线三个看似独立的症状实为同源故障。

## 机器信息

| 项目 | 型号 |
|---|---|
| 机型 | 机械革命蛟龙系列（MECHREVO JIAOLONG） |
| CPU | AMD Ryzen 9 9955HX（Fire Range，16 核） |
| GPU | NVIDIA GeForce RTX 5070 Ti Laptop |
| 无线网卡 | MediaTek MT7922（Wi-Fi 6E，AzureWave 模块，SUBSYS_59111A3B，WiFi 走 PCIe / 蓝牙走 USB） |
| 有线网卡 | Realtek RTL8125 2.5GbE（PCIe，挂于 AMD GPP 根端口 0:1:6） |

## 症状

1. **蓝牙整体消失**：任务栏蓝牙图标、设置页蓝牙开关全部不见；设备管理器中
   `MediaTek Bluetooth Adapter` 变为幽灵设备（`CM_PROB_PHANTOM`，Present=False），
   普通重启无效
2. **音频爆音**（早于蓝牙问题出现）：内置喇叭突然出现爆音/杂音
3. **喇叭掉线**：爆音反复发作后，音频输出设备整体消失，任务栏喇叭图标变红叉（X）

## 诊断关键证据

| 证据 | 数据 | 指向 |
|---|---|---|
| WHEA-Logger 事件 17（已更正硬件错误） | 30 天 11133 条，其中 11116 条（99.8%）来自 PCI 根端口 0:1:6 | 错误风暴源头 |
| 根端口 0:1:6 下挂设备 | RTL8125 有线网卡 | 风暴由 Realtek 网卡链路产生 |
| MT7922 蓝牙适配器 | `CM_PROB_PHANTOM`，Present=False，蓝牙支持服务（bthserv 等）全部正常 | 硬件级掉线，非软件故障 |
| BTHUSB 事件 18（无法存储链接密钥） | 掉线前每 5 分钟一条，持续数小时 | 蓝牙固件挂死前兆 |
| 时间关联 | 错误风暴持续爆发期间蓝牙挂死；爆音"早就有了"对应单日 10008 条的风暴峰值（11-20） | 三症状同源 |
| 驱动版本 | 音频 10.0.1.38（2024/4）、无线 3.4.0.1115（2024/10）、芯片组旧版 | 驱动陈旧放大问题 |

### 根因结论

- **错误风暴**：RTL8125 有线网卡的 PCIe 链路持续产生已更正（corrected）AER 错误，
  由 AMD 根端口上报。风暴期间总线传输被打断 → 音频流丢包（爆音）→ 剧烈时下游设备
  固件挂死（喇叭掉线变红叉）。
- **蓝牙消失**：MT7922 蓝牙控制器固件挂死后从 USB 总线消失。该卡 WiFi 走 PCIe、
  蓝牙走 USB，故 WiFi 正常而蓝牙消失。挂死状态热重启无法清除，**只有彻底断电能复位**。
- 两颗芯片均为跨品牌、跨平台的已知问题芯片，恰巧装在同一台机器上互相放大了观感。

## 修复方案（按执行顺序）

1. **彻底断电冷启动**：关机 → 拔电源 → 长按电源键 30 秒 → 静置后开机
   —— 蓝牙恢复的唯一方法；普通重启（含快速启动模式下的关机再开）无效
2. **关闭 PCIe 链接状态电源管理**：电源计划中 PCI Express → 链接状态电源管理设为
   "关闭"（交流 + 电池两种模式）
3. **停用 Realtek RTL8125 有线网卡**（本机从不用有线网）—— 错误风暴源头直接切断
4. **更新驱动**：
   - MT7922 WLAN：3.4.0.1115 → 3.6.0.1434（Microsoft Update Catalog）
   - AMD 芯片组：8.09.28.1124（AMD 官方）
   - 音频 APO：1.0.4.55794（Windows Update）
5. **BIOS**（可选，本次未执行）：出厂 N.1.12MRO13（2025/4），可查厂商新版

## 修复后验证

- 停用有线网卡后 WHEA 事件 17 **新增 0 条**（此前半小时可新增 1000+ 条）
- MT7922 WLAN 驱动 3.6.0.1434 已绑定，WiFi 正常连接（2.4 Gbps）
- 蓝牙适配器恢复在线（状态 OK），全部外设可正常配对

## 快速恢复速查

**蓝牙再消失（MT7922 老毛病，新驱动只能降低概率）**：

> 关机 → 拔电源适配器 → 长按电源键 30 秒 → 静置 1~2 分钟 → 插电开机

注意：必须是"重启"或彻底断电，快速启动开启时"关机再开"不算冷启动（内核从休眠镜像恢复，
坏的设备状态会被原样带回）。

**有线网卡恢复**（如需临时使用有线网）：设备管理器 → 网络适配器 → 右键 Realtek Gaming
2.5GbE → 启用设备。用完建议再停用。

**复发判断方法**：事件查看器筛选 System 日志、来源 Microsoft-Windows-WHEA-Logger、
事件 ID 17。数量快速增长即错误风暴复发，优先检查电源计划里 ASPM 是否被重置、
有线网卡是否被启用。

## 跨平台佐证（Linux / GitHub 同类报告）

蓝牙消失（MT7922/7921/7925 同族）：

- [omarchy-pkgs #2152：MT7922 Bluetooth USB disappears](https://github.com/omacom/omarchy-pkgs/issues/2152)
- [Arch Linux BBS #308169：No Bluetooth adapters found，拔电源可解（已结帖）](https://bbs.archlinux.org/viewtopic.php?id=308169)

PCIe 错误风暴（RTL8125 / r8169 同族）：

- [kernel Bugzilla #215027：corrected 物理层错误刷屏，缓解方案为关 ASPM](https://bugzilla.kernel.org/show_bug.cgi?id=215027)
- [kernel Bugzilla #215644：corrected 数据链路层错误风暴](https://bugzilla.kernel.org/show_bug.cgi?id=215644)
- [kernel Bugzilla #221606：r8169 AER 错误与挂起后恢复失败](https://bugzilla.kernel.org/show_bug.cgi?id=221606)
- [Launchpad #2015670：RTL810xE PCIe Bus Error 死循环](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/2015670)

Linux 社区总结的缓解方案（关 ASPM、断电冷启动、禁用 AER 报告、换卡）与本记录的
Windows 方案一一对应。

## 遗留事项

- AMD 音频主驱动（HD Audio 10.0.1.38，2024/4）未更新：更新渠道在 AMD Adrenalin
  完整安装包内，需连带 GPU 软件，暂缓观察
- 机械革命 BIOS 新版本未确认（刷写有风险，仅在前述方案失效时考虑）
- 长期方案（如错误风暴复发且停用网卡无法接受）：更换 M.2 无线网卡（如 Intel AX210）
  与加装独立 PCIe 有线网卡/更换主板级方案
