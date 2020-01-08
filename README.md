# domoticz

Install Raspian (here, we use the official Raspbian Buster)

Update Raspbian
```bash
root# apt update
root# apt upgrade -y
```

Switch some directory in tmpfs
```bash
root# cat >>/etc/fstab <<EOF
tmpfs    /tmp        tmpfs    defaults,noatime,nosuid,size=32m    0 0
tmpfs    /var/tmp    tmpfs    defaults,noatime,nosuid,size=32m    0 0
tmpfs    /var/log    tmpfs    defaults,noatime,nosuid,mode=0755,size=32m    0 0
```

Add a user "domoticz", allow the user to connect to the Zwave.me dongle and create a password
```bash
root# useradd -m -d /opt/domoticz --system domoticz
root# usermod -a -G dialout domoticz
root# passwd domoticz
```

Reboot to use the upgraded packages and use tmpfs directories (the lazy way)
```bash
root# shutdown -r now
```

Allow domoticz to become root (for the installer)
```bash
root# echo 'domoticz ALL=(ALL) NOPASSWD: ALL' >/etc/sudoers.d/020_nopasswd
```

Disconnect and reconnect with the domoticz user (important, do not "su - domoticz" from root), then install domoticz
```bash
domoticz$ mkdir release
domoticz$ cd release
domoticz$ curl -sSL install.domoticz.com | sudo bash
```

Go back under root, then replace the cheap init script with a wonderful systemd one
```bash
root# cat >/etc/systemd/system/domoticz.service <<EOF
[Unit]
  Description=domoticz
  After=time-sync.target
[Service]
  User=domoticz
  Group=domoticz
  ExecStart=/opt/domoticz/release/domoticz -www 8080 -sslwww 8443
  WorkingDirectory=/opt/domoticz/release
  Restart=on-failure
  RestartSec=10
[Install]
  WantedBy=multi-user.target
EOF

root# systemctl daemon-reload
root# systemctl enable domoticz
root# systemctl start domoticz
```

Setup iptables (basic, only cover IPv4)
```bash
root# apt install -y iptables-persistent
root# cat >/etc/iptables/rules.v4 <<EOF
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A PREROUTING -p tcp -m tcp --dport 80 -j REDIRECT --to-ports 8080
-A PREROUTING -p tcp -m tcp --dport 443 -j REDIRECT --to-ports 8443
COMMIT

*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT DROP [0:0]
-A INPUT -i lo -j ACCEPT
-A INPUT -p icmp --icmp-type echo-request -j ACCEPT
-A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A OUTPUT -m conntrack --ctstate NEW,RELATED,ESTABLISHED -j ACCEPT

-A INPUT -p tcp -m tcp -m conntrack --ctstate NEW --dport 22 -j ACCEPT
-A INPUT -p tcp -m tcp -m conntrack --ctstate NEW --dport 8080 -j ACCEPT
-A INPUT -p tcp -m tcp -m conntrack --ctstate NEW --dport 8443 -j ACCEPT

COMMIT
EOF
```

Reboot the ensure everything boot up as expected
```bash
root# shutdown -r now
```

Remove the temporary sudo rule
```bash
root# rm -f /etc/sudoers.d/020_nopasswd
```

You're done
