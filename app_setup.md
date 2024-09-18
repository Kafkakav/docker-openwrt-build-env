## 一些軟體服務的安裝 - Raspberry Pi 與 OpenWRT

### OpenSSH and SFTP
``` bash
# step 0
opkg update
opkg install openssh-server openssh-sftp-server
opkg install shadow-useradd shadow-groupadd shadow-userdel shadow-groupdel shadow-groupmod shadow-usermod

# step 1
mkdir /home
mkdir /home/sftpuser

useradd adminuser
passwd adminuser
useradd appuser
passwd appuser

usermod -s /bin/sh adminuser
usermod -s /bin/false appuser

groupadd sftpadmin
groupadd sftpuser
usermod -G sftpadmin adminuser
usermod -G sftpuser appuser

chown -R root:root /home/sftpuser
chmod 755 /home/sftpuser
mkdir /home/sftpuser/appuser
chown -R appuser:appuser /home/sftpuser/appuser

/home/sftpuser/appuser

```
安裝 sudo套件
Edit /etc/sudoers to let any user 'sudo' with root password
```
opkg install sudo
visudo
#改這二行
Defaults targetpw  # Ask for the password of the target user
ALL ALL=(ALL) ALL  # WARNING: only use this together with 'Defaults targetpw'
```
修改Openssh 設定檔 /etc/ssh/sshd_config, 因為port 22已被佔用, 另綁一個port: 2222 專門給sftp用
```
Port 2222
LogLevel INFO

AuthorizedKeysFile      .ssh/authorized_keys

PermitEmptyPasswords no
PermitRootLogin no
StrictModes no

AllowTcpForwarding yes
GatewayPorts yes
X11Forwarding no
PermitTTY no
ClientAliveInterval 300
ClientAliveCountMax 3
UseDNS no
PidFile /var/run/sshd.pid

# override default of no subsystems
Subsystem       sftp    /usr/lib/sftp-server

#群組設定
Match Group sftpuser
        ForceCommand internal-sftp
        ChrootDirectory %h
        AllowTcpForwarding no
        PermitTunnel no
        X11Forwarding no
        AllowAgentForwarding no

# 單一使用者設定
#Match User appuser
#        ForceCommand internal-sftp
#        ChrootDirectory /home/sftpuser/appuser
#        AllowTcpForwarding no
#        PermitTunnel no
#        X11Forwarding no
#        AllowAgentForwarding no
```

執行sshd
```
/etc/init.d/sshd enable
/etc/init.d/sshd start
/etc/init.d/sshd restart

```

(非心要)這裡不停用系統中的dropbear, 讓它提供ssh login服務, 如果要停用, 必須另外處理Openssh提供登入的設定
```
/etc/init.d/dropbear disable
/etc/init.d/dropbear stop

```

### Nginx Server
```
```
