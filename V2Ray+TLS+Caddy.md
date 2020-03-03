# V2Ray+TLS+Caddy

> * I wrote this in Chinese because I think this tutorial is super useful to Chinese people.
>
> * 参考：
>
>   [从零开始在Ubuntu上搭建 V2Ray h2 + tls + web 代理服务](https://canmipai.com/index.php/2018/06/28/v2ray_h2_web_tutorial/)
>
>   [WebSocket+TLS+Web](https://toutyrater.github.io/advanced/wss_and_web.html)
>
>   [caddy官方脚本一键安装与使用]([https://medium.com/@jestem/caddy%E5%AE%98%E6%96%B9%E8%84%9A%E6%9C%AC%E4%B8%80%E9%94%AE%E5%AE%89%E8%A3%85%E4%B8%8E%E4%BD%BF%E7%94%A8-1e6d25154804](https://medium.com/@jestem/caddy官方脚本一键安装与使用-1e6d25154804))

## 前言

本教程实现了：

> * 利用`Caddy`反向代理隐藏`V2Ray`服务
> * ~~`HTTP/2`和`WebSocket`两种协议的支持（支持后者是因为iOS平台的`Shadowrocket`不支持前者）~~
> * 移除`HTTP/2`，因未知原因在我的使用环境中性能变得非常糟糕

在开始前，你需要：

> * 运行`Ubuntu 18.04`的服务器（别的系统也行，不过指令可能略有不同）
> * 域名（必须要有，可在`Freenom`免费申请）
> * 提升服务器安全性，尽量榨干性能（参考另一篇教程：`Optimize & Secure a Linux Server`）

## Step I. 安装Caddy

使用官方脚本一键安装`Caddy`：

```
#如需更新，再次执行该命令即可（其他的命令需不要执行第二次）
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
sudo curl -s  [to replace]  -o /etc/systemd/system/caddy.service
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

## Step II. 配置Caddy和HTTPS

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

粘贴如下内容保存（`<example.com>`替换成指向服务器地址的域名，`<user@example.com>`替换成邮箱，后文的`<ip-address>`替换成服务器IP地址，以防万一还是要多嘴一句尖括号也是要被替换掉内容的一部分）：

```
http://<example.com> {
    redir https://<example.com>{url}
}
 
https://<example.com> {
    root /var/www/
    gzip
 
    tls <user@example.com> {
        ciphers ECDHE-ECDSA-WITH-CHACHA20-POLY1305 ECDHE-ECDSA-AES256-GCM-SHA384 ECDHE-ECDSA-AES256-CBC-SHA
        curves p384
        key_type p384
    }
 
    header / {
        Strict-Transport-Security "max-age=31536000;"
        X-XSS-Protection "1; mode=block"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "DENY"
    }
}
```

手动切换到配置文件所在文件夹：

```
cd /etc/caddy
```

以命令行方式启动`Caddy`：

```
#假如你很心急已经启动过Caddy，需先结束Caddy进程
#sudo systemctl stop caddy
caddy
```

第一次启动采用命令行启动的方式时因为需要接受`Let’s Encrypt`的协议，一路同意过去就行了，当看到`done`表示证书获取成功，此时进程并不会退出，可以`Ctrl + C`终止然后通过`Systemd`重启`Caddy`：

```
sudo systemctl start caddy
#如需开机自启，将start换成enable执行一次即可，如需重启，将start换成restart
```

如需验证`HTTPS`配置可到[SSL Labs](https://www.ssllabs.com/)检测一下评分，正常情况下可以获得满分`A+`的评价。

## Step III. 安装与配置V2Ray

获取并执行一键安装脚本：

```
wget https://install.direct/go.sh
# 如需更新，再次执行go.sh即可
sudo bash go.sh
```

**注：首次安装完毕后会打印一串随机生成的UUID，建议保存下来，之后配置文件里可以用到**

编辑配置文件：

```
sudo nano /etc/v2ray/config.json
```

删光里面的内容替换成下面这些（替换尖括号内容，建议先在本地编辑好，到[这里](https://www.json.cn/)检验`JSON`文件正确性，然后再复制回去）：

```
{
  "log": {
    "access": "/var/log/v2ray/access.log",
    "loglevel": "warning",
    "error": "/var/log/v2ray/error.log"
  },
  "inbounds": [{
    "listen": "127.0.0.1",
    "port": <websocket_port>,
    "protocol": "vmess",
    "settings": {
      "udp": true,
      "clients": [
        {
          "id": "<uuid>",
          "level": 1,
          "alterId": 64
        }
      ]
    },
    "streamSettings": {
      "network": "ws",
      "wsSettings": {
        "path": "/<websocket_path>"
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

其中`<websocket_port>`、`<websocket_path>`要和后文`Caddyfile`中的一致，假如想要其它的`<uuid>`，可以访问[这个](https://www.uuidgenerator.net/)网站。

最后再次编辑`Caddyfile`：

```
http://<example.com> {
    redir https://<example.com>{url}
}

https://<example.com> {
    root /var/www/

    tls <user@example.com> {
        ciphers ECDHE-ECDSA-WITH-CHACHA20-POLY1305 ECDHE-ECDSA-AES256-GCM-SHA384 ECDHE-ECDSA-AES256-CBC-SHA
        curves p384
        key_type p384
    }

    proxy /<websocket_path> http://127.0.0.1:<websocket_port> {
        websocket
        header_upstream -Origin
    }

    header / {
        Strict-Transport-Security "max-age=31536000;"
        X-XSS-Protection "1; mode=block"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "DENY"
    }
}

http://<ip-address> {
    redir https://<example.com>{url}
}
```

启动`V2Ray`并重启`caddy`，通过实际连接测试配置的情况，一切顺利的话就可以把两个服务设置成开机自启的了。