</details>
<details>
  <summary>2,3. Конфигурация файлового хранилища на сервере HQ-SRV</summary>

###### Необходимо добавить 2 диска по 1 ГБ

- HQ-SRV

```
lsblk
mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sd[b-c]
mdadm --detail --scan --verbose >> /etc/mdadm.conf
apt-get update && apt-get install fdisk -y
echo -e "n\np\n1\n\n\nw" | fdisk /dev/md0
mkfs.ext4 /dev/md0p1
echo "/dev/md0p1 /raid ext4 defaults 0 0" >> /etc/fstab
mkdir /raid
mount -a
apt-get install nfs-kernel-server -y
mkdir /raid/nfs
chown 99:99 /raid/nfs
chmod 777 /raid/nfs
echo "/raid/nfs 192.168.2.0/28(rw,sync,no_subtree_check)" >> /etc/exports
exportfs -a
exportfs -v
systemctl enable nfs-server
systemctl restart nfs-server
```

- HQ-CLI

```
apt-get update && apt-get install nfs-common -y
mkdir -p /mnt/nfs
echo "192.168.1.10:/raid/nfs /mnt/nfs nfs intr,soft,_netdev,x-systemd.automount 0 0" >> /etc/fstab
mount -a
mount -v
touch /mnt/nfs/test
```

- HQ-SRV

```
ls /raid/nfs
```

</details>
<details>
  <summary>2-4. Настройка службы сетевого времени</summary>


- ISP

```
apt-get install chrony -y
cat > /etc/chrony.conf <<EOF
driftfile /var/lib/chrony/drift
logdir /var/log/chrony
log measurements statistics tracking
local stratum 5
allow 0/0
server 127.0.0.1 iburst prefer
hwtimestamp *
EOF
systemctl enable --now chronyd
systemctl restart chronyd
chronyc sources
chronyc tracking | grep Stratum
```

- HQ-RTR

```
en
conf t
ntp server 172.16.1.4
ntp timezone utc+5
exit
show ntp status
write memory
```

- BR-RTR

```
en
conf t
ntp server 172.16.2.5
ntp timezone utc+5
exit
show ntp status
write memory
```

- HQ-CLI

```
apt-get install -y chrony
cat > /etc/chrony.conf <<EOF
server 172.16.1.4 iburst prefer
driftfile /var/lib/chrony/drift
logdir /var/log/chrony
EOF
systemctl enable --now chronyd
systemctl restart chronyd
chronyc sources
timedatectl
```

- HQ-SRV

```
apt-get install -y chrony
cat > /etc/chrony.conf <<EOF
server 172.16.1.4 iburst prefer
driftfile /var/lib/chrony/drift
logdir /var/log/chrony
EOF
systemctl enable --now chronyd
systemctl restart chronyd
chronyc sources
timedatectl
```

- BR-SRV

```
apt-get install -y chrony
cat > /etc/chrony.conf <<EOF
server 172.16.2.5 iburst prefer
driftfile /var/lib/chrony/drift
logdir /var/log/chrony
EOF
systemctl enable --now chronyd
systemctl restart chronyd
chronyc sources
timedatectl
```

</details>
<details>
  <summary>2-5. Конфигурация Ansible на сервере BR-SRV</summary>


- BR-SRV
  
```
apt-get update
apt-get install ansible -y
cat > /etc/ansible/hosts <<EOF
[HQ-SRV]
192.168.1.10 ansible_user=remote_user ansible_port=2026
[HQ-CLI]
192.168.2.10 ansible_user=remote_user ansible_port=2026
[HQ-RTR]
192.168.1.1 ansible_user=net_admin ansible_password=P@ssw0rd ansible_connection=network_cli ansible_network_os=ios
[BR-RTR]
192.168.3.1 ansible_user=net_admin ansible_password=P@ssw0rd ansible_connection=network_cli ansible_network_os=ios
EOF
cat > /etc/ansible/ansible.cfg <<EOF
[defaults]
ansible_python_interpreter=/usr/bin/python3
interpreter_python=auto_silent
host_key_checking=false
EOF
```

- HQ-CLI
  
```
useradd remote_user -u 2026
echo "remote_user:P@ssw0rd" | chpasswd
sed -i 's/^#\s*\(%wheel\s*ALL=(ALL:ALL)\s*NOPASSWD:\s*ALL\)/\1/' /etc/sudoers
gpasswd -a "remote_user" wheel
cat > /etc/ssh/sshd_config <<EOF
Port 2026
AllowUsers remote_user
MaxAuthTries 2
PasswordAuthentication yes
Banner /etc/openssh/banner
EOF
echo "Authorized access only" > /etc/openssh/banner
systemctl restart sshd
```

- BR-SRV
  
```
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
ssh-copy-id -o StrictHostKeyChecking=no -p 2026 remote_user@192.168.1.10
ssh-copy-id -o StrictHostKeyChecking=no -p 2026 remote_user@192.168.2.10
sleep 3
ansible all -m ping
```
