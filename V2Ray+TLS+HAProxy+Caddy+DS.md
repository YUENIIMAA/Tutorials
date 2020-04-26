# V2Ray+TLS+HAProxy+Caddy+DS

> * 参考：
>
>   [新 V2Ray 白话文指南](https://guide.v2fly.org/advanced/tcp_tls_web.html)
>
>   [HaProxy 通过 Domain Socket 与 V2Ray 通讯](https://gist.github.com/liberal-boy/b2d5597285b4202b6d607faaa1078d27)
>
>   [关于一键安装脚本安全优化的一点小小建议](https://github.com/v2ray/v2ray-core/issues/1011)

## 前言

本教程实现了：

> * 利用`HAProxy`反向代理隐藏`V2Ray`服务
> * 利用`Caddy`获取证书并提供`Web`服务
> * 使用`TCP`模式，对比`WS`和`HTTP2`有蜜汁性能加成
> * 使用`Domain Socket`在`HAProxy`和`VRay`间通讯

在开始前，你需要：

> * 运行`Ubuntu 18.04`的服务器（别的系统也行，不过指令可能略有不同）
> * 域名（必须要有，可在`Freenom`免费申请）
> * 提升服务器安全性，尽量榨干性能（参考另一篇教程：`Optimize & Secure a Linux Server`）

## Step I. 安装Caddy

使用官方脚本一键安装`Caddy`：

```
#此脚本安装的是Caddy的1.X版本，2.0已推出但目前没有一键脚本
sudo curl https://getcaddy.com | bash -s personal
```

配置root拥有二进制文件防止其他账户修改：

```
sudo chown root:root /usr/local/bin/caddy
```

修改权限为755，`root`可读写执行，其他账户不可写：

```
sudo chmod 755 /usr/local/bin/caddy
```

使用`setcap`允许`Caddy`作为用户进程绑定低号端口（服务器需要80和443）：

```
sudo setcap 'cap_net_bind_service=+ep' /usr/local/bin/caddy
```

创建文件夹存储`Caddy`的配置文件：

```
sudo mkdir /etc/caddy
```

创建文件夹存储`Caddy`所管理的站点证书：

```
sudo mkdir /etc/ssl/caddy
```

允许`root`及`www-data`组访问相关文件，允许`Caddy`写入站点证书文件夹：

```
sudo chown -R root:www-data /etc/caddy
sudo chown -R root:www-data /etc/ssl/caddy
sudo chmod 0770 /etc/ssl/caddy
```

如果默认站点根目录不存在，创建以下文件夹：

```
sudo mkdir /var/www
```

允许`www-data`组拥有站点文件夹：

```
sudo chown www-data:www-data /var/www
```

创建空的`Caddy`配置文件：

```
sudo touch /etc/caddy/Caddyfile
```

配置`systemd`服务：

**【更新】Caddy因版本升级，官方Repo中的文件已不再适用于1.X版本，此处提供我的备份**

```
sudo curl -s https://raw.githubusercontent.com/YUENIIMAA/Tutorials/master/Attachments/caddy.service -o /etc/systemd/system/caddy.service
```

调整权限使其只可被`root`修改：

```
sudo chmod 644 /etc/systemd/system/caddy.service
```

重载`systemd`使其检测到新安装的`Caddy`服务：

```
sudo systemctl daemon-reload
```

**验证`Caddy`是否安装成功：**

```
sudo systemctl status caddy
#看到如下输出表示服务注册成功
	caddy.service - Caddy HTTP/2 web server
	Loaded: loaded (/etc/systemd/system/caddy.service; disabled; vendor preset: enabled)
	Active: inactive (dead)
	Docs: https://caddyserver.com/docs
```

## Step II. 配置Caddy

先创建一个站点文件：

```
sudo touch /var/www/index.html
```

编辑该文件：

```
sudo nano /var/www/index.html
```

假如你没有预先写好的站点文件，可以粘贴以下内容保存：

```html
<!DOCTYPE html>
<html>
  <head>
    <title>社会主义核心价值观</title>
  </head>
  <body>
	<div align='center'>
		<h1>社会主义核心价值观</h1>
		<h2>富强  民主  文明  和谐</h2>
		<h2>自由  平等  公正  法治</h2>
		<h2>爱国  敬业  诚信  友善</h2>
	</div>
  </body>
</html>
```

**注：这个页面将成为直接访问域名后看到的页面，保持这个状态是可行的，但是一个只有简单页面的网站每个月流量几十GB感觉还是很可疑的，建议后期自行写一个好看点的网站覆盖。**

编辑`Caddy`配置文件：

```
sudo nano /etc/caddy/Caddyfile
```

粘贴如下内容保存（`<example.com>`替换成服务器的域名，`<user@example.com>`替换成你的邮箱，`<ip-address>`替换成服务器IP地址，以防万一还是要多嘴一句尖括号也是要被替换掉内容的一部分）：

```
:8080 {
    root /var/www/
}
 
https://<example.com>:8443 {
    tls <user@example.com> {
        ciphers ECDHE-ECDSA-WITH-CHACHA20-POLY1305 ECDHE-ECDSA-AES256-GCM-SHA384 ECDHE-ECDSA-AES256-CBC-SHA
        curves p384
        key_type p384
    }
}

http://<example.com> {
    redir https://<example.com>{url}
}

:80 {
    redir https://<example.com>{url}
}
```

手动切换到配置文件所在文件夹：

```
cd /etc/caddy
```

以命令行方式启动`Caddy`：

```
sudo systemctl stop caddy
caddy
```

第一次启动采用命令行启动的方式时因为需要接受`Let’s Encrypt`的协议，一路同意过去就行了，当看到`done`表示证书获取成功，此时进程并不会退出，可以`Ctrl + C`终止。

如需验证`HTTPS`，全部配置完毕可到[SSL Labs](https://www.ssllabs.com/)检测一下评分，正常情况下可以获得`A`的评价，如需提升到`A+`请看另一篇教程。

## Step III. 安装V2Ray

获取并执行一键安装脚本：

```
wget https://install.direct/go.sh
#如需更新，再次执行go.sh即可
sudo bash go.sh
```

**注：首次安装完毕后会打印的UUID是随机生成的可以直接使用**

## Step IV. 配置V2Ray

配置`v2ray`用户和`Domain Socket`文件：

```
sudo mkdir /var/lib/haproxy/v2ray
sudo useradd v2ray -s /usr/sbin/nologin
sudo chown -R v2ray:v2ray /var/log/v2ray
sudo chown -R v2ray:v2ray /var/lib/haproxy/v2ray
```

编辑`service`文件：

```
sudo nano /etc/systemd/system/v2ray.service
```

如果系统不是`Ubuntu 18.04`，`rm`等工具的路径可能和下面配置里写的不一样：

```
[Unit]
Description=V2Ray - A unified platform for anti-censorship
Documentation=https://v2ray.com https://guide.v2fly.org
After=network.target nss-lookup.target
Wants=network-online.target

[Service]
# If the version of systemd is 240 or above, then uncommenting Type=exec and commenting out Type=simple
#Type=exec
Type=simple
# Runs as root or add CAP_NET_BIND_SERVICE ability can bind 1 to 1024 port.
# This service runs as root. You may consider to run it as another user for security concerns.
# By uncommenting User=v2ray and commenting out User=root, the service will run as user v2ray.
# More discussion at https://github.com/v2ray/v2ray-core/issues/1011
#User=root
User=v2ray
CapabilityBoundingSet=CAP_NET_BIND_SERVICE CAP_NET_RAW
NoNewPrivileges=yes

ExecStartPre=/bin/rm -rf /var/lib/haproxy/v2ray/*.sock

ExecStart=/usr/bin/v2ray/v2ray -config /etc/v2ray/config.json

ExecStartPost=/bin/sleep 1
ExecStartPost=/bin/chmod 777 /var/lib/haproxy/v2ray/vmess.sock

Restart=on-failure
# Don't restart in the case of configuration error
RestartPreventExitStatus=23

[Install]
WantedBy=multi-user.target
```

编辑配置文件：

```
sudo nano /etc/v2ray/config.json
```

增加`Domain Socket`的部分（建议先在本地编辑好，到[这里](https://www.json.cn/)检验`JSON`文件正确性，然后再复制回去）：

```
{
  "log": {
    "access": "/var/log/v2ray/access.log",
    "loglevel": "warning",
    "error": "/var/log/v2ray/error.log"
  },
  "inbounds": [{
    "listen": "127.0.0.1",
    "port": 1080,
    "protocol": "vmess",
    "settings": {
      "clients": [
        {
          "id": "<uuid>",
          "level": 1,
          "alterId": 64
        }
      ]
    },
    "streamSettings": {
      "network": "ds",
      "dsSettings": {
        "path": "/var/lib/haproxy/v2ray/vmess.sock"
      }
    }
  }],
  "outbounds": [{
    "protocol": "freedom",
    "settings": {}
  },{
    "protocol": "blackhole",
    "settings": {},
    "tag": "blocked"
  }],
  "routing": {
    "rules": [
      {
        "type": "field",
        "ip": ["geoip:private"],
        "outboundTag": "blocked"
      }
    ]
  }
}
```

`<uuid>`如果不想用默认生成的，可以访问[这个](https://www.uuidgenerator.net/)网站。

## Step V. 安装HAProxy

`Ubuntu 18.04`官方仓库自带的`HAProxy`版本太低了所以要先手动添加仓库：

```
sudo add-apt-repository ppa:vbernat/haproxy-1.8
sudo apt update
sudo apt install haproxy
```

## Step VI. 配置HAProxy

`Caddy`搞到的是`crt`和`key`文件但是`HAProxy`要的是`pem`文件所以要先合成证书：

```
sudo cat /etc/ssl/caddy/acme/acme-v02.api.letsencrypt.org/sites/<example.com>/<example.com>.crt /etc/ssl/caddy/acme/acme-v02.api.letsencrypt.org/sites/<example.com>/<example.com>.key | sudo tee /etc/ssl/private/<example.com>.pem
```

然后编辑配置文件：

```
sudo nano /etc/haproxy/haproxy.cfg
```

修改内容如下：

```
global
        log /dev/log    local0
        log /dev/log    local1 notice
        chroot /var/lib/haproxy
        stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
        stats timeout 30s
        user haproxy
        group haproxy
        daemon

        # Default SSL material locations
        ca-base /etc/ssl/certs
        crt-base /etc/ssl/private

        # Default ciphers to use on SSL-enabled listening sockets.
        # For more information, see ciphers(1SSL). This list is from:
        #  https://hynek.me/articles/hardening-your-web-servers-ssl-ciphers/
        # An alternative list with additional directives can be obtained from
        #  https://mozilla.github.io/server-side-tls/ssl-config-generator/?server=haproxy
        # ssl-default-bind-ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:RSA+AESGCM:RSA+AES:!aNULL:!MD5:!DSS
        # ssl-default-bind-options no-sslv3
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options no-sslv3 no-tlsv10 no-tlsv11
        tune.ssl.default-dh-param 2048

defaults
        log     global
        mode    tcp
        option  dontlognull
        timeout connect 5s
        timeout client  300s
        timeout server  300s
        errorfile 400 /etc/haproxy/errors/400.http
        errorfile 403 /etc/haproxy/errors/403.http
        errorfile 408 /etc/haproxy/errors/408.http
        errorfile 500 /etc/haproxy/errors/500.http
        errorfile 502 /etc/haproxy/errors/502.http
        errorfile 503 /etc/haproxy/errors/503.http
        errorfile 504 /etc/haproxy/errors/504.http

frontend gateway
        bind 0.0.0.0:443 tfo ssl crt /etc/ssl/private/<example.com>.pem
        tcp-request inspect-delay 5s
        tcp-request content accept if HTTP
        use_backend web if HTTP
        default_backend proxy

backend web
        mode http
        server caddy 127.0.0.1:8080

backend proxy
        server v2ray /v2ray/vmess.sock
```

## Step VII. 收尾

因为修改了`service`文件所以要重载一下：

```
sudo systemctl daemon-reload
```

然后从后往前启动：

```
sudo systemctl start v2ray
sudo systemctl start caddy
sudo systemctl start haproxy
```

如果有报错，根据报错处理完之后设置开机启动：

```
sudo systemctl enable v2ray
sudo systemctl enable caddy
sudo systemctl enable haproxy
```

然后就可以愉快地科学上网了。