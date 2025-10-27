### 1. Произведение настройки устройств
- ISP
```
hostnamectl set-hostname ISP
mkdir -p /etc/net/ifaces/{ens20,ens21,ens22}
cat > /etc/net/ifaces/ens20/options <<EOF
BOOTPROTO=dhcp
CONFIG_IPV4=yes
DISABLED=no
TYPE=eth
EOF
cp /etc/net/ifaces/ens20/options /etc/net/ifaces/ens21/options
cp /etc/net/ifaces/ens20/options /etc/net/ifaces/ens22/options
sed -i 's/BOOTPROTO=dhcp/BOOTPROTO=static/' /etc/net/ifaces/ens21/options
sed -i 's/BOOTPROTO=dhcp/BOOTPROTO=static/' /etc/net/ifaces/ens22/options
echo "172.16.1.1/28" > /etc/net/ifaces/ens21/ipv4address
echo "172.16.2.1/28" > /etc/net/ifaces/ens22/ipv4address
sed -i 's/net.ipv4.ip_forward = 0/net.ipv4.ip_forward = 1/' /etc/net/sysctl.conf
sysctl -p
systemctl restart network
apt-get update && apt-get install iptables -y
iptables -t nat -A POSTROUTING -o ens20 -s 0/0 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables
apt-get update && apt-get install --reinstall tzdata -y
timedatectl set-timezone Asia/Yekaterinburg
exec bash
```
- HQ-RTR

```
en
conf t
hostname hq-rtr
ip domain-name au-team.irpo
interface int0
description "to isp"
ip address 172.16.1.4/28
ip nat outside
exit
port te0
service-instance te0/int0
encapsulation untagged
exit
exit
interface int0
connect port te0 service-instance te0/int0
exit
interface int1
description "to hq-srv"
ip address 192.168.1.1/27
ip nat inside
exit
interface int2
description "to hq-cli"
ip address 192.168.2.1/28
ip nat inside
exit
port te1
service-instance te1/int1
encapsulation dot1q 100
rewrite pop 1
exit
service-instance te1/int2
encapsulation dot1q 200
rewrite pop 1
exit
exit
interface int1
connect port te1 service-instance te1/int1
exit
interface int2
connect port te1 service-instance te1/int2
exit
ip route 0.0.0.0 0.0.0.0 172.16.1.1
write
username net_admin
password P@ssw0rd
role admin
exit
write
interface int3
description "999"
ip address 192.168.99.1/29
exit
port te1
service-instance te1/int3
encapsulation dot1q 999
rewrite pop 1
exit
exit
interface int3
connect port te1 service-instance te1/int3
write
interface tunnel.0
ip address 172.16.0.1/30
ip mtu 1400
ip tunnel 172.16.1.4 172.16.2.5 mode gre
ip ospf authentication-key ecorouter
exit
write
router ospf 1
network 172.16.0.1/30 area 0
network 192.168.1.0/27 area 0
network 192.168.2.0/28 area 0
passive-interface default
no passive-interface tunnel.0
area 0 authentication
exit
write
ip name-server 8.8.8.8
ip nat pool NAT_POOL 192.168.1.1-192.168.1.254,192.168.2.1-192.168.2.254
ip nat source dynamic inside-to-outside pool NAT_POOL overload interface int0
write
ip pool cli_pool 192.168.2.10-192.168.2.10
dhcp-server 1
pool cli_pool 1
mask 255.255.255.240
gateway 192.168.2.1
dns 192.168.1.10
domain-name au-team.irpo
exit
interface int2
dhcp-server 1
exit
exit
write
en
conf t
ntp timezone utc+5
ntp server 172.16.1.1
write
```

- BR-RTR

```
en
conf t
hostname br-rtr
ip domain-name au-team.irpo
interface int0
description "to isp"
ip address 172.16.2.5/28
ip nat outside
exit
port te0
service-instance te0/int0
encapsulation untagged
exit
exit
interface int0
connect port te0 service-instance te0/int0
exit
interface int1
description "to br-srv"
ip address 192.168.3.1/28
ip nat inside
exit
port te1
service-instance te1/int1
encapsulation untagged
exit
exit
interface int1
connect port te1 service-instance te1/int1
exit
ip route 0.0.0.0 0.0.0.0 172.16.2.1
write
username net_admin
password P@ssw0rd
role admin
exit
write
interface tunnel.0
ip address 172.16.0.2/30
ip mtu 1400
ip tunnel 172.16.2.5 172.16.1.4 mode gre
ip ospf authentication-key ecorouter
exit
write
router ospf 1
network 172.16.0.2/30 area 0
network 192.168.3.0/28 area 0
passive-interface default
no passive-interface tunnel.0
area 0 authentication
exit
write
ip name-server 8.8.8.8
ip nat pool NAT_POOL 192.168.3.1-192.168.3.254
ip nat source dynamic inside-to-outside pool NAT_POOL overload interface int0
write
ntp timezone utc+5
ntp server 172.16.2.1
write
```

- BR-SRV

```
hostnamectl set-hostname br-srv.au-team.irpo
exec bash
mkdir -p /etc/net/ifaces/ens20
cat > /etc/net/ifaces/ens20/options <<EOF
TYPE=eth
BOOTPROTO=static
DISABLED=no
CONFIG_IPV4=yes
EOF
echo "192.168.3.10/28" > /etc/net/ifaces/ens20/ipv4address
echo "default via 192.168.3.1" > /etc/net/ifaces/ens20/ipv4route
systemctl restart network
useradd sshuser -u 2026
echo "sshuser:P@ssw0rd" | chpasswd
sed -i 's/^#\s*\(%wheel\s*ALL=(ALL:ALL)\s*NOPASSWD:\s*ALL\)/\1/' /etc/sudoers
gpasswd -a "sshuser" wheel
cat > /etc/openssh/sshd_config <<EOF
Port 2026
AllowUsers sshuser
MaxAuthTries 2
PasswordAuthentication yes
Banner /etc/openssh/banner
EOF
echo "Authorized access only" > /etc/openssh/banner
systemctl restart sshd
echo nameserver 192.168.1.10 > /etc/resolv.conf
timedatectl set-timezone Asia/Yekaterinburg
```

- HQ-SRV

```
hostnamectl set-hostname hq-srv.au-team.irpo
mkdir -p /etc/net/ifaces/ens20
cat > /etc/net/ifaces/ens20/options <<EOF
TYPE=eth
BOOTPROTO=static
DISABLED=no
CONFIG_IPV4=yes
EOF
echo "192.168.1.10/27" > /etc/net/ifaces/ens20/ipv4address
echo "default via 192.168.1.1" > /etc/net/ifaces/ens20/ipv4route
systemctl restart network
useradd sshuser -u 2026
echo "sshuser:P@ssw0rd" | chpasswd
sed -i 's/^#\s*\(%wheel\s*ALL=(ALL:ALL)\s*NOPASSWD:\s*ALL\)/\1/' /etc/sudoers
gpasswd -a "sshuser" wheel
cat > /etc/openssh/sshd_config <<EOF
Port 2026
AllowUsers sshuser
MaxAuthTries 2
PasswordAuthentication yes
Banner /etc/openssh/banner
EOF
echo "Authorized access only" > /etc/openssh/banner
systemctl restart sshd
echo "nameserver 8.8.8.8" > /etc/resolv.conf
apt-get update && apt-get install dnsmasq -y
systemctl enable --now dnsmasq
cat > /etc/dnsmasq.conf <<EOF
no-resolv
domain=au-team.irpo
server=8.8.8.8
interface=*
address=/hq-rtr.au-team.irpo/192.168.1.1
ptr-record=1.1.168.192.in-addr.arpa,hq-rtr.au-team.irpo
address=/docker.au-team.irpo/192.168.1.1
address=/web.au-team.irpo/192.168.2.1
address=/br-rtr.au-team.irpo/192.168.3.1
address=/hq-srv.au-team.irpo/192.168.1.10
ptr-record=10.1.168.192.in-addr.arpa,hq-srv.au-team.irpo
address=/hq-cli.au-team.irpo/192.168.2.10
ptr-record=10.2.168.192.in-addr.arpa,hq-cli.au-team.irpo
address=/br-srv.au-team.irpo/192.168.3.10
EOF
echo "192.168.1.1 hq-rtr.au-team.irpo" >> /etc/hosts
systemctl restart dnsmasq
timedatectl set-timezone Asia/Yekaterinburg
```

- HQ-CLI

```
hostnamectl set-hostname hq-cli.au-team.irpo
exec bash
mkdir -p /etc/net/ifaces/ens20
cat > /etc/net/ifaces/ens20/options <<EOF
TYPE=eth
BOOTPROTO=static
DISABLED=no
CONFIG_IPV4=yes
EOF
echo "192.168.2.10/28" > /etc/net/ifaces/ens20/ipv4address
echo "default via 192.168.2.1" > /etc/net/ifaces/ens20/ipv4route
systemctl restart network
useradd sshuser -u 2026
echo "sshuser:P@ssw0rd" | chpasswd
sed -i 's/^#\s*\(%wheel\s*ALL=(ALL:ALL)\s*NOPASSWD:\s*ALL\)/\1/' /etc/sudoers
gpasswd -a "sshuser" wheel
cat > /etc/openssh/sshd_config <<EOF
Port 2026
AllowUsers sshuser
MaxAuthTries 2
PasswordAuthentication yes
Banner /etc/openssh/banner
EOF
echo "Authorized access only" > /etc/openssh/banner
systemctl restart sshd
echo "nameserver 8.8.8.8" > /etc/resolv.conf
rm -rf /etc/net/ifaces/ens20/ipv4address /etc/net/ifaces/ens20/ipv4route
cat > /etc/net/ifaces/ens20/options <<EOF
TYPE=eth
BOOTPROTO=dhcp
DISABLED=no
CONFIG_IPV4=yes
EOF
systemctl restart network
timedatectl set-timezone Asia/Yekaterinburg
systemctl restart sshd
```
