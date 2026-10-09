# Debian 13 VPS 安装 SmartDNS

本教程适用于使用 `systemd` 的 Debian 13 VPS，目标是让 SmartDNS 仅为本机提供 DNS 解析，并使用 DoH3 上游服务器。

## 功能与限制

- 仅监听 `127.0.0.1:53`，不会向公网开放 DNS 服务。
- 使用 Cloudflare 和 Google 的 DoH3 服务作为上游。
- 使用两个传统 DNS 地址作为 DoH3 域名的引导解析服务器；它们不会进入默认查询组。
- 默认阻止 IPv6 `AAAA` 结果，适合没有 IPv6 网络的 VPS。
- 安装包来自 SmartDNS 官方 GitHub Release，并使用 SHA-256 校验。
- 本文使用手动下载的 `.deb` 包，因此 SmartDNS 不会随 `apt upgrade` 自动更新。

## 适用环境

- Debian 13
- `amd64` / `x86_64` 架构
- 拥有 `root` 权限，或当前用户可以使用 `sudo`
- 服务器使用 `systemd`

本文命令可以逐段复制执行。若当前已经是 `root` 用户且系统没有安装 `sudo`，请删除命令开头的 `sudo`。

## 一、检查系统环境

检查系统架构：

```bash
dpkg --print-architecture
```

预期输出：

```text
amd64
```

检查 PID 1 是否为 `systemd`：

```bash
ps -p 1 -o comm=
```

预期输出：

```text
systemd
```

检查 53 端口是否已被其他程序占用：

```bash
sudo ss -lntup | grep -E '127\.0\.0\.1:53|0\.0\.0\.0:53|\[::\]:53' || true
```

如果没有输出，可以继续。如果已有 `systemd-resolved`、`dnsmasq`、`named` 或其他 DNS 程序监听 53 端口，应先查明用途并解决端口冲突。

## 二、安装必要工具

这里只更新软件包索引并安装必要工具，不升级其他软件包：

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates wget dnsutils
```

## 三、下载并校验 SmartDNS

本教程固定使用官方稳定版 `Release48.4`：

```bash
cd /tmp
wget -O smartdns.deb https://github.com/pymumu/smartdns/releases/download/Release48.4/smartdns.1.2026.08.05-0921.x86_64-debian-all.deb
```

校验 SHA-256：

```bash
echo '4cc1b8f9db10110e1212d981cfee7eb89a3c80f5766760e3b5f7541c8da8b546  smartdns.deb' | sha256sum -c -
```

必须看到以下结果后再继续：

```text
smartdns.deb: OK
```

如果显示 `FAILED`，不要安装该文件。请删除损坏的下载文件并重新下载。

## 四、安装 SmartDNS

使用 APT 安装本地 `.deb` 文件，以便自动处理依赖关系：

```bash
sudo apt-get install -y ./smartdns.deb
```

查看已安装版本：

```bash
smartdns -v
```

## 五、备份并写入配置

先备份软件包自带的配置文件：

```bash
sudo cp -a /etc/smartdns/smartdns.conf /etc/smartdns/smartdns.conf.package-default
```

写入配置：

```bash
sudo tee /etc/smartdns/smartdns.conf >/dev/null <<'EOF'
# 仅监听本机回环地址，避免成为公网 DNS 服务器。
bind 127.0.0.1:53
bind-tcp 127.0.0.1:53

# 该 VPS 不使用 IPv6，因此让 AAAA 查询返回 SOA。
force-AAAA-SOA yes
dualstack-ip-selection no

# 缓存与域名预取设置。
cache-size 4096
prefetch-domain yes

# 使用 TCP 443、TCP 80 和 ICMP 检查候选地址速度。
speed-check-mode tcp:443,tcp:80,ping

# 仅用于解析 DoH3 服务器域名，不加入默认查询组。
server 1.1.1.1 -bootstrap-dns -exclude-default-group
server 8.8.8.8 -bootstrap-dns -exclude-default-group

# 默认上游 DNS：Cloudflare 和 Google DoH3。
server-h3 h3://cloudflare-dns.com/dns-query
server-h3 h3://dns.google/dns-query

# 上游暂时不可用时，允许短时间返回过期缓存。
serve-expired yes
serve-expired-ttl 3600
serve-expired-reply-ttl 3
EOF
```

> [!NOTE]
> 如果 VPS 可以正常使用 IPv6，并且你希望客户端获得 IPv6 地址，请删除 `force-AAAA-SOA yes` 和 `dualstack-ip-selection no` 两行。

## 六、启动并验证 SmartDNS

启用 SmartDNS 开机启动，并立即重启服务以载入新配置：

```bash
sudo systemctl enable smartdns
sudo systemctl restart smartdns
```

查看服务状态：

```bash
sudo systemctl status smartdns --no-pager -l
```

预期看到：

```text
Active: active (running)
```

在修改系统 DNS 以前，先明确指定 `127.0.0.1`，确认应答者确实是 SmartDNS：

```bash
nslookup -querytype=ptr smartdns 127.0.0.1
```

正常结果中应出现 `smartdns name = smartdns.`，或者显示当前主机名。这是官方验证方法的指定服务器版本；此时不能省略末尾的 `127.0.0.1`，因为系统默认 DNS 尚未切换到 SmartDNS。

再执行普通域名查询，验证实际解析能力：

```bash
dig @127.0.0.1 debian.org A +short
dig @127.0.0.1 cloudflare.com A +short
```

两条命令都应返回一个或多个 IPv4 地址。

确认 53 端口只监听本机地址：

```bash
sudo ss -lntup | grep ':53'
```

输出中的监听地址应为 `127.0.0.1:53`，不应出现 `0.0.0.0:53` 或 `[::]:53`。

如服务启动失败，查看日志：

```bash
sudo journalctl -u smartdns -n 100 --no-pager
```

在上述测试成功以前，不要修改 `/etc/resolv.conf`。

## 七、让本机使用 SmartDNS

先检查 `/etc/resolv.conf` 是否为符号链接：

```bash
ls -l /etc/resolv.conf
```

### 情况 A：`/etc/resolv.conf` 是普通文件

先备份：

```bash
sudo cp -a /etc/resolv.conf /etc/resolv.conf.before-smartdns
```

再将本机 DNS 指向 SmartDNS：

```bash
printf 'nameserver 127.0.0.1\noptions timeout:2 attempts:2\n' | sudo tee /etc/resolv.conf >/dev/null
```

验证系统默认 DNS：

```bash
nslookup -querytype=ptr smartdns
dig debian.org A +short
getent ahostsv4 debian.org
```

第一条是 SmartDNS 官方提供的生效验证命令。正常结果中应出现 `smartdns name = smartdns.`，或者显示当前主机名；同时，命令开头显示的 DNS 服务器地址应为 `127.0.0.1`。

### 情况 B：`/etc/resolv.conf` 是符号链接

这通常表示 DNS 配置由 `systemd-resolved`、NetworkManager、云服务商初始化程序或其他网络管理器维护。此时不要直接删除链接，也不要直接覆盖文件；应在对应的网络管理器中把 DNS 设置为 `127.0.0.1`。

可以先用下面的命令查看链接目标：

```bash
readlink -f /etc/resolv.conf
```

不同 VPS 服务商的网络配置方式可能不同。在不了解管理程序前直接替换该链接，可能导致重启后 DNS 失效。

> [!WARNING]
> 不建议对 `/etc/resolv.conf` 执行 `chattr +i`。将文件锁定会阻止网络管理器正常更新 DNS，也会增加以后修改和排错的难度。

## 八、最终检查

依次执行：

```bash
systemctl is-active smartdns
systemctl is-enabled smartdns
nslookup -querytype=ptr smartdns
dig debian.org A +short
sudo ss -lntup | grep ':53'
sudo journalctl -u smartdns -n 30 --no-pager
```

正常情况下：

- 前两条命令分别返回 `active` 和 `enabled`。
- `nslookup` 结果中的 `name` 为 `smartdns` 或当前主机名。
- DNS 查询返回 IPv4 地址。
- SmartDNS 只监听 `127.0.0.1:53`。
- 日志中没有持续重复的错误。

## 更新 SmartDNS

本教程安装的是从 GitHub 下载的本地 `.deb` 包，没有添加 APT 软件源。因此：

- Debian 的定期 APT 更新任务不会自动更新 SmartDNS。
- 更新时应从 [SmartDNS 官方 Releases](https://github.com/pymumu/smartdns/releases) 下载新版 `.deb`。
- 安装新版前应核对文件名、版本和发布者提供的校验信息。

下载新版安装包后，可以使用以下形式升级，并保留现有配置文件：

```bash
cd /tmp
sudo apt-get install -y -o Dpkg::Options::='--force-confold' ./新版-smartdns.deb
sudo systemctl restart smartdns
sudo systemctl status smartdns --no-pager -l
```

`新版-smartdns.deb` 是示例文件名，执行前必须替换为实际下载的文件名。

## 回滚 DNS 设置

如果切换本机 DNS 后出现解析故障，且之前按照本文创建了备份，可执行：

```bash
sudo cp -a /etc/resolv.conf.before-smartdns /etc/resolv.conf
sudo systemctl disable --now smartdns
```

然后测试：

```bash
getent ahostsv4 debian.org
```

如果你曾按照旧教程锁定 `/etc/resolv.conf`，需要先解除锁定：

```bash
sudo chattr -i /etc/resolv.conf
```

## 卸载 SmartDNS

先恢复原 DNS 配置，再卸载软件包：

```bash
sudo cp -a /etc/resolv.conf.before-smartdns /etc/resolv.conf
sudo systemctl disable --now smartdns
sudo apt-get purge -y smartdns
```

如果备份文件不存在，请不要直接复制；先为服务器设置一个可用的 DNS，再卸载 SmartDNS。

## 为什么不执行全面升级

安装 SmartDNS 只需要安装它自身及必要依赖。以下命令属于整台 Debian 系统的维护操作，不是安装 SmartDNS 的必要步骤：

```bash
sudo apt-get upgrade
sudo apt-get full-upgrade
sudo apt-get autoremove
sudo apt-get clean
```

其中 `full-upgrade` 可能安装新包或移除已有软件包，应由独立的系统维护计划执行。不要把它与单个应用的安装教程捆绑运行。

## 官方资料

- [SmartDNS GitHub 仓库](https://github.com/pymumu/smartdns)
- [SmartDNS 官方 Releases](https://github.com/pymumu/smartdns/releases)
- [SmartDNS 配置示例](https://github.com/pymumu/smartdns/blob/master/etc/smartdns.conf)
- [Debian APT 使用手册](https://www.debian.org/doc/manuals/debian-handbook/apt.zh-cn.html)

## 安全提醒

- 不要将 SmartDNS 绑定到 `0.0.0.0:53`，除非你明确需要为其他设备提供 DNS，并已经设置访问控制和防火墙。
- 不要从不明网站下载 `.deb` 安装包。
- 修改 DNS 前先完成 `dig @127.0.0.1` 测试。
- 建议保留一个已登录的 SSH 会话，确认新 DNS 正常后再断开连接。
