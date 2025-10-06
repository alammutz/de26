### 1. Произведение базовой настройки устройств
- ISP
  
  ```bash
#!/bin/bash

# Установка hostname
echo "ISP" > /etc/hostname
hostname ISP

# Создание директорий для сетевых интерфейсов
mkdir -p /etc/net/ifaces/{ens20,ens21,ens22}

# Настройка интерфейса ens20 (DHCP)
cat <<EOF > /etc/net/ifaces/ens20/options
BOOTPROTO=dhcp
CONFIG_IPV4=yes
DISABLED=no
TYPE=eth
EOF

# Копирование конфигурации для ens21 и ens22
cp /etc/net/ifaces/ens20/options /etc/net/ifaces/ens21/options
cp /etc/net/ifaces/ens20/options /etc/net/ifaces/ens22/options

# Настройка статического IP для ens21
echo "BOOTPROTO=static" >> /etc/net/ifaces/ens21/options
echo "172.16.1.1/28" > /etc/net/ifaces/ens21/ipv4address

# Настройка статического IP для ens22
echo "BOOTPROTO=static" >> /etc/net/ifaces/ens22/options
echo "172.16.2.1/28" > /etc/net/ifaces/ens22/ipv4address

# Включение IP-форвардинга
sed -i 's/net.ipv4.ip_forward = 0/net.ipv4.ip_forward = 1/' /etc/net/sysctl.conf
sysctl -p /etc/net/sysctl.conf

# Перезапуск сети
systemctl restart network

# Установка iptables и настройка NAT
apt-get update && apt-get install -y iptables
iptables -t nat -A POSTROUTING -o ens20 -s 0/0 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables

# Включение iptables как сервиса
systemctl enable --now iptables

echo "Настройка завершена!"
