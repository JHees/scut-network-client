# 树莓派网络维护记录

记录日期：2026-10-19。设备：JHees-rpi，Raspberry Pi 4B，Ubuntu 22.04.1 LTS。

## scutclient 与旧网络设置的关系

本地项目位于 `/home/jhees/scutclient`，检查时 HEAD 为 `7d579cb`。它包含 Dr.com 有线校园网认证程序，以及一个无线门户认证程序。

- `README.md` 和 `src/main.cpp`：有线认证、在线及离线钩子参数。
- `wireless/main.cpp:11`：连接 Wi-Fi、关闭有线接口和切换路由仍列在 TODO 中。
- `wireless/wireless.h:74`：`addRouteRable()` 只解析并打印 IP，未实际添加路由。因此不能据此认定已实现自动有线／无线切换。
- `CMakeLists.txt`：分别构建 `DrcomScutClient` 和 `wireless`。
- `DrcomScutClient.service`：已安装的服务原先开机启动，依赖 `dhcpcd.service`；旧服务定义含认证凭据，不能直接发布。
- `DrcomScutClient.timer`：检查时未启用，且指向 `drcom8021x.service`，与本项目服务名不一致。
- AP 由独立的 `create_ap`、`hostapd`、`dnsmasq`、`radvd` 服务和系统配置管理，没有在已检查的项目代码中发现统一的 AP 管理实现。

项目源码保留。后续整理将服务中的硬编码凭据迁入私有备份，公开模板改为读取 `/etc/scutclient.env`；同时修正定时器目标。

## 旧配置与此次问题

旧有线配置为 `10.40.68.183/23`，默认网关 `10.40.69.254`。旧 AP 使用 USB 无线接口 `wlxe0e1a9146d51`，热点名 `JHees-rpi`，网段 `172.18.1.0/24`，并启用了 IPv4／IPv6 转发。该 USB 网卡当前不存在。

初次检查时，实验室 SSID 被写成 `Acoustiv filter lab`；后来已改为 `Acoustic filter lab` 并成功连接。旧默认路由仍抢占出口；`systemd-resolved` 被屏蔽，`/etc/resolv.conf` 是失效链接。初次检查时间停在 2022 年，后已恢复同步。

## 首次修复后的配置（历史记录）

| 项目 | 配置 |
|---|---|
| 网线 | `eth0` 固定 `192.168.50.2/24`，无默认网关、不接受 IPv6 RA |
| Wi-Fi | `Acoustic filter lab`，DHCP 获取地址；检查时为 `192.168.4.102` |
| 默认出口 | Wi-Fi，网关 `192.168.4.1`；配置优先级 100，清理后运行时保留的唯一默认路由优先级为 303 |
| 网络管理 | Netplan → systemd-networkd；主文件 `/etc/netplan/90-lab.yaml` |
| DNS | systemd-resolved；使用 Wi-Fi DHCP 提供的 DNS |
| 时间 | systemd-timesyncd，Asia/Shanghai，已验证 NTP 同步 |
| cloud-init | 禁止重新生成网络配置 |
| 转发 | IPv4／IPv6 转发关闭 |

有线连接电脑时，电脑侧也需设置同网段地址，例如 `192.168.50.1/24`，该接口无需默认网关。网线未连接时，networkd 可能移除接口运行时地址；重新插线后按持久配置配置地址。2026-10-19 曾在有线链路正常时验证 `ssh jhees@192.168.50.2`，拔线后改用 Wi-Fi 地址继续维护。

## 归档与清理

完整结果见 `network-backups/cleanup-result.json`；该文件只在清理与验证成功后写入。

- `network-backups/legacy/network-backup-*/`：先前 `/root/network-backup-*` 的完整副本，包含修复前配置归档。迁移逐文件核对 SHA-256 和符号链接目标后，才删除原目录。
- `network-backups/current/network-config.tar`：本次集中归档时的网络配置，包括凭据；tar 内保留原文件权限和链接。
- `network-backups/current/services-before.txt`、`services-after.txt`：清理前后相关服务的启用及运行状态。
- `network-backups/SHA256SUMS`：集中备份的文件校验值。
- `network-backups/maintenance-scripts/repair-network-20261019.py`：保留上一次修复脚本作历史参考。
- `network-backups/maintenance-scripts/archive-network-cleanup.py`：本次归档和清理脚本。

`network-backups/` 权限为 700，普通文件为 600，并已加入 `.gitignore`。备份包含 Wi-Fi 和校园网认证凭据，不应提交到 Git 或公开分享。

本次清理停用旧 Drcom 认证服务及其定时器，防止它在重启时通过 Wants 依赖重新拉起 dhcpcd；源码与原服务定义仍保留。旧 AP 软件及防火墙规则留作历史配置，相关服务、自动启动和转发已经关闭。

ZeroTier 已卸载，并已删除 `/var/lib/zerotier-one` 中的本机身份、成员配置和已确认的旧 moon 文件。若以后重新安装，会创建新身份，需重新加入网络。不会注销或修改远端 ZeroTier 管理控制台中的成员记录。

“其他备份”仅清理本次网络维护产生且已核对迁移的 `/root/network-backup-*`，不删除其他项目、个人文件或系统备份。旧修复脚本归档到本地备份后删除家目录中的重复副本。

## 检查与恢复

日常检查：

```bash
ip -br addr
ip route
resolvectl status wlan0
timedatectl
curl -I https://example.com
systemctl is-active systemd-networkd systemd-resolved systemd-timesyncd
```

检查备份完整性：

```bash
cd /home/jhees/scutclient/network-backups
sha256sum -c SHA256SUMS
```

恢复前先将 tar 解包到临时目录进行比较，不要直接覆盖 `/etc`。当前配置归档中的 Netplan、DNS 和 sysctl 配置可用于恢复当前网络；恢复旧 AP 时需要原 USB 网卡、相应服务以及对上游网络重新验证。旧归档中的恢复脚本可能仍含迁移前的绝对路径，不应直接运行。

本次没有重装系统、升级内核或实现新的自动切换功能，也未通过重启测试确认整机启动过程。最终验证状态以 `cleanup-result.json` 和实际网卡状态为准。


## 首次清理验证（2026-10-19 19:29–19:30 CST）

- 已迁移 `/root/network-backup-20261019-191959`，迁移后逐项校验通过，原目录已删除。
- 集中备份的 18 个文件通过 SHA-256 复核；目录与文件均无组用户或其他用户访问权限。
- ZeroTier 软件包、虚拟网卡、本机数据目录和旧 moon 文件已删除。
- DrcomScutClient 服务、定时器、dhcpcd、create_ap、hostapd、dnsmasq、radvd 和 wide-dhcpv6-client 均为 inactive / disabled。
- systemd 没有失败服务；此前 wide-dhcpv6-client 的历史失败标记已清除。
- Wi-Fi 地址 `192.168.4.102`，唯一默认出口为 `wlan0`；DNS 正常，HTTPS 返回 200，NTP 已同步。
- 有线目前没有物理链路，`192.168.50.2/24` 的持久配置仍在；此次未重新插线或重启复测。
- 项目原有 `DrcomScutClient.service` 修改保持原样。新增文档、工具与忽略规则未提交 Git。

## 后续系统维护

按机主要求，后续维护移除 `192.168.50.2/24` 网线直连配置，保留实验室 Wi-Fi DHCP 联网。
Ubuntu 继续使用 22.04（jammy），通过清华 Ubuntu Ports 的 jammy、updates、backports、security 仓库更新，不执行发行版升级或系统重装。
ZeroTier 软件源删除，已停用的 hostapd、dnsmasq、radvd、dhcpcd5、wide-dhcpv6-client 软件包清除；个人代码目录保留。

Oh My Zsh 的失效镜像地址已改为 `https://github.com/ohmyzsh/ohmyzsh.git`。由于树莓派直连 GitHub 超时，更新由另一台电脑从官方仓库下载后以 Git bundle 中转，保持完整 Git 历史并仅执行快进合并。更新提交为 `964ddf78`；用户 Git 配置中把 GitHub 自动改写为失效镜像的 insteadOf 规则也已移除；未来在线更新仍取决于树莓派到 GitHub 的网络连通性。

`tools/update-system.py` 是这台设备的一次性维护记录：更新前配置与更新结果放在被 Git 忽略的 `network-backups/system-update-*/`，其中 `postboot.json` 记录重启后 Wi-Fi、DNS、HTTPS 和时间同步的验证结果。脚本会自动重启，不应在其他设备直接运行。

GitHub 上传不包含 `network-backups/`、构建产物、真实环境文件或认证日志。服务模板使用 `scutclient.env.example`，账号密码应写入仓库外的 `/etc/scutclient.env`（root 所有，权限 600）。
