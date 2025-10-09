<span title="### 2-4. Настройка службы сетевого времени"</span>

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
