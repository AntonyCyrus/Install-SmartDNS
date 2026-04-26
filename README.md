# Install-SmartDNS
更新
```bash
sudo apt update && sudo apt upgrade -y && sudo apt full-upgrade -y && sudo apt autoremove -y && sudo apt clean
```
下载SmartDNS对应版本安装包
```bash
wget https://github.com/pymumu/smartdns/releases/download/Release47.1/smartdns.1.2025.11.09-1443.x86_64-debian-all.deb
```
安装
```bash
dpkg -i smartdns.1.2025.11.09-1443.x86_64-debian-all.deb
```
让 systemd 重新读取服务配置文件
```bash
systemctl daemon-reload
```
更新
```bash
sudo apt update && sudo apt upgrade -y && sudo apt full-upgrade -y && sudo apt autoremove -y && sudo apt clean
```
修改SmartDNS.conf
```bash
cat << EOF > /etc/smartdns/smartdns.conf
bind 127.0.0.1:53
bind-tcp 127.0.0.1:53

force-AAAA-SOA yes
dualstack-ip-selection no

cache-size 4096
prefetch-domain yes

speed-check-mode tcp:443,tcp:80,ping

server-h3 h3://cloudflare-dns.com/dns-query
server-h3 h3://dns.google/dns-query

server 8.8.8.8 -group bootstrap-dns
nameserver /dns.google/bootstrap-dns
nameserver /cloudflare-dns.com/bootstrap-dns

serve-expired yes
serve-expired-ttl 3600
serve-expired-reply-ttl 3
EOF
```
安装dnsutils
```bash
sudo apt install dnsutils
```
修改resolv.conf
```bash
echo "nameserver 127.0.0.1" | sudo tee /etc/resolv.conf
```
上锁
```bash
sudo chattr +i /etc/resolv.conf
```
启动
```bash
systemctl enable smartdns
systemctl start smartdns
```
验证
```bash
nslookup -querytype=ptr smartdns
```
检测向443目标端口发送的UDP请求
```bash
sudo tcpdump -i any udp dst port 443 -nn
```
安装iptables
```bash
apt install iptables
```
关闭53端口公网访问（但是放行本地回环）
```bash
sudo iptables -I INPUT 1 -i lo -j ACCEPT
sudo iptables -I INPUT 2 -p udp --dport 53 -j DROP
sudo iptables -I INPUT 3 -p tcp --dport 53 -j DROP
```
