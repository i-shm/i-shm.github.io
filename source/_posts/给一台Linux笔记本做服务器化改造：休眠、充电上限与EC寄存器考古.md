title: 给一台 Linux 笔记本做服务器化改造：休眠、充电上限与 EC 寄存器考古
date: 2026-08-29 15:30:00
tags: [Technique, Linux]
---

一台退役笔记本装载 Hermes Agent 后转为 7×24 常驻服务器，笔记本的默认行为与服务器角色有三处冲突：电源管理在拔电半小时后自动休眠、合盖即失联、充电不设上限。这些是厂商为移动场景设计的合理默认值，对服务器却是故障。本文记录改造全程。前两项属常规配置；第三项在 Linux 下无现成驱动，需要从主板固件代码中定位充电上限的寄存器地址并写入硬件，构成本文主体。

改造对象为 Redmi Book Pro 15 2022，系统 Pop!_OS 24.04，内核 6.18.7。所有脚本完整源码收录于正文。

## 自动休眠由两套机制叠加控制，仅修改其一必然复发

第一处位于 GNOME 会话。`org.gnome.settings-daemon.plugins.power` 键值组区分供电与电池两种策略，该机出厂默认前者为 `nothing`（不休眠）、后者为 `suspend`（30 分钟无操作休眠）。服务器拔电即失联，直接原因即电池策略。修改两条键值即可禁用：

```bash
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-type 'nothing'
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
```

第二处位于 systemd-logind。不处理它，合盖仍触发 suspend。向 `/etc/systemd/logind.conf.d/99-server-nosuspend.conf` 写入 drop-in 配置：

```ini
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
AllowSuspend=no
AllowHibernation=no
```

`AllowSuspend=no` 与 `AllowHibernation=no` 使 suspend、hibernate 两个 systemd 目标在系统层不可达：任何组件发起休眠请求，systemd 直接拒绝执行。这一层保证无论用户会话如何变更，服务器不会被睡掉。

上述配置对应脚本 `setup-server-power.sh`，其主体即本节三条命令加上 logind 重启，此处不再展开。

## 充电上限配置存在两年，从未生效

机器长期安装 TLP，配置如下：

```
START_CHARGE_THRESH_BAT0=20
STOP_CHARGE_THRESH_BAT0=80
```

电量仍充至 94%（衰减后的实际满充点）。TLP 自身不操作硬件，仅调用内核接口。内核标准接口为 `/sys/class/power_supply/BAT0/charge_control_end_threshold`；TLP 依次探测三种后端：`natacpi`（内核 ACPI 驱动直写）、`acpi_call`（第三方模块调用任意 ACPI 方法）、`sysfs`（上述标准接口）。三种后端在本机全部缺失：Redmi 未向内核提交任何 EC 驱动，`charge_control_end_threshold` 不存在，`acpi_call` 未安装。

TLP 对后端缺失的处理是静默降级：保留配置，不执行任何操作。此即"设置了上限却充到满"的完整成因。补装 `acpi-call-dkms` 后重启 TLP，`tlp-stat` 报告 `natacpi (system76_acpi) = inactive (laptop not supported)`，阈值接口依旧未出现——ACPI 层无方法可调，此路不通。

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

偏移推算须经读取验证。验证脚本用 `dd` 从 `/dev/mem` 读出窗口内 32 字节，再按偏移表逐字段解码：

```bash
#!/bin/bash
# ec-verify-readonly.sh — 只读验证偏移表，不做任何写入
[ "$(id -u)" -eq 0 ] || { echo "请用 sudo 运行"; exit 1; }
dd if=/dev/mem bs=1 skip=$((0xFE0B0480)) count=32 of=/tmp/ec-read.bin status=none
python3 - <<'PYEOF'
data = open("/tmp/ec-read.bin","rb").read()
def u8(o):  return data[o]
def u16(o): return data[o] | (data[o+1] << 8)
b80 = data[0]
print(f"ACIN(AC插电)  = {b80 & 1}     (期望 1)")
print(f"BTDC 0x84     = {u16(4)} mAh   (对照 energy_full_design/电压)")
print(f"BTDV 0x86     = {u16(6)} mV    (期望约 15440)")
print(f"BTFC 0x88     = {u16(8)} mAh   (对照 energy_full)")
print(f"BTPR 0x8E     = {u16(14)} mAh  (对照当前容量)")
print(f"BTVT 0x90     = {u16(16)} mV   (期望 16000-16800)")
print(f"RSOC 0x92     = {u8(18)}%     (对照 upower 实时值)")
print(f"BTCC 0x95     = {u16(21)}%     (充电上限，验证目标)")
print(f"BTCT 0x8C     = {u16(12)} 次   (循环次数)")
PYEOF
```

一次读出 32 字节，与系统已知值交叉比对：

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

写入是针对内存映射窗口的 2 字节操作。写入时电量 87%，高于上限，电池状态立即转为 Discharging；电量自然回落至 83% 后状态转为 Charging。EC 严格执行"低于 80 充电、达到 80 停充"。

EC 值可能在冷启动后复位，需开机重写。方案为 oneshot 型 systemd 服务，执行 18 行运行时脚本 `/usr/local/bin/ec-charge-limit`——这是全套配置中唯一长期驻留的代码，全文如下：

```bash
#!/bin/bash
# ec-charge-limit — BTCC @ 0xFE0B0495 (little-endian u16)
# 用法: ec-charge-limit [20-100]，由 ec-charge-limit.service 开机调用
LIMIT=${1:-80}
[ "$LIMIT" -ge 20 ] && [ "$LIMIT" -le 100 ] || exit 1
BASE=$((0xFE0B0400)); OFF=$((0x95))
[ "$(id -u)" -eq 0 ] || exit 1
for i in 1 2 3 4 5; do
    printf "$(printf '\\x%02x\\x00' "$LIMIT")" \
        | dd of=/dev/mem bs=1 seek=$((BASE + OFF)) count=2 conv=notrunc status=none \
        || { sleep 2; continue; }
    sleep 1
    dd if=/dev/mem bs=1 skip=$((BASE + OFF)) count=2 of=/tmp/.btcc-check status=none
    V=$(python3 -c "d=open('/tmp/.btcc-check','rb').read(); print(d[0]|(d[1]<<8))")
    [ "$V" = "$LIMIT" ] && { logger -t ec-charge-limit "BTCC set to $LIMIT (attempt $i)"; exit 0; }
    sleep 2
done
logger -t ec-charge-limit "FAILED to set BTCC=$LIMIT after 5 attempts"
exit 1
```

脚本包含三重保护：参数限制在 20-100；写入后读回比对，不匹配则重试；至多 5 次，失败写 syslog 并以非零码退出。服务单元 5 行：

```ini
[Unit]
Description=Set Redmi Book Pro charge limit to 80% via EC memory window
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/ec-charge-limit 80
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

部署后验证：`journalctl -t ec-charge-limit` 应含 `BTCC set to 80 (attempt 1)`。

## 方法论：固件代码是最后的文档

TLP 配置存在而从未生效，根源在于用户态工具依赖内核 ABI 的覆盖面。内核接口缺失时，`acpi_call` 一类通用调用器是第一层补救，但其前提仍是 ACPI 层存在可调用的方法。DSDT 反汇编显示本机 EC 声明了 BTCC 字段，而全部 AML 代码均未读写它：小米将充电控制实现为纯 EC 寄存器协议，仅 Windows 侧私有驱动知晓协议细节。这一层 acpi_call 无法覆盖，剩余路径只有直接读写 EC RAM。

直接写 EC 有真实风险：该内存窗口同时被固件自身使用，写错偏移可能影响风扇、键盘背光乃至供电策略。本次风险控制依赖三项措施：写入前完成只读交叉验证，五项指标全部命中后才执行写入；写入范围收窄至 2 字节；写入后立即读回并观察充电行为。回滚与写入同路：向同一地址写 100 即恢复满充。

## 改造结果

- 拔电、合盖均不休眠；suspend 目标在 systemd 层全局禁用
- 充电上限 80% 由 EC 强制执行，不依赖任何用户态进程存活
- 开机自动重写上限，冷启动不丢配置
- 电池档案：设计 72 Wh，满充 60.8 Wh（健康度 84.5%），循环 1196 次

辅助脚本归档于本地（篇幅所限未全文收录）：`setup-server-power.sh`（休眠禁用，含备份与逐步回显）、`acpi-probe-battery.sh`（DSDT 反汇编与 ACPI 只读探测）、`ec-set-charge-limit.sh`（首次写入与 60 秒行为观察）、`power-rollback.sh`（一键回滚）。其他"EC 沉默"的机型可直接复用核心流程：反汇编 DSDT 检索字段表 → 手工推算偏移 → 只读交叉验证 → 验证命中后再写入。
