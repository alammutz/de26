</details>
<details>
  <summary>ISP</summary>

```
apt-get update && apt-get install nginx -y
cat << EOF > /etc/nginx/sites-available.d/proxy.conf
server {
    listen 80;
    server_name docker.au-team.irpo;
    location / {
        proxy_pass http://172.16.2.2:8080;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
    }
}
server {
    listen 80;
    server_name web.au-team.irpo;
    location / {
        proxy_pass http://192.168.1.10:80;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
    }
}
EOF
ln -sf /etc/nginx/sites-available.d/proxy.conf /etc/nginx/sites-enabled.d/
nginx -t
systemctl enable --now nginx
systemctl restart nginx

apt-get install chrony -y
cat << EOF > /etc/chrony.conf
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

apt-get install apache2-htpasswd -y
htpasswd -bc /etc/nginx/.htpasswd WEB P@ssw0rd
cat << EOF > /etc/nginx/sites-available.d/proxy.conf
server {
    listen 80;
    server_name web.au-team.irpo;
    auth_basic "Restricted Access";
    auth_basic_user_file /etc/nginx/.htpasswd;
    location / {
        proxy_pass http://192.168.1.10:80;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
    }
}
server {
    listen 80;
    server_name docker.au-team.irpo;
    location / {
        proxy_pass http://172.16.2.5:8080;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
    }
}
EOF
systemctl restart nginx
nginx -t
echo "Done!"
```

</details>
<details>
  <summary>HQ-RTR</summary>

```
en
conf t
ip nat source static tcp 192.168.1.10 80 172.16.1.4 8080
ip nat source static tcp 192.168.1.10 2026 172.16.1.4 2026
ntp server 172.16.1.4
ntp timezone utc+5
exit
write
```

</details>
<details>
  <summary>BR-RTR</summary>
  
```
en
conf t
ip nat source static tcp 192.168.3.10 8080 172.16.2.5 8080
ip nat source static tcp 192.168.3.10 2026 172.16.2.5 2026
ntp server 172.16.2.5
ntp timezone utc+5
exit
write
```

</details>
<details>
  <summary>HQ-SRV</summary>

```
echo "server=/au-team.irpo/192.168.3.10" >> /etc/dnsmasq.conf
systemctl restart dnsmasq


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


apt-get update
apt-get install -y apache2 php8.2 apache2-mod_php8.2 mariadb-server php8.2-{opcache,curl,gd,intl,mysqli,xml,xmlrpc,ldap,zip,soap,mbstring,json,xmlreader,fileinfo,sodium}
mount -o loop /dev/sr0 /mnt
systemctl enable --now httpd2 mysqld
echo -e "\n\nP@ssw0rd\nP@ssw0rd\ny\ny\ny\ny\ny" | mysql_secure_installation
mariadb -u root -pP@ssw0rd -e "CREATE DATABASE webdb;"
mariadb -u root -pP@ssw0rd -e "CREATE USER 'webc'@'localhost' IDENTIFIED BY 'P@ssw0rd';"
mariadb -u root -pP@ssw0rd -e "GRANT ALL PRIVILEGES ON webdb.* TO 'webc'@'localhost';"
mariadb -u root -pP@ssw0rd -e "FLUSH PRIVILEGES;"
iconv -f UTF-16LE -t UTF-8 /mnt/web/dump.sql > /tmp/dump_utf8.sql
mariadb -u root -pP@ssw0rd webdb < /tmp/dump_utf8.sql
chmod 777 /var/www/html
cp /mnt/web/index.php /var/www/html/
cp /mnt/web/logo.png /var/www/html/
rm -f /var/www/html/index.html
chown apache2:apache2 /var/www/html
sed -i 's/$servername = ".*";/$servername = "localhost";/' /var/www/html/index.php
sed -i 's/$username = ".*";/$username = "webc";/' /var/www/html/index.php
sed -i 's/$password = ".*";/$password = "P@ssw0rd";/' /var/www/html/index.php
sed -i 's/$dbname = ".*";/$dbname = "webdb";/' /var/www/html/index.php
systemctl restart httpd2

apt-get install -y chrony
cat << EOF > /etc/chrony.conf
server 172.16.1.4 iburst prefer
driftfile /var/lib/chrony/drift
logdir /var/log/chrony
EOF
systemctl enable --now chronyd
systemctl restart chronyd
echo "Done!"
```

</details>
<details>
  <summary>BR-SRV</summary>

```
if ! grep -q '^nameserver 8\.8\.8\.8$' /etc/resolv.conf; then
    echo 'nameserver 8.8.8.8' | tee -a /etc/resolv.conf
fi
apt-get update && apt-get install wget dos2unix task-samba-dc -y
sleep 3
echo "nameserver 192.168.1.10" >> /etc/resolv.conf
sleep 2
echo "192.168.3.10 br-srv.au-team.irpo" >> /etc/hosts
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
wget https://raw.githubusercontent.com/sudo-project/sudo/main/docs/schema.ActiveDirectory -O schema.ActiveDirectory
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
ldbsearch -H /var/lib/samba/private/sam.ldb -s base -b 'CN=prava_hq,OU=sudoers,DC=au-team,DC=irpo' 'nTSecurityDescriptor' | sed -n '/^#/d;s/O:DAG:DAD:AI/O:DAG:DAD:AI\(A;\;RPLCRC;\;\;AU\)\(A;\;RPWPCRCCDCLCLORCWOWDSDDTSW;\;\;SY\)/;3,$p' | sed ':a;N;$!ba;s/\n\s//g' | sed -e 's/.\{78\}/&\n /g' >> ntGen.ldif
ldbmodify -v -H /var/lib/samba/private/sam.ldb ntGen.ldif


apt-get install ansible -y
cat << EOF > /etc/ansible/hosts
[servers]
HQ-SRV ansible_host=192.168.1.10
HQ-CLI ansible_host=192.168.2.10
[servers:vars]
ansible_user=remote_user
ansible_port=2026
[routers]
HQ-RTR ansible_host=192.168.1.1
BR-RTR ansible_host=192.168.3.1
[routers:vars]
ansible_user=net_admin
ansible_password=P@ssw0rd
ansible_connection=network_cli
ansible_network_os=ios
EOF
cat << EOF > /etc/ansible/ansible.cfg
[defaults]
ansible_python_interpreter=/usr/bin/python3
interpreter_python=auto_silent
host_key_checking=false
EOF


apt-get install -y chrony
cat << EOF > /etc/chrony.conf
server 172.168.2.5 iburst prefer
driftfile /var/lib/chrony/drift
logdir /var/log/chrony
EOF
systemctl enable --now chronyd
systemctl restart chronyd


apt-get install -y docker-compose docker-engine
systemctl enable --now docker
mount -o loop /dev/sr0
docker load -i /media/ALTLinux/docker/site_latest.tar
docker load -i /media/ALTLinux/docker/mariadb_latest.tar
cat << EOF > /root/site.yml
services:
  db:
    image: mariadb
    container_name: db
    environment:
      DB_NAME: testdb
      DB_USER: test
      DB_PASS: Passw0rd
      MYSQL_ROOT_PASSWORD: Passw0rd
      MYSQL_DATABASE: testdb
      MYSQL_USER: test
      MYSQL_PASSWORD: Passw0rd
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - app_network
    restart: unless-stopped
  testapp:
    image: site
    container_name: testapp
    environment:
      DB_TYPE: maria
      DB_HOST: db
      DB_NAME: testdb
      DB_USER: test
      DB_PASS: Passw0rd
      DB_PORT: 3306
    ports:
      - "8080:8000"
    networks:
      - app_network
    depends_on:
      - db
    restart: unless-stopped
volumes:
  db_data:
networks:
  app_network:
    driver: bridge
EOF
cat << EOF > launch.sh
docker compose -f site.yml up -d
sleep 10
docker exec -it db mysql -u root -pPassw0rd -e "
CREATE DATABASE IF NOT EXISTS testdb;
CREATE USER IF NOT EXISTS 'test'@'%' IDENTIFIED BY 'Passw0rd';
GRANT ALL PRIVILEGES ON testdb.* TO 'test'@'%';
FLUSH PRIVILEGES;"
EOF
chmod +x launch.sh
./launch.sh


ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
ssh-copy-id -o StrictHostKeyChecking=no -p 2026 remote_user@192.168.1.10
ssh-copy-id -o StrictHostKeyChecking=no -p 2026 remote_user@192.168.2.10
ansible all -m ping
echo "Done!"
```

</details>
<details>
  <summary>HQ-CLI</summary>

```
sed -i 's/BOOTPROTO=static/BOOTPROTO=dhcp/' /etc/net/ifaces/ens20/options
systemctl restart network
apt-get update && apt-get install bind-utils -y
system-auth write ad AU-TEAM.IRPO cli AU-TEAM 'administrator' 'P@ssw0rd'
reboot
sleep 3
apt-get install sudo libsss_sudo -y
control sudo public
sed -i '19 a\sudo_provider = ad' /etc/sssd/sssd.conf
sed -i 's/services = nss, pam/services = nss, pam, sudo/' /etc/sssd/sssd.conf
sed -i '28 a\sudoers: files sss' /etc/nsswitch.conf
rm -rf /var/lib/sss/db/*
sss_cache -E
systemctl restart sssd


apt-get update && apt-get install nfs-clients -y
mkdir -p /mnt/nfs
echo "192.168.1.10:/raid/nfs /mnt/nfs nfs intr,soft,_netdev,x-systemd.automount 0 0" >> /etc/fstab
mount -a
mount -v
touch /mnt/nfs/test


useradd remote_user -u 2026
echo -e "P@ssw0rd\nP@ssw0rd" | passwd remote_user
gpasswd -a "remote_user" wheel
sed -i '/WHEEL_USERS.*ALL.*NOPASSWD.*ALL/s/^#//' /etc/sudoers
echo "Authorized access only" > /etc/openssh/banner
sed -i '1i\Port 2026\nAllowUsers remote_user\nMaxAuthTries 2\nPasswordAuthentication yes\nBanner /etc/openssh/banner' /etc/ssh/sshd_config
systemctl enable --now sshd
systemctl restart sshd
sleep 3

apt-get install yandex-browser-stable -y
echo "Done!"
```
