title: 给一台 Linux 笔记本做服务器化改造：休眠、充电上限与 EC 寄存器考古
date: 2026-08-29 15:30:00
tags: [Technique, Linux]
---

一台退役笔记本装载 Hermes Agent 后转为 7×24 常驻服务器，出厂的笔记本形态留下三类故障：拔电半小时后自动休眠、合盖即失联、电池持续充至满电。本文记录改造全程。前两项属常规配置；第三项在 Linux 下无现成驱动，需要从主板固件代码中定位充电上限的寄存器地址并写入硬件，构成本文主体。

改造对象为 Redmi Book Pro 15 2022，系统 Pop!_OS 24.04，内核 6.18.7。关键步骤均附可复核证据，脚本清单见文末。

## 自动休眠由两套机制叠加控制，仅修改其一必然复发

第一处位于 GNOME 会话。`org.gnome.settings-daemon.plugins.power` 键值组区分供电与电池两种策略，该机出厂默认前者为 `nothing`（不休眠）、后者为 `suspend`（30 分钟无操作休眠）。服务器拔电即失联，直接原因即电池策略。修改两条键值即可禁用：

```bash
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-type 'nothing'
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
```

第二处位于 systemd-logind。不处理它，合盖仍触发 suspend。向 `/etc/systemd/logind.conf.d/` 写入 drop-in 配置：

```ini
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
AllowSuspend=no
AllowHibernation=no
```

`AllowSuspend=no` 与 `AllowHibernation=no` 使 suspend、hibernate 两个 systemd 目标在系统层不可达：任何组件发起休眠请求，systemd 直接拒绝执行。这一层保证无论用户会话如何变更，服务器不会被睡掉。

## 充电上限配置存在两年，从未生效

机器长期安装 TLP，配置如下：

```
START_CHARGE_THRESH_BAT0=20
STOP_CHARGE_THRESH_BAT0=80
```

电量仍充至 94%（衰减后的实际满充点）。TLP 自身不操作硬件，仅调用内核接口。内核标准接口为 `/sys/class/power_supply/BAT0/charge_control_end_threshold`；TLP 依次探测三种后端：`natacpi`（内核 ACPI 驱动直写）、`acpi_call`（第三方模块调用任意 ACPI 方法）、`sysfs`（上述标准接口）。三种后端在本机全部缺失：Redmi 未向内核提交任何 EC 驱动，`charge_control_end_threshold` 不存在，`acpi_call` 未安装。

TLP 对后端缺失的处理是静默降级：保留配置，不执行任何操作。此即"设置了上限却充到满"的完整成因。

## 从 DSDT 定位 EC 充电上限字段

内核不提供接口时，固件代码是唯一可靠的文档。ACPI 的 DSDT 表是主板固件的 AML 执行代码，用 `iasl` 反汇编后可检索：

```bash
cp /sys/firmware/acpi/tables/DSDT /tmp/probe/
iasl -d /tmp/probe/DSDT.bin
```

反汇编产物中，`Device (EC0)` 即嵌入式控制器（Embedded Controller，主板上管理电池、键盘、风扇等底层事务的 8 位单片机）。EC 向 ACPI 暴露的字段表以 `Field (ERAM…)` 开头，逐项列出字段名与位宽。检索电池相关命名，得到关键结构：

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

依据 ACPI Field 的位累积规则逐字段推算，各字段在 EC RAM 中的字节位置随之确定：电量百分比 RSOC 位于偏移 0x92，充电上限 BTCC 位于 0x95-0x96（16 位小端）。本机 EC RAM 经内存映射窗口 `0xFE0B0400` 可直接访问。

偏移推算须经读取验证。一次读出 32 字节，与系统已知值交叉比对：

| EC 字段 | 读数 | 系统对照值 | 结果 |
|---|---|---|---|
| RSOC | 89% | upower 实时 89% | 命中 |
| BTDC | 4664 mAh × 15440 mV = 72.0 Wh | energy_full_design 72.012 Wh | 命中 |
| BTFC | 3939 mAh → 60.8 Wh | energy_full 60.818 Wh | 命中 |
| BTFC/BTDC | 84.5% | 电池健康度 84.455% | 命中 |
| BTPR/BTFC | 88.9% | 与 RSOC 89% 自洽 | 命中 |

五项独立指标全部命中，偏移表成立。读取同时得到一项内核长期隐瞒的数据：循环次数 1196 次——sysfs 的 `cycle_count` 一直报 0，真实值存于 EC。84.5% 的健康度对应一千二百次循环，衰减幅度在正常区间；80% 上限将显著延缓后续衰减。

首次读取时 BTCC 为 0：该字段从未被任何系统写入过。TLP 静默失效，小米管理软件从未在 Linux 侧运行，此字段空置两年。

## 写入与开机固化

写入是针对内存映射窗口的 2 字节操作：`dd` 定位 `0xFE0B0495`，写入小端编码的 80，读回确认。行为验证给出更强的证据：写入时电量 87%，高于上限，电池状态立即转为 Discharging；电量自然回落至 83% 后状态转为 Charging。EC 严格执行"低于 80 充电、达到 80 停充"。

EC 值可能在冷启动后复位，需开机重写。方案为 oneshot 型 systemd 服务（`WantedBy=multi-user.target`），执行 20 行 shell 脚本：写入、至多 5 次读回重试、结果写 syslog。验证一行命令：`journalctl -t ec-charge-limit`。

## 方法论：固件代码是最后的文档

TLP 配置存在而从未生效，根源在于用户态工具依赖内核 ABI 的覆盖面。内核接口缺失时，`acpi_call` 一类通用调用器是第一层补救，但其前提仍是 ACPI 层存在可调用的方法。DSDT 反汇编显示本机 EC 声明了 BTCC 字段，而全部 AML 代码均未读写它：小米将充电控制实现为纯 EC 寄存器协议，仅 Windows 侧私有驱动知晓协议细节。这一层 acpi_call 无法覆盖，剩余路径只有直接读写 EC RAM。

直接写 EC 有真实风险：该内存窗口同时被固件自身使用，写错偏移可能影响风扇、键盘背光乃至供电策略。本次风险控制依赖三项措施：写入前完成只读交叉验证，五项指标全部命中后才执行写入；写入范围收窄至 2 字节；写入后立即读回并观察充电行为。回滚与写入同路：向同一地址写 100 即恢复满充。

## 改造结果

- 拔电、合盖均不休眠；suspend 目标在 systemd 层全局禁用
- 充电上限 80% 由 EC 强制执行，不依赖任何用户态进程存活
- 开机自动重写上限，冷启动不丢配置
- 电池档案：设计 72 Wh，满充 60.8 Wh（健康度 84.5%），循环 1196 次

脚本归档：`setup-server-power.sh`（休眠禁用）、`acpi-probe-battery.sh`（DSDT 反汇编与只读探测）、`ec-verify-readonly.sh`（字段交叉验证）、`ec-set-charge-limit.sh`（写入与行为观察）、`power-rollback.sh`（一键回滚）。其他"EC 沉默"的机型可直接复用该流程：先以只读交叉验证证明地址表，再执行写入。
