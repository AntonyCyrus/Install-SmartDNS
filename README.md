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
更新
```bash
sudo apt update && sudo apt upgrade -y && sudo apt full-upgrade -y && sudo apt autoremove -y && sudo apt clean
```
修改SmartDNS.conf
```bash
nano /etc/smartdns/smartdns.conf
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
