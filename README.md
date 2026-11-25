# loongfire-wan-fix

龙芯 `loongfire`（IPFire 派生）防火墙 WAN 口修复记录：

- 开机后 WAN 口拿不到 IP、没有默认路由；
- Web 管理页 `zoneconf.cgi` 的 RED 区域只显示“- 无 -”，没有 `Native`；
- 根因是 WAN 网卡没有永久 MAC，内核每次启动随机生成，导致按 MAC 的接口命名和区域绑定全部失效。

> **脱敏说明**：本文中的 IP 地址、MAC 地址、主机名、PCI 地址、账号与口令均为占位符或文档示例地址，已替换真实环境信息。实际操作时请替换成自己的值。

## 环境

| 项目 | 说明 |
| --- | --- |
| 平台 | loongarch64，内核版本带 `-loongfire` 后缀 |
| 系统 | loongfire（IPFire 派生，配置在 `/var/ipfire/`） |
| LAN | 板载 Loongson 千兆网卡，逻辑名 `green0` |
| WAN | PCIe Realtek RTL8125 2.5GbE，期望逻辑名 `red0` |
| WAN 模式 | DHCP |

## 症状

1. 重启后 `red0` 不存在，网卡仍叫 `eth0`；`ip route` 没有默认路由，`dhcpcd` 没在跑。
2. `zoneconf.cgi` 里 WAN 网卡对应的 RED 列是“- 无 -”，而 LAN 网卡的 GREEN 列是 `Native`。
3. 手动执行一次启动 WAN 的服务后可以临时拿到 IP，但下次重启又失效。

## 排查过程

### 1. 确认 WAN 没有起来

```bash
ip -br link                 # 看到 eth0 DOWN，没有 red0
ip -br addr                 # eth0 无地址
ip route                    # 只有 LAN 网段路由，没有 default
ps -ef | grep dhcpcd        # 没有 WAN 的 dhcpcd 进程
```

查看 IPFire 的以太网配置：

```bash
cat /var/ipfire/ethernet/settings
```

关键项应类似：

```text
RED_TYPE=DHCP
RED_DEV=red0
RED_MACADDR=<配置里记录的固定 MAC>
GREEN_DEV=green0
```

也就是说：服务期望的接口名是 `red0`，但系统里只有 `eth0`。

### 2. 根因：WAN 网卡没有永久 MAC

```bash
ethtool -P eth0
# Permanent address: not set
```

`ethtool -P` 读不到永久地址，说明这块网卡的 EEPROM 里没有烧录 MAC。内核每次启动会随机生成一个本地管理地址：

```bash
cat /sys/class/net/eth0/address
# 每次重启都不一样，例如 02:xx:xx:xx:xx:xx / 12:xx:xx:xx:xx:xx ...
```

于是出现连锁反应：

1. 如果 udev 规则按 MAC 匹配（例如 `ATTR{address}=="<某次 MAC>"`），重启后 MAC 变了，规则不再命中；
2. 网卡没有被命名成 `red0`，仍是 `eth0`；
3. IPFire 随后用 `RED_DEV=red0` 启动 WAN 服务，找不到接口，DHCP 客户端不启动，WAN 没有 IP；
4. Web 页面也按 MAC 判断区域归属，见下一步。

### 3. 为什么 RED 列没有 Native

IPFire 的 `/srv/web/ipfire/cgi-bin/zoneconf.cgi` 逻辑大致是：

```perl
# 只有当网卡实际 MAC 等于该区域的 *_MACADDR 配置时，才勾选 Native
if ($mac eq $ethsettings{"${uc}_MACADDR"}) {
    $access_selected{"NATIVE"} = "selected";
}
```

所以 `RED_MACADDR` 与 WAN 网卡当前 MAC 不一致时，RED 列就不会显示 `Native`，而是 `- 无 -`。

## 修复

### 步骤 1：按 PCI 槽位把 WAN 命名为 red0（不要按 MAC）

先找到 WAN 网卡的 PCI 地址。修复前接口叫 `eth0`：

```bash
readlink -f /sys/class/net/eth0/device
# 例：/sys/devices/pci0000:00/0000:00:0b.0/0000:03:00.0

udevadm info -a -p /sys/class/net/eth0 | grep -E 'KERNELS|DRIVERS'
# 例：KERNELS=="0000:03:00.0"
```

新建 udev 规则，**按 PCI 槽位匹配**，不要按 MAC：

```bash
cat > /etc/udev/rules.d/70-red-wan.rules <<'EOF'
SUBSYSTEM=="net", ACTION=="add", KERNELS=="<PCI_BDF>", NAME="red0"
EOF
```

将 `<PCI_BDF>` 换成上一步查到的值，例如 `0000:03:00.0`。

> 注意：不要写 `ATTR{address}="<MAC>"` 来固定 MAC——在该 RTL8125 + r8169 驱动组合上，这个属性在 udev 的 add 阶段不生效。

### 步骤 2：启动时把 MAC 固定为配置值

因为 `zoneconf.cgi` 按 MAC 判断 Native，另外还需要一个启动脚本，把 `red0` 的 MAC 设置成 `/var/ipfire/ethernet/settings` 里的 `RED_MACADDR`：

```bash
cat > /etc/rc.d/rcsysinit.d/S84fixwanmac <<'EOF'
#!/bin/bash
# Pin WAN MAC so zoneconf RED stays Native.
eval $(/usr/local/bin/readhash /var/ipfire/ethernet/settings)

[ -z "$RED_DEV" ] && exit 0
[ -d "/sys/class/net/$RED_DEV" ] || exit 0

if [ -n "$RED_MACADDR" ]; then
    ip link set dev "$RED_DEV" down
    ip link set dev "$RED_DEV" address "$RED_MACADDR"
    ip link set dev "$RED_DEV" up
fi
EOF
chmod +x /etc/rc.d/rcsysinit.d/S84fixwanmac
```

脚本名用 `S84...`，保证它在 `S85firewall` 和 rc3 的网络服务之前执行。

### 步骤 3：立即生效（不用等重启）

需要先把当前接口改名为 `red0`（仅本次会话；下次启动由 udev 规则完成）：

```bash
ip link set eth0 down
ip link set eth0 name red0
ip link set red0 up

/etc/rc.d/rcsysinit.d/S84fixwanmac
/etc/init.d/networking/red restart
```

如果重启服务时提示 `dhcpcd already running`，通常不影响结果，按下一步核对地址和路由即可。必要时再执行一次 `/etc/init.d/networking/red start`。

### 步骤 4：验证

```bash
ip -br link show red0
cat /sys/class/net/red0/address        # 应等于 RED_MACADDR，且重启后不变
ip -br addr show red0                  # 应有 DHCP 地址
ip route show default                  # default via <网关> dev red0
ping -c 2 <公网地址>
iptables -t nat -L -n -v | grep -i masq # out red0 的 MASQUERADE 规则
```

再打开 Web 管理页 `zoneconf.cgi`，确认：

- WAN 网卡的 RED = `Native`，GREEN = `- 无 -`；
- LAN 网卡的 GREEN = `Native`，RED = `- 无 -`。

最后重启一次防火墙，重复上述检查，确认全部持久生效。

## 附：工作站双网口优先级（配套经验）

如果本机同时有无线和有线两条链路，可用 NetworkManager 的路由 metric 决定优先级：**数值越小越优先**。

```bash
# 所有 WiFi profile 用低 metric
nmcli -t -f NAME,TYPE connection show | while IFS=: read -r c t; do
  [ "$t" = "802-11-wireless" ] && \
    nmcli connection modify "$c" ipv4.route-metric 50 ipv6.route-metric 50
done

nmcli device reapply wlan0
```

以太网走 NetworkManager 默认值（通常 100 或 200），大于 50，因此无线优先、有线备用。验证：

```bash
ip route show default                 # wlan0 metric 50，enp* metric 更高
ip route get <公网地址>               # 应从 wlan0 出

nmcli device disconnect wlan0         # 模拟无线断开
ip route show default                 # 应切换到有线默认路由
nmcli connection up <无线配置名>      # 恢复，默认路由回到无线
```

注意：如果 WAN 和对端无线在同一个网段、共用同一个网关，只能防本机无线链路故障，防不了上级网关整体故障。

## 常见坑

1. **不要按 MAC 写 udev 规则**：无永久 MAC 的网卡每次启动地址都变。
2. **`ATTR{address}` 不一定生效**：本例中 udev 阶段设置 MAC 无效，改用启动脚本 `ip link set ... address`。
3. **zone 页面按 MAC 判断**：只改接口名还不够，MAC 不对 RED 仍显示 `- 无 -`。
4. **`red restart` 报警**：`dhcpcd already running` 不代表失败，核对地址与路由为准。
5. **改前备份**：`/var/ipfire/ethernet/settings`、原有 udev 规则目录都先备份，便于回滚。

## 回滚

删除本教程新增的两个文件并重启，即可回到原始状态：

```bash
rm -f /etc/udev/rules.d/70-red-wan.rules
rm -f /etc/rc.d/rcsysinit.d/S84fixwanmac
```
