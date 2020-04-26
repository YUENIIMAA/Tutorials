# Optimize & Secure a Linux Server

> * Based on `Ubuntu 18.04`
>
> * Replace `<xxx>` with your own data
>
> * References:
>
>   [How to Secure Your Server](https://www.linode.com/docs/security/securing-your-server/)
>

## Step I. Add a Limited User Account

Skip this step if you've already had a limited user account

otherwise I suppose your server only has a root account, add a limited user account first:

```
adduser <new-user>
```

Add the user to the `sudo` group so it'll have administrative privileges:

```
adduser <new-user> sudo
```

## Step II. Harden SSH Access

On your local machine, create an authentication key:

> * If you've already created a key-pair, skip this command
> * For `Windows 7` users, I recommend using `Git` command line tool
> * For `Windows 10` users, I recommend using `Windows PowerShell`
> * You should not leave the passphrase blank

```
ssh-keygen -t ed25519
```

Upload the public key to your server:

```
ssh-copy-id <new-user>@<server ip>
```

Now you should be able to access your server without using the password of your `<new-user>`.

## Step III. Disable Root Login and Password Authentication

On your server, modify the following file:

```
sudo nano /etc/ssh/sshd_config
```

Remove `#` and set the value to `no`:

```
PermitRootLogin no
PasswordAuthentication no
```

Restart SSH service:

```
sudo systemctl restart sshd
```

## Step IV. Enable Firewall

Allow all outgoing traffic and disable all incoming traffic first:

```
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Set `allow` rules according to the service running on your server:

```
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https
```

You should at least allow `ssh`, otherwise you'll be blocked by your server.

Enable firewall:

```
sudo ufw enable
```

## Step V. Upgrade Your System

```
sudo apt update
sudo apt dist-upgrade
sudo apt autoremove
```

## Step VI. Optimize Configuration

Modify `/etc/sysctl.conf`, add the following content to the end of the file:

```
# max open files
fs.file-max = 51200
# max read buffer
net.core.rmem_max = 67108864
# max write buffer
net.core.wmem_max = 67108864
# default read buffer
net.core.rmem_default = 65536
# default write buffer
net.core.wmem_default = 65536
# max processor input queue
net.core.netdev_max_backlog = 4096
# max backlog
net.core.somaxconn = 4096
# resist SYN flood attacks
net.ipv4.tcp_syncookies = 1
# reuse timewait sockets when safe
net.ipv4.tcp_tw_reuse = 1
# turn off fast timewait sockets recycling
net.ipv4.tcp_tw_recycle = 0
# short FIN timeout
net.ipv4.tcp_fin_timeout = 30
# short keepalive time
net.ipv4.tcp_keepalive_time = 1200
# outbound port range
net.ipv4.ip_local_port_range = 10000 65000
# max SYN backlog
net.ipv4.tcp_max_syn_backlog = 4096
# max timewait sockets held by system simultaneously
net.ipv4.tcp_max_tw_buckets = 5000
# turn on TCP Fast Open on both client and server side
net.ipv4.tcp_fastopen = 3
# TCP receive buffer
net.ipv4.tcp_rmem = 4096 87380 67108864
# TCP write buffer
net.ipv4.tcp_wmem = 4096 65536 67108864
# turn on path MTU discovery
net.ipv4.tcp_mtu_probing = 1
# for high-latency network
net.ipv4.tcp_congestion_control = bbr
```

Modify `/etc/profile`, add the following content to the end of the file:

```
ulimit -n 8192
```

Save changes and reboot:

```
sudo sysctl --system
sudo reboot
```

**Congratulations! You have secured and optimized your server!**

**If you want to further enhance the security of your server, please check [this](https://github.com/imthenachoman/How-To-Secure-A-Linux-Server) out.**

