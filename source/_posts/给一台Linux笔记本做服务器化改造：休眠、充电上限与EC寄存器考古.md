title: 给一台 Linux 笔记本做服务器化改造：休眠、充电上限与 EC 寄存器考古
date: 2026-08-29 15:30:00
tags: [Technique, Linux]
---

一台旧笔记本装上 Hermes Agent 之后，就成了 7×24 挂在家庭网络里的常驻服务机。机器继续服役，但"笔记本"这个出厂身份留下的问题需要逐个拆除：拔电半小时自动休眠、合盖掉线、电池永远充到顶。这篇文章记录一次完整的改造过程——大部分是常规配置，最后一段是真正的硬骨头：Linux 下没有现成驱动的 Redmi EC，如何从固件代码里考古出充电上限的寄存器地址，并把 80% 上限真正写进硬件。

改造对象是 Redmi Book Pro 15 2022，系统 Pop!_OS 24.04，内核 6.18.7。文中所有验证步骤都给出了可复核的证据，文末附完整的脚本清单。

## 休眠问题的两个来源：一个在用户会话，一个在系统层

先说结论：这台机器的自动休眠由两套机制叠加控制，只改任何一处都会被另一处重新睡掉。

第一处在 GNOME。`org.gnome.settings-daemon.plugins.power` 这组键值把插电和用电池分成两个策略：这台机器出厂默认插电模式 `nothing`（永不睡），电池模式 `suspend`（30 分钟无操作就睡）。服务器场景下，这个"电池模式"就是拔电失联的直接原因。禁用只需两条命令：

```bash
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-type 'nothing'
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
```

第二处在 systemd-logind。不处理它，合盖依然会触发 suspend。做法是往 `/etc/systemd/logind.conf.d/` 放一个drop-in 配置：

```ini
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
AllowSuspend=no
AllowHibernation=no
```

`AllowSuspend=no` 这几行比合盖策略更进一步——它让 suspend/hibernate 这两个电源目标在系统层面不可达。从此无论哪个组件想休眠这台机器，systemd 都会拒绝执行。对服务器而言这是最保险的一层。

## 充电上限：配置了两年，从来没生效过

机器上一直装着 TLP，配置文件里写得明明白白：

```
START_CHARGE_THRESH_BAT0=20
STOP_CHARGE_THRESH_BAT0=80
```

但电量依然冲到了 94%（对应衰减后的满充点）。原因在 TLP 的工作方式：它自己不碰硬件，只调用内核提供的接口。Linux 内核标准接口是 `/sys/class/power_supply/BAT0/charge_control_end_threshold`，TLP 依次尝试三种后端——`natacpi`（内核 ACPI 驱动直写）、`acpi_call`（通过第三方模块调用任意 ACPI 方法）、`sysfs`（上述标准接口）。三种在这台机器上全部缺席：Redmi 没有为自家 EC 提交过任何内核驱动，`charge_control_end_threshold` 文件不存在，`acpi_call` 也没有安装。

TLP 的处理方式是静默降级——配置留着，什么都不做。这解释了"设置了却充到顶"的全部现象。

## 从 DSDT 考古：找到 EC 里的充电上限字段

内核不提供接口，就要自己读固件。ACPI 的 DSDT 表是主板固件的执行代码，用 `iasl` 反汇编后就是一份可检索的"硬件说明书"：

```bash
cp /sys/firmware/acpi/tables/DSDT /tmp/probe/
iasl -d /tmp/probe/DSDT.bin
```

在反汇编产物里，`Device (EC0)` 是嵌入式控制器（Embedded Controller，主板上负责电池、键盘、风扇等底层事务的 8 位单片机）。EC 暴露给 ACPI 的字段表以 `Field (ERAM…)` 开头，逐项列出每个字段的名称和位宽。搜索电池相关的命名，一段关键结构浮现出来：

```
Offset (0x80),
ACIN,   1,    // AC 已插电
BTIN,   1,    // 电池在位
BTST,   4,    // 电池状态
ADPW,   8,
BTSN,   16,   // 序列号
BTDC,   16,   // Design Capacity 设计容量
BTDV,   16,   // Design Voltage 设计电压
BTFC,   16,   // Full Charge Capacity 满充容量
BTTP,   16,
BTCT,   16,   // Cycle Count 循环次数
BTPR,   16,   // Present Capacity 当前容量
BTVT,   16,   // Voltage 电压
RSOC,   8,    // Relative State of Charge 电量百分比
…                // 8 个状态位占满一个字节
BTCC,   16,   // Charge Cap 充电上限
BATM,   16,
```

按 ACPI Field 的位累积规则手工推算，每个字段落在 EC RAM 的具体字节位置：电量百分比 RSOC 在偏移 0x92，充电上限 BTCC 在 0x95-0x96（16 位小端）。这台机器的 EC RAM 通过内存映射窗口 `0xFE0B0400` 可直接访问。

推断需要验证。验证方法是交叉比对：把 32 个字节一次性读出来，看各项读数能否和系统已知的数值对上。实际读取结果：

| EC 字段 | 读数 | 系统对照值 | 结果 |
|---|---|---|---|
| RSOC | 89% | upower 实时 89%（自放电后的实时值） | 命中 |
| BTDC | 4664 mAh × 15440 mV = 72.0 Wh | energy_full_design 72.012 Wh | 命中 |
| BTFC | 3939 mAh → 60.8 Wh | energy_full 60.818 Wh | 命中 |
| BTFC/BTDC | 84.5% | 电池健康度 84.455% | 命中 |
| BTPR/BTFC | 88.9% | ≈ RSOC 89%，内部自洽 | 命中 |

六项全部命中，偏移表确认无误。这次读取还带来一个意外收获：**循环次数 1196 次**——内核 sysfs 一直报 0，真实数字一直躺在 EC 里。84.5% 的健康度对一千二百次循环来说衰减曲线相当正常，而 80% 上限从现在开始会显著放缓后续衰减。

验证时 BTCC 读数为 0，即从未有系统写入过它——TLP 静默失效、小米自己的管理软件从未在 Linux 侧运行，这个字段空了两年。

## 写入与固化：从一次性验证到开机自启

写入本身是对内存映射窗口的 2 字节操作。用 `dd` 定位到 `0xFE0B0495`，写入小端编码的 80，读回确认 EC 接受。行为验证更直接：写入时电量 87%，高于上限，状态立即表现为 Discharging（拒绝充电）；电量自然回落到 83% 后，状态转为 Charging——EC 正在按"低于 80 才充、充到 80 停"的逻辑工作。

EC 的值在冷启动后可能复位，所以最后一步是固化：一个 20 行的 shell 脚本（写入 + 最多 5 次读回重试 + 写 syslog），注册成 oneshot 的 systemd 服务，`WantedBy=multi-user.target`。每次开机自动重写。日志验证一行即可：`journalctl -t ec-charge-limit`。

## 为什么绕了这么多路

这套探索的完整链条值得复盘，因为它演示了一个通用模式：**当厂商不为 Linux 提供驱动时，固件代码是最后的、也是完全可靠的文档**。

TLP 配置存在但从未生效，说明用户态工具依赖于内核 ABI 的覆盖面。内核接口缺失时，`acpi_call` 这类"万能调用器"是第一层补救——但它的前提仍是 ACPI 层存在可调用的方法。DSDT 反汇编显示这台机器的 EC 确实声明了 BTCC 字段，但没有任何 AML 代码（ACPI 层的解释执行代码）读写它——小米把充电控制做成了纯 EC 寄存器协议，只有 Windows 侧的私有驱动知道怎么写。这一层是 acpi_call 也救不了的，于是只剩最后一条路：直接读写 EC RAM。

这已经触及了此类探索的边界。直接写 EC 是有真实风险的操作——这段内存窗口同时被固件本身使用，写错偏移可能影响风扇、键盘背光乃至供电策略。整个过程中风险控制靠三件事：所有写入前先完成只读交叉验证（六个字段全命中才敢写）；写入目标收窄到 2 字节；写入后立即读回并观察行为。至于回滚，同样的通道写 100 就能恢复原状。

## 交付清单

改造后的最终状态：

- 拔电、合盖均不休眠，suspend 目标在 systemd 层全局禁用
- 充电到 80% 由 EC 强制执行，不依赖任何用户态进程存活
- 开机自动重写上限，冷启动不丢配置
- 电池档案：设计 72 Wh，满充 60.8 Wh（健康度 84.5%），循环 1196 次

完整脚本已归档：`setup-server-power.sh`（休眠部分）、`acpi-probe-battery.sh`（DSDT 反汇编 + 只读探测）、`ec-verify-readonly.sh`（字段交叉验证）、`ec-set-charge-limit.sh`（写入与行为观察）、`power-rollback.sh`（一键回滚）。如果你有一台同样"EC 沉默"的笔记本，这套只读验证优先的流程可以直接复用——先证明地址表是对的，再谈写入。
