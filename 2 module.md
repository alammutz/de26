</details>
<details>
  <summary>1. Настройка доменного контроллера Samba на машине BR-SRV</summary>
  
- HQ-SRV
  
```
echo "server=/au-team.irpo/192.168.3.10" >> /etc/dnsmasq.conf
systemctl restart dnsmasq
```
- BR-SRV
  
```
if ! grep -q '^nameserver 8\.8\.8\.8$' /etc/resolv.conf; then
    echo 'nameserver 8.8.8.8' | sudo tee -a /etc/resolv.conf
fi
apt-get update && apt-get install wget dos2unix task-samba-dc -y
sleep 3
echo nameserver 192.168.1.10 >> /etc/resolv.conf
sleep 2
echo 192.168.3.10 br-srv.au-team.irpo >> /etc/hosts
rm -rf /etc/samba/smb.conf
samba-tool domain provision --realm=AU-TEAM.IRPO --domain=AU-TEAM --adminpass=P@ssw0rd --dns-backend=SAMBA_INTERNAL --server-role=dc --option='dns forwarder=192.168.1.10'
mv -f /var/lib/samba/private/krb5.conf /etc/krb5.conf
systemctl enable --now samba.service
samba-tool user add hquser1 P@ssw0rd
samba-tool user add hquser2 P@ssw0rd
samba-tool user add hquser3 P@ssw0rd
samba-tool user add hquser4 P@ssw0rd
samba-tool user add hquser5 P@ssw0rd
samba-tool group add hq
samba-tool group addmembers hq hquser1,hquser2,hquser3,hquser4,hquser5
wget https://raw.githubusercontent.com/sudo-project/sudo/main/docs/schema.ActiveDirectory
dos2unix schema.ActiveDirectory
sed -i 's/DC=X/DC=au-team,DC=irpo/g' schema.ActiveDirectory
head -$(grep -B1 -n '^dn:$' schema.ActiveDirectory | head -1 | grep -oP '\d+') schema.ActiveDirectory > first.ldif
tail +$(grep -B1 -n '^dn:$' schema.ActiveDirectory | head -1 | grep -oP '\d+') schema.ActiveDirectory | sed '/^-/d' > second.ldif
ldbadd -H /var/lib/samba/private/sam.ldb first.ldif --option="dsdb:schema update allowed"=true
ldbmodify -v -H /var/lib/samba/private/sam.ldb second.ldif --option="dsdb:schema update allowed"=true
samba-tool ou add 'ou=sudoers'
cat << EOF > sudoRole-object.ldif
dn: CN=prava_hq,OU=sudoers,DC=au-team,DC=irpo
changetype: add
objectClass: top
objectClass: sudoRole
cn: prava_hq
name: prava_hq
sudoUser: %hq
sudoHost: ALL
sudoCommand: /bin/grep
sudoCommand: /bin/cat
sudoCommand: /usr/bin/id
sudoOption: !authenticate
EOF
ldbadd -H /var/lib/samba/private/sam.ldb sudoRole-object.ldif
echo -e "dn: CN=prava_hq,OU=sudoers,DC=au-team,DC=irpo\nchangetype: modify\nreplace: nTSecurityDescriptor" > ntGen.ldif
ldbsearch  -H /var/lib/samba/private/sam.ldb -s base -b 'CN=prava_hq,OU=sudoers,DC=au-team,DC=irpo' 'nTSecurityDescriptor' | sed -n '/^#/d;s/O:DAG:DAD:AI/O:DAG:DAD:AI\(A\;\;RPLCRC\;\;\;AU\)\(A\;\;RPWPCRCCDCLCLORCWOWDSDDTSW\;\;\;SY\)/;3,$p' | sed ':a;N;$!ba;s/\n\s//g' | sed -e 's/.\{78\}/&\n /g' >> ntGen.ldif
ldbmodify -v -H /var/lib/samba/private/sam.ldb ntGen.ldif
```
  
- HQ-CLI
  
```
sed -i 's/BOOTPROTO=static/BOOTPROTO=dhcp/' /etc/net/ifaces/ens20/options
systemctl restart network
apt-get update && apt-get install bind-utils -y
system-auth write ad AU-TEAM.IRPO cli AU-TEAM 'administrator' 'P@ssw0rd'
reboot
sleep 3
apt-get install sudo libsss_sudo -y
control sudo public
sed -i '19 a\
sudo_provider = ad' /etc/sssd/sssd.conf
sed -i 's/services = nss, pam/services = nss, pam, sudo/' /etc/sssd/sssd.conf
sed -i '28 a\
sudoers: files sss' /etc/nsswitch.conf
rm -rf /var/lib/sss/db/*
sss_cache -E
systemctl restart sssd
```

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
echo -e "n\n\n\n\n\nw\n" | fdisk /dev/md0
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
apt-get update && apt-get install nfs-clients -y
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
  <summary>4. Настройка службы сетевого времени</summary>


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
write
```

- BR-RTR

```
en
conf t
ntp server 172.16.2.5
ntp timezone utc+5
exit
show ntp status
write
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
  <summary>5. Конфигурация Ansible на сервере BR-SRV</summary>


- BR-SRV
  
```
apt-get update
apt-get install ansible -y
cat > /etc/ansible/hosts <<EOF
VHS:
 hosts:
  HQ-SRV:
   ansible_host: 192.168.1.10
   ansible_user: remote_user
   ansible_port: 2026
  HQ-CLI:
   ansible_host: 192.168.2.10
   ansible_user: remote_user
   ansible_port: 2026
  HQ-RTR:
   ansible_host: 192.168.1.1
   ansible_user: net_admin
   ansible_password: P@sswOrd
   ansible_connection: network_cli
   ansible_network_os: ios
  BR-RTR:
   ansible_host: 192.168.3.1
   ansible_user: net_admin
   ansible_password: P@sswOrd
   ansible_connection: network_cli
   ansible_network_os: ios
EOF
if ! grep -q "^\[defaults\]" /etc/ansible/ansible.cfg; then
  echo "[defaults]" >> /etc/ansible/ansible.cfg
fi
sed -i '/^\[defaults\]/!b;n;/ansible_python_interpreter/!{i\
ansible_python_interpreter=/usr/bin/python3
\};/interpreter_python/!{i\
interpreter_python=auto_silent
\};/host_key_checking/!{i\
host_key_checking=false
\}' /etc/ansible/ansible.cfg

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
