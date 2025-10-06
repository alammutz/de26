### 1. Произведение базовой настройки устройств
- ISP
  ```bash
#!/bin/bash
echo "ISP" > /etc/hostname
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
echo "BOOTPROTO=static" >> /etc/net/ifaces/ens21/options
echo "172.16.1.1/28" > /etc/net/ifaces/ens21/ipv4address
echo "BOOTPROTO=static" >> /etc/net/ifaces/ens22/options
echo "172.16.2.1/28" > /etc/net/ifaces/ens22/ipv4address
sed -i 's/net.ipv4.ip_forward = 0/net.ipv4.ip_forward = 1/' /etc/net/sysctl.conf
sysctl -p /etc/net/sysctl.conf
systemctl restart network
apt-get update && apt-get install -y iptables
iptables -t nat -A POSTROUTING -o ens20 -s 0/0 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables
echo "Настройка завершена!" ```
