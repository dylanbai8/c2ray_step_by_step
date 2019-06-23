## 0.前期准备

为尽可能的节省服务器资源 此教程为（80端口、无底层ssl加密）示例

```
V2ray 的 WebSocket+TLS+Web 配置需要有域名配合才能搭建使用

你可以使用本站提供的临时域名：
如服务器IP为 103.1.14.203 临时域名即 103-1-14-203.ip.c2ray.ml

也可以使用自有域名如：
解析域名 “youdiangan.ga” A记录到 “103.1.14.203”
或者解析域名 “youdiangan.ga” AAAA记录到 “2607:2200:0:2347:0:3353:e1:1a2b”

(AAAA记录可以配合cloudflare转换为ip4)
```

## 1.VPS安装Debian8 更新系统 安装必要软件

```
apt update -y

apt install curl -y
```

## 2.执行官方脚本 安装caddy和v2ray

```
curl https://getcaddy.com | bash -s personal

bash <(curl -L -s https://install.direct/go.sh)
```

## 3.配置caddy和v2ray

添加caddy配置文件：/usr/local/bin/  [Caddyfile]

```
touch /usr/local/bin/Caddyfile

cat <<EOF > /usr/local/bin/Caddyfile
103-1-14-203.ip.c2ray.ml:80 {
root /www
gzip
index index.html
}
outlook.live.com:80 {
proxy / localhost:10000 {
        websocket
        header_upstream Connection {>Connection}
        header_upstream Upgrade {>Upgrade}
        header_upstream Host "103-1-14-203.ip.c2ray.ml"
    }
}
EOF
```

添加v2ray配置文件：/etc/v2ray/  [config.json]

```
touch /etc/v2ray/config.json

cat <<EOF > /etc/v2ray/config.json
{
  "inbounds": [
    {
      "port": 10000,
      "listen":"127.0.0.1",
      "protocol": "vmess",
      "settings": {
        "clients": [
          {
            "id": "b831381d-6324-4d53-ad4f-8cda48b30811",
            "alterId": 64
          }
        ]
      },
      "streamSettings": {
        "network": "ws",
        "wsSettings": {
        "path": "/ray"
        }
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "settings": {}
    }
  ]
}
EOF
```

## 4.设置caddy开机自启动

添加开机自启动文件：/etc/systemd/system/  [caddy.service]
```
touch /etc/systemd/system/caddy.service

cat <<EOF > /etc/systemd/system/caddy.service
[Unit]
Description=Caddy_Server
After=network.target
Wants=network.target
[Service]
Type=simple
ExecStart=/usr/local/bin/caddy -conf /usr/local/bin/Caddyfile
RestartPreventExitStatus=23
Restart=always
User=root
[Install]
WantedBy=multi-user.target
EOF

systemctl enable caddy
```

## 5.添加伪装网站

```
mkdir /www/

touch /www/index.html

cat <<EOF > /www/index.html
It's a little awkward!
EOF
```

## 6.重启caddy和v2ray 载入配置文件

```
systemctl daemon-reload

systemctl restart caddy

systemctl restart v2ray
```

## 7.客户端配置

手动配置 [推荐]

```
地址(address)：youdiangan.ga
端口(port)：80
用户ID(id)：c1fa2c00-c45c-42bd-8d28-6ee844a88a2d
额外ID(alterid)：6
传输协议(network)：ws
伪装路径(ws path)：/ray
```

客户端json 一键导入（不同客户端可能需修改inbounds port 如v2rayNG为10808）

```
{
  "inbounds": [
    {
      "port": 1080,
      "listen": "127.0.0.1",
      "protocol": "socks",
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls"]
      },
      "settings": {
        "auth": "noauth",
        "udp": false
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "vmess",
      "settings": {
        "vnext": [
          {
            "address": "mydomain.me",
            "port": 443,
            "users": [
              {
                "id": "b831381d-6324-4d53-ad4f-8cda48b30811",
                "alterId": 64
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "ws",
        "security": "tls",
        "wsSettings": {
          "path": "/ray"
        }
      }
    }
  ]
}
```



## 可能用到的命令

```
关闭apache2
systemctl stop apache2
systemctl disable apache2

关闭SElinux
setsebool -P httpd_can_network_connect 1

配置文件参照
https://toutyrater.github.io/advanced/wss_and_web.html

```