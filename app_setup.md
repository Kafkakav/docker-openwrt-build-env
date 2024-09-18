## 一些軟體服務的安裝 - Raspberry Pi 與 OpenWRT

### OpenSSH and SFTP - 透過網路上傳檔案
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

### Nginx Server - nginx reverse proxy
[OpenWrt - Nginx]([https://kafkakav.github.io/](https://openwrt.org/docs/guide-user/services/webserver/nginx)).
``` bash
opkg update && opkg install nginx-ssl 

# 原始uci 設定檔, 使用前檢測 config-file有沒有問題
nginx -t -c /etc/nginx/uci.conf   #基本
nginx -T -c /etc/nginx/uci.conf   #詳綑
```

不用uci設定, 恢復原來的nginx.conf使用, 方便移植到其它環境執行
*如果已修改uci.conf, 要先備份, 否則會被清空*
```
uci get nginx.global.uci_enable
uci set nginx.global.uci_enable=false
uci commit nginx
```
修改/etc/nginx/nginx.conf, 以符合OpenWrt執行環境, 目的保時原來的uhttpd 運作, 以及Nginx獨立運作
*修改來自uci.conf的內容*
* changed port 80 => port 81, port 443 => port 444
* removed restrict_locally
* removed http redirect to https
* http ONLY support location /.well-known for ACMME Let's Encrypt access
```
# This file is re-created when Nginx starts.
# Consider using UCI or creating files in /etc/nginx/conf.d/ for configuration.
# Parsing UCI configuration is skipped if uci set nginx.global.uci_enable=false
# For details see: https://openwrt.org/docs/guide-user/services/webserver/nginx
# UCI_CONF_VERSION=1.2

worker_processes 4;
user root;
include module.d/*.module;
events {
	worker_connections 1024;
}

http {
	access_log off;
	log_format openwrt
		'$request_method $scheme://$host$request_uri => $status'
		' (${body_bytes_sent}B in ${request_time}s) <- $http_referer';

	include mime.types;
	default_type application/octet-stream;
	sendfile on;

	client_max_body_size 64M;
	large_client_header_buffers 2 1k;

	gzip on;
	gzip_vary on;
	gzip_proxied any;

 	## Proxy settings
 	proxy_set_header   Host $host:$server_port;
	proxy_set_header   X-Real-IP $remote_addr;
	proxy_set_header   X-Real-PORT $remote_port;
	proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
	proxy_set_header   X-Forwarded-Host $server_name;
 	proxy_set_header   X-Forwarded-Proto $scheme; 

 	## websocket settings
	map $http_upgrade $connection_upgrade {
 		default upgrade;
		'' close;
	}

	root /www;

	server { #see uci show 'nginx._lan'
		listen 444 ssl default_server;
		listen [::]:444 ssl default_server;
		server_name _lan;
		# include restrict_locally;
		include conf.d/*.locations;

		ssl_certificate /etc/nginx/conf.d/_lan.crt;
		ssl_certificate_key /etc/nginx/conf.d/_lan.key;
		# ssl_session_cache shared:SSL:32k;
		# ssl_session_timeout 64m;

		access_log off; # logd openwrt;
		
		include /etc/nginx/options-ssl-nginx.conf;
		location = /robots.txt { access_log off; log_not_found off; }
		location = /favicon.ico { access_log off; log_not_found off; }

	}

	server { #see uci show 'nginx._redirect2ssl'
		listen 81;
		listen [::]:81;
		# server_name _redirect2ssl;
		# return 302 https://$host$request_uri;
		
		location /.well-known/ {
			allow all;
			root /www/acme_wellknown;
		}

		location / {
			#try_files $uri $uri/ /index.html /index.php?$args =404;
			#return 301 https://$host$request_uri;
			# simply close the connection without responding to it.
			return 444;
		}

	}

	include conf.d/*.conf;
}

```

執行nginx server
``` bash
service nginx reload
service nginx restart

```

### appendix A
原始/etc/nginx/uci.conf
* port 80 會直接轉跳 port 443 走https
* self-signed certificates
* 限制存取來源 include restrict_locally;
```
# This file is re-created when Nginx starts.
# Consider using UCI or creating files in /etc/nginx/conf.d/ for configuration.
# Parsing UCI configuration is skipped if uci set nginx.global.uci_enable=false
# For details see: https://openwrt.org/docs/guide-user/services/webserver/nginx
# UCI_CONF_VERSION=1.2

worker_processes auto;

user root;

include module.d/*.module;

events {}

http {
        access_log off;
        log_format openwrt
                '$request_method $scheme://$host$request_uri => $status'
                ' (${body_bytes_sent}B in ${request_time}s) <- $http_referer';

        include mime.types;
        default_type application/octet-stream;
        sendfile on;

        client_max_body_size 64M;
        large_client_header_buffers 2 1k;

        gzip on;
        gzip_vary on;
        gzip_proxied any;

        root /www;

        server { #see uci show 'nginx._lan'
                listen 444 ssl default_server;
                listen [::]:443 ssl default_server;
                server_name _lan;
                include restrict_locally;
                include conf.d/*.locations;
                ssl_certificate /etc/nginx/conf.d/_lan.crt;
                ssl_certificate_key /etc/nginx/conf.d/_lan.key;
                ssl_session_cache shared:SSL:32k;
                ssl_session_timeout 64m;
                access_log off; # logd openwrt;
        }

        server { #see uci show 'nginx._redirect2ssl'
                listen 88;
                listen [::]:80;
                server_name _redirect2ssl;
                return 302 https://$host$request_uri;
        }

        include conf.d/*.conf;
}
```
