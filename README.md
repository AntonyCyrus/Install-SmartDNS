# Debian 13 VPS 安装 SmartDNS（完善版）

> 基于 [AntonyCyrus/Install-SmartDNS](https://github.com/AntonyCyrus/Install-SmartDNS) 的原教程整理。适用目标：**Debian 13、systemd、amd64 VPS，仅为本机提供 DNS、使用 Cloudflare / Google 的 DoH3 上游**。本教程不要求全面升级 Debian，也不默认锁定 `/etc/resolv.conf`。
>
> **阅读方式：** 第 1–6 步为共同步骤；第 7 步根据检测结果**仅选择适用的分支**；第 8 步验证；故障时按第 10 步中**对应的分支**回滚。

## 功能、前提与限制

- SmartDNS 只监听 `127.0.0.1:53`（UDP / TCP），不是公网递归 DNS 服务器。
- DoH3 上游使用 Cloudflare、Google。配置中的传统 DNS 地址仅用于 DoH3 域名的 bootstrap 解析；**首次建立上游连接仍可能发生明文 DNS 查询**，不要将其误认为全程加密或匿名。
- 默认将 IPv6 `AAAA` 查询处理为 SOA，适用于无需 IPv6 结果的 VPS；若服务器需要 IPv6，请按第 5 步调整。
- 固定下载原教程使用的 `Release48.4`、并核验其给出的 SHA-256。**这是固定版本，不代表始终是最新版**。
- 本地 `.deb` 安装**不会因为添加了这个教程而自动追踪 GitHub Release**。
- 需要 root 权限或 `sudo`。若使用纯净 Debian root 帐户且无 sudo，请将下文命令的 `sudo` 去掉；若是普通帐户，应先以 root 安装并配置 sudo，确认该帐户拥有 sudo 权限。
- 有特殊网络配置（自定义 VPN / Tailscale / 企业内网域名 / 多网卡 DNS / 特定云厂商控制面）时，不能直接照搬“覆盖所有 DNS”的配置。

## 1. 检查系统与 DNS 现状（操作前）

```bash
dpkg --print-architecture
ps -p 1 -o comm=
cat /etc/os-release
```

应确认架构为 `amd64`、PID 1 为 `systemd`、发行版为 Debian 13。若不符合，请勿原样执行固定架构的下载命令。

**先记录原来的 DNS 和配置来源：**

```bash
ls -ld /etc/resolv.conf
readlink /etc/resolv.conf || true
readlink -f /etc/resolv.conf || true
cat /etc/resolv.conf 2>/dev/null || true
systemctl is-active systemd-resolved NetworkManager systemd-networkd 2>/dev/null || true
```

如果 `/etc/resolv.conf` 不存在，请先确认网络管理配置；不要直接假设创建普通文件即可永久生效。

**看懂 `ls -ld` 的结果：**

| 可能的输出（示例） | 判断依据 | 对应第 7 步 |
| --- | --- | --- |
| `-rw-r--r-- ... /etc/resolv.conf` | 首字符为 `-`，普通文件 | **7A**；仍要排查谁在维护它 |
| `lrwxrwxrwx ... /etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf` | 首字符为 `l`，指向 resolved 的 stub | **7B** |
| `lrwxrwxrwx ... /etc/resolv.conf -> /run/systemd/resolve/resolv.conf` | 指向 resolved 的上游服务器列表 | **7B**；注意可能绕过 stub |
| `lrwxrwxrwx ... /etc/resolv.conf -> ../run/resolvconf/resolv.conf` | 指向 resolvconf 生成文件 | **7D** |
| `lrwxrwxrwx ... /etc/resolv.conf -> /run/NetworkManager/resolv.conf` | 指向 NetworkManager 生成文件 | **7C** |
| `ls: cannot access ... No such file or directory` | 不存在；也可能是断链，需要进一步检查 | **7D** |

上表的日期、字节数和箭头后的相对路径只是示例；**以实际链接目标及实际运行的管理程序为准**。`readlink -f` 能显示能解析的最终路径；对于断链，其结果不一定可靠。额外判断：

```bash
if [ -L /etc/resolv.conf ]; then
  echo '符号链接'
elif [ -f /etc/resolv.conf ]; then
  echo '普通文件'
else
  echo '不存在、断链或其他文件类型：请手动排查'
fi
```

**检查本机 53 端口（包含 IPv4、IPv6、UDP、TCP）：**

```bash
sudo ss -lntup '( sport = :53 )'
```

如果有输出，先确认绑定地址和进程：例如 `127.0.0.53:53`（systemd-resolved 的常见 stub）通常**不会与 `127.0.0.1:53` 冲突**；但 `0.0.0.0:53`、`127.0.0.1:53` 或覆盖对应 IPv6 地址的监听可能冲突。不要因为看到 `systemd-resolved` 就直接停用它。若已有 `dnsmasq`、`named`、AdGuard Home 等占用目标端口，先处理冲突再继续。

## 2. 安装必要工具

仅更新包索引、安装依赖，不执行整机升级或清理：

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates wget dnsutils
```

`ss` 通常由 `iproute2` 提供；若系统提示找不到该命令：

```bash
sudo apt-get install -y iproute2
```

## 3. 下载并校验 SmartDNS

沿用原教程固定版本 `Release48.4`：

```bash
cd /tmp
wget -O smartdns.deb 'https://github.com/pymumu/smartdns/releases/download/Release48.4/smartdns.1.2026.08.05-0921.x86_64-debian-all.deb'
```

核验原教程提供的哈希：

```bash
echo '4cc1b8f9db10110e1212d981cfee7eb89a3c80f5766760e3b5f7541c8da8b546  smartdns.deb' | sha256sum -c -
```

只有返回 `smartdns.deb: OK` 才可继续。若为 `FAILED`，立即停止安装，重新核对下载来源、文件名和发布者公布的校验信息。**哈希仅能证明与预期文件一致，不能单独证明软件绝对安全。**

## 4. 安装 SmartDNS

```bash
sudo apt-get install -y /tmp/smartdns.deb
smartdns -v
```

若安装期间服务自动启动但尚未完成配置，稍后第 6 步会重启载入新配置。

## 5. 备份并配置 SmartDNS

```bash
sudo cp -a /etc/smartdns/smartdns.conf /etc/smartdns/smartdns.conf.package-default
sudo tee /etc/smartdns/smartdns.conf >/dev/null <<'EOF'
# 仅向本机提供 DNS 服务。
bind 127.0.0.1:53
bind-tcp 127.0.0.1:53

# 本方案默认不需要 IPv6 AAAA 结果。
force-AAAA-SOA yes
dualstack-ip-selection no

# 缓存和预取。
cache-size 4096
prefetch-domain yes

# 候选地址测速。
speed-check-mode tcp:443,tcp:80,ping

# 仅用于 bootstrap，不作为普通查询的默认上游。
server 1.1.1.1 -bootstrap-dns -exclude-default-group
server 8.8.8.8 -bootstrap-dns -exclude-default-group

# Cloudflare / Google DoH3。
server-h3 h3://cloudflare-dns.com/dns-query
server-h3 h3://dns.google/dns-query

# 暂时失联时使用短时间过期缓存。
serve-expired yes
serve-expired-ttl 3600
serve-expired-reply-ttl 3
EOF
```

> 如果 VPS 需要 IPv6 AAAA 结果，请删去 `force-AAAA-SOA yes` 和 `dualstack-ip-selection no`。DoH3 需要出站 UDP/443；若此端口受限，应先解决上游连接，而不是贸然切换系统 DNS。

## 6. 启动并先测试 SmartDNS

```bash
sudo systemctl enable smartdns
sudo systemctl restart smartdns
sudo systemctl status smartdns --no-pager -l
```

应看到 `Active: active (running)`。在切换默认 DNS **以前**，先执行：

```bash
dig @127.0.0.1 debian.org A +time=3 +tries=2
dig @127.0.0.1 cloudflare.com A +time=3 +tries=2
nslookup -querytype=ptr smartdns 127.0.0.1
sudo ss -lntup '( sport = :53 )'
```

应返回 `NOERROR`、包含真实 IPv4 的答案，且 SmartDNS 仅监听 `127.0.0.1:53`。PTR 查询是辅助诊断，**不能替代真实域名测试**。启动异常时：

```bash
sudo journalctl -u smartdns -n 100 --no-pager
```

**以上测试未通过，停止操作，不要改 `/etc/resolv.conf`。**

## 7. 让本机使用 SmartDNS：选择对应分支

建议远程 VPS 操作时保持现有 SSH 会话不退出，并提前确认云厂商控制台可用。**四个分支只执行相符的一个，不要顺序全做。**

### 7A. 普通文件，且不受 DNS 管理服务自动覆盖

若 DHCP、cloud-init 等服务持续重写这个普通文件，请转到 7D，不要直接套用本分支。

```bash
sudo cp -a /etc/resolv.conf /etc/resolv.conf.before-smartdns
printf 'nameserver 127.0.0.1\noptions timeout:2 attempts:2\n' | sudo tee /etc/resolv.conf >/dev/null
cat /etc/resolv.conf
dig debian.org A +time=3 +tries=2
getent ahostsv4 debian.org
```

如果失败，**立即恢复备份**：

```bash
sudo cp -a /etc/resolv.conf.before-smartdns /etc/resolv.conf
```

不要默认执行 `chattr +i`，普通文件也可能由其他程序维护。

### 7B. systemd-resolved 符号链接

当 `systemd-resolved` 运行，且符号链接指向 `/run/systemd/resolve/...` 时，推荐保留 resolved，让其向 SmartDNS 转发查询。常见 stub `127.0.0.53:53` 与 SmartDNS `127.0.0.1:53` 可以分别监听。

检查：

```bash
systemctl is-active systemd-resolved
resolvectl status
ls -l /etc/resolv.conf
```

使用独立配置文件，避免修改软件包默认文件：

```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
sudo tee /etc/systemd/resolved.conf.d/90-smartdns.conf >/dev/null <<'RESOLVED_EOF'
[Resolve]
DNS=127.0.0.1
Domains=~.
FallbackDNS=
RESOLVED_EOF
sudo systemctl restart systemd-resolved
```

```bash
resolvectl status
resolvectl query debian.org
dig debian.org A +time=3 +tries=2
```

**注意每链路 DNS：** DHCP、NetworkManager 和 systemd-networkd 可能设置独立 DNS 和路由域。全局 `DNS=127.0.0.1` 并不绝对保证所有查询都使用它。仔细检查 `resolvectl status`；若发现其他每链路 DNS 参与解析，应在对应的链路管理器中处理，尤其不要破坏 VPN 或内部域名的 DNS 路由。

如果 `/etc/resolv.conf` 原来指向 `/run/systemd/resolve/resolv.conf`，应用程序可能绕过 stub 而读取上游列表。**仅当确认 `127.0.0.53` stub 可正常解析，且未配置 `DNSStubListener=no` 时**，可将链接切换为 stub 模式：

```bash
sudo cp -a /etc/resolv.conf /etc/resolv.conf.before-smartdns
sudo ln -sfn /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

原本已是 stub 链接则不必执行。改后继续执行第 8 步。

### 7C. NetworkManager 管理 DNS

先查找活动连接与网卡（不要猜连接名）：

```bash
nmcli device status
nmcli -f NAME,UUID,DEVICE connection show --active
```

记录对应连接修改前的设置，用于以后回滚：

```bash
nmcli connection show '连接名称' | grep -E 'ipv4\.dns:|ipv4\.ignore-auto-dns|ipv6\.ignore-auto-dns'
```

把 `连接名称` 替换为实际名称，设置本机 DNS：

```bash
sudo nmcli connection modify '连接名称' ipv4.dns '127.0.0.1' ipv4.ignore-auto-dns yes
```

仅在需要忽略 IPv6 自动 DNS 时，再执行：

```bash
sudo nmcli connection modify '连接名称' ipv6.ignore-auto-dns yes
```

远程 VPS 上**不要轻易 down/up 网络连接**，以免断开 SSH。可尝试 `sudo nmcli device reapply '网卡设备名'`，但仍需按实际设备、NetworkManager 的 DNS 模式及是否接入 resolved 检查生效情况。执行 `nmcli device show`、`cat /etc/resolv.conf` 和（如适用）`resolvectl status`，再执行第 8 步。

### 7D. resolvconf / cloud-init / DHCP / 不存在 / 断链 / 来源不明

这些情形没有适用于所有发行版镜像和云平台的安全替换命令。先检查：

```bash
ls -ld /etc/resolv.conf
readlink /etc/resolv.conf || true
systemctl is-active systemd-resolved NetworkManager systemd-networkd 2>/dev/null || true
ls -l /run/resolvconf/resolv.conf /etc/resolvconf/resolv.conf.d/ 2>/dev/null || true
ls /etc/network/interfaces /etc/netplan/ /etc/cloud/cloud.cfg.d/ 2>/dev/null || true
```

确认实际负责重写 DNS 的管理程序后，在**该程序的持久化配置**中将 DNS 指向 `127.0.0.1` 并按其机制生效。不要未经核实就删除符号链接、强制建立普通文件，或对断链执行 `tee`；这可能将内容写到错误目标。

## 8. 最终验收

```bash
systemctl is-active smartdns
systemctl is-enabled smartdns
sudo ss -lntup '( sport = :53 )'
dig @127.0.0.1 debian.org A +time=3 +tries=2
dig debian.org A +time=3 +tries=2
getent ahostsv4 debian.org
cat /etc/resolv.conf
sudo journalctl -u smartdns -n 30 --no-pager
```

预期 SmartDNS 是 `active` 与 `enabled`，指定地址和系统默认 DNS 查询均可正常返回 IPv4，且没有公网 53 端口意外监听。**7B 分支中，默认 DNS 显示 `127.0.0.53` 是正常的 resolved stub**；是否最终转发 SmartDNS，还需要检查 `resolvectl status`、每链路 DNS 和 SmartDNS 的解析活动。验证重启后持久性时，应提前准备云控制台回滚。

## 9. 更新 SmartDNS

由于采用本地 `.deb` 安装，没有自动跟随 GitHub Release 的更新渠道。更新时从 [官方 Releases](https://github.com/pymumu/smartdns/releases) 获取与系统架构相符的文件，核对发布者和校验信息，备份配置后运行（替换示例文件名）：

```bash
cd /tmp
sudo apt-get install -y -o Dpkg::Options::='--force-confold' ./新版-smartdns.deb
sudo systemctl restart smartdns
sudo systemctl status smartdns --no-pager -l
```

更新后应重新验证解析；保留旧配置的选项并不能代替发行说明和兼容性检查。

## 10. 回滚与卸载

**先恢复系统可用的 DNS，确认成功后再停用 SmartDNS。不要颠倒顺序。**

- **7A 普通文件：** `sudo cp -a /etc/resolv.conf.before-smartdns /etc/resolv.conf`，再测试 `dig debian.org A`。
- **7B systemd-resolved：** 执行 `sudo rm -f /etc/systemd/resolved.conf.d/90-smartdns.conf` 和 `sudo systemctl restart systemd-resolved`。若第 7B 步额外改动了 resolv.conf 的符号链接，确认之前保存的是**原始链接**后，用 `sudo rm /etc/resolv.conf && sudo cp -a /etc/resolv.conf.before-smartdns /etc/resolv.conf` 恢复。测试 `resolvectl query debian.org`。
- **7C NetworkManager：** 用修改前记录的值恢复连接的 `ipv4.dns`、`ipv4.ignore-auto-dns` 和实际修改过的 `ipv6.ignore-auto-dns`，必要时谨慎 reapply 网卡；远程操作不要猜原值或直接断开网络连接。
- **7D 其他管理器：** 按原管理程序的备份/回滚流程操作，不适用固定的文件复制命令。

确认 `dig debian.org A` 解析正常后：

```bash
sudo systemctl disable --now smartdns
```

若要彻底卸载：

```bash
sudo apt-get purge -y smartdns
```

如果旧教程曾对**普通文件** `/etc/resolv.conf` 设置 `chattr +i`，恢复前可能需先 `sudo chattr -i /etc/resolv.conf`；不要对未知符号链接目标盲目操作。

## 11. 系统维护与安全说明

安装 SmartDNS 本身不需要执行 `apt-get upgrade`、`full-upgrade`、`autoremove` 或 `clean`。这些属于独立的系统维护；正常的 Debian 安全更新仍应按维护策略执行。

- 不要在未设置访问控制时把 SmartDNS 监听改为 `0.0.0.0:53`。
- 不要从不明来源下载安装包。
- DNS-over-HTTPS/3 负责上游传输加密，不等于匿名，公共 DNS 服务方仍可能接触查询数据。
- 不建议锁死 `/etc/resolv.conf`；尤其不要破坏由管理程序自动生成的文件。
- 先测试 `dig @127.0.0.1`，再切换系统默认 DNS；保持 SSH 登录并预备控制台恢复。

## 参考资料

- [原教程](https://github.com/AntonyCyrus/Install-SmartDNS)
- [SmartDNS 官方仓库](https://github.com/pymumu/smartdns)
- [SmartDNS Releases](https://github.com/pymumu/smartdns/releases)
- [SmartDNS 官方配置示例](https://github.com/pymumu/smartdns/blob/master/etc/smartdns.conf)
- [systemd-resolved 配置文档](https://www.freedesktop.org/software/systemd/man/latest/resolved.conf.html)
- [NetworkManager nmcli 文档](https://networkmanager.dev/docs/api/latest/nmcli.html)
- [Debian APT 手册](https://www.debian.org/doc/manuals/debian-handbook/apt.zh-cn.html)
