### 1. Произведение настройки устройств
- ISP
```
hostname ISP
mkdir -p /etc/net/ifaces/{ens20,ens21,ens22}
cat <<EOF > /etc/net/ifaces/ens20/options
BOOTPROTO=dhcp
CONFIG_IPV4=yes
DISABLED=no
TYPE=eth
EOF
cp /etc/net/ifaces/ens20/options /etc/net/ifaces/ens21/options
cp /etc/net/ifaces/ens20/options /etc/net/ifaces/ens22/options
sed -i '/BOOTPROTO=dhcp/d' /etc/net/ifaces/ens21/options
echo "BOOTPROTO=static" >> /etc/net/ifaces/ens21/options
echo "172.16.1.1/28" > /etc/net/ifaces/ens21/ipv4address
sed -i '/BOOTPROTO=dhcp/d' /etc/net/ifaces/ens22/options
echo "BOOTPROTO=static" >> /etc/net/ifaces/ens22/options
echo "172.16.2.1/28" > /etc/net/ifaces/ens22/ipv4address
sed -i 's/#net.ipv4.ip_forward = 0/net.ipv4.ip_forward = 1/' /etc/sysctl.conf
sysctl -p /etc/sysctl.conf
systemctl restart network
apt-get update && apt-get install -y iptables
iptables -t nat -A POSTROUTING -o ens20 -s 0/0 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables
echo "Настройка завершена!"
```
- HQ-RTR
  ```
  en
conf t
hostname HQ-RTR
ip domain-name au-team.irpo
interface int0
  description "to isp"
  ip address 172.16.4.4/28
  ip nat outside
  ex
port te0
  service-instance te0/int0
    encapsulation untagged
    ex
  ex
interface int0
  connect port te0 service-instance te0/int0
  ex
interface int1
  description "to hq-srv"
  ip address 192.168.1.1/26
  ip nat inside
  ex
interface int2
  description "to hq-cli"
  ip address 192.168.2.1/28
  ip nat inside
  ex
port te1
  service-instance te1/int1
    encapsulation dot1q 100
    rewrite pop 1
    ex
  service-instance te1/int2
    encapsulation dot1q 200
    rewrite pop 1
    ex
  ex
interface int1
  connect port te1 service-instance te1/int1
  ex
interface int2
  connect port te1 service-instance te1/int2
  ex
interface int3
  description "999"
  ip address 192.168.99.1/29
  ex
port te1
  service-instance te1/int3
    encapsulation dot1q 999
    rewrite pop 1
    ex
  ex
interface int3
  connect port te1 service-instance te1/int3
  ex
write
ip route 0.0.0.0 0.0.0.0 172.16.4.1
write
username net_admin
password P@ssw0rd
role admin
ex
write
int tunnel.0
  ip add 172.16.0.1/30
  ip mtu 1400
  ip tunnel 172.16.4.4 172.16.5.5 mode gre
  ip ospf authentication-key ecorouter
  exit
write
router ospf 1
  network 172.16.5.0/28 area 0
  network 192.168.3.0/27 area 0
  passive-interface default
  no passive-interface tunnel.0
  area 0 authentication
  ex
write
ip nat pool NAT_POOL 192.168.1.1-192.168.1.254,192.168.2.1-192.168.2.254
ip nat source dynamic inside-to-outside pool NAT_POOL overload interface int0
interface int0
  write memory
  ex
ntp timezone utc+5
ex
write
ip pool cli_pool 192.168.2.10-192.168.2.10
dhcp-server 1
  ip pool cli_pool 1
    mask 255.255.255.240
    gateway 192.168.2.1
    dns 192.168.1.10
    domain-name au-team.irpo
    exit
  ex
interface int2
  dhcp-server 1
  write
  ex
  ```
