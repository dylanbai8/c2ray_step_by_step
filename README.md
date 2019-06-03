## 0.前期准备 （借用域名youdiangan.ga为案例）

域名 IP等配置信息 照猫画虎 自行更改

```
解析域名 “youdiangan.ga” A记录到 “199.187.125.84”
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

添加caddy配置文件：/usr/local/bin/  Caddyfile

```
touch /usr/local/bin/Caddyfile

cat <<EOF > /usr/local/bin/Caddyfile
http://youdiangan.ga:80 {
    root /www/

    proxy /dylanbai8 127.0.0.1:2087 {
        websocket
        header_upstream -Origin
    }
}
EOF
```

添加v2ray配置文件：/etc/v2ray/  config.json

```
touch /etc/v2ray/config.json

cat <<EOF > /etc/v2ray/config.json
{
  "log": {
    "access": "/var/log/v2ray/access.log",
    "error": "/var/log/v2ray/error.log",
    "loglevel": "warning"
  },
  "dns": {},
  "stats": {},
  "inbounds": [
    {
      "port": 2087,
      "protocol": "vmess",
      "settings": {
        "clients": [
          {
            "id": "c1fa2c00-c45c-42bd-8d28-6ee844a88a2d",
            "alterId": 6
          }
        ]
      },
      "tag": "in-0",
      "streamSettings": {
        "network": "ws",
        "security": "none",
        "wsSettings": {
          "path": "/dylanbai8"
        }
      },
      "listen": "127.0.0.1"
    }
  ],
  "outbounds": [
    {
      "tag": "direct",
      "protocol": "freedom",
      "settings": {}
    },
    {
      "tag": "blocked",
      "protocol": "blackhole",
      "settings": {}
    }
  ],
  "routing": {
    "domainStrategy": "AsIs",
    "rules": [
      {
        "type": "field",
        "ip": [
          "geoip:private"
        ],
        "outboundTag": "blocked"
      }
    ]
  },
  "policy": {},
  "reverse": {},
  "transport": {}
}
EOF
```

## 4.设置caddy开机自启动

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
伪装路径(ws path)：/dylanbai8
```

客户端json 一键导入（不同客户端可能需修改inbounds port 如v2rayNG为10808）

```
{
  "log": {},
  "dns": {},
  "stats": {},
  "inbounds": [
    {
      "port": "1080",
      "protocol": "socks",
      "settings": {
        "auth": "noauth",
        "udp": true
      },
      "tag": "in-0"
    },
    {
      "port": "1081",
      "protocol": "http",
      "settings": {},
      "tag": "in-1"
    }
  ],
  "outbounds": [
    {
      "protocol": "vmess",
      "settings": {
        "vnext": [
          {
            "address": "youdiangan.ga",
            "port": 80,
            "users": [
              {
                "id": "c1fa2c00-c45c-42bd-8d28-6ee844a88a2d",
                "alterId": 6
              }
            ]
          }
        ]
      },
      "tag": "out-0",
      "streamSettings": {
        "network": "ws",
        "security": "none",
        "wsSettings": {
          "path": "/dylanbai8"
        }
      }
    },
    {
      "tag": "direct",
      "protocol": "freedom",
      "settings": {}
    },
    {
      "tag": "blocked",
      "protocol": "blackhole",
      "settings": {}
    }
  ],
  "routing": {
    "domainStrategy": "IPOnDemand",
    "rules": [
      {
        "type": "field",
        "ip": [
          "geoip:private"
        ],
        "outboundTag": "direct"
      }
    ]
  },
  "policy": {},
  "reverse": {},
  "transport": {}
}
```

## 8.附录一

```
1.为减轻服务器负担 额外ID仅设置为6，如有必要可以酌情配置

2.为减轻服务器负担 未使用底层tls 及ssl加密，如有必要 可修改为443端口 在 root /www/ 下一行加入 tls admin@youdiangan.ga
并在caddy.service内修改 ExecStart=/usr/local/bin/caddy -conf=/usr/local/bin/Caddyfile -agree=true -ca=https://acme-v02.api.letsencrypt.org/directory
客户端json文件亦需修改：
"security": "tls",
path（不要忘记逗号）下一行添加
"tlsSettings": {
"serverName": "youdiangan.ga"
}
此次不再详述

3.如有必要安装php+sqlite3环境 执行以下代码：

安装源
参照：https://blog.csdn.net/yueqifeng_812/article/details/79045670

安装程序
apt install php7.0-cgi php7.0-fpm php7.0-curl php7.0-gd php7.0-mbstring php7.0-xml php7.0-sqlite3 sqlite3 -y
systemctl enable php7.0-fpm
systemctl restart php7.0-fpm

给予权限
groupadd www-data
useradd --shell /sbin/nologin -g www-data www-data
chown www-data:www-data -R /www/*
chmod -R 777 /www/

修改caddy配置文件 在}前加入：
fastcgi / /run/php/php7.0-fpm.sock php
```

## 9.附录二

将客户端json写入伪装网站内 方便分享

```
自定义目录 起到简单加密的效果
mkdir /www/mima888888

写入配置文件（同上文 7.客户端配置）
touch /www/mima888888/v2ray.json

cat <<EOF > /www/mima888888/v2ray.json
{
  "log": {},
  "dns": {},
  "stats": {},
  "inbounds": [
    {
      "port": "1080",
      "protocol": "socks",
      "settings": {
        "auth": "noauth",
        "udp": true
      },
      "tag": "in-0"
    },
    {
      "port": "1081",
      "protocol": "http",
      "settings": {},
      "tag": "in-1"
    }
  ],
  "outbounds": [
    {
      "protocol": "vmess",
      "settings": {
        "vnext": [
          {
            "address": "youdiangan.ga",
            "port": 80,
            "users": [
              {
                "id": "c1fa2c00-c45c-42bd-8d28-6ee844a88a2d",
                "alterId": 6
              }
            ]
          }
        ]
      },
      "tag": "out-0",
      "streamSettings": {
        "network": "ws",
        "security": "none",
        "wsSettings": {
          "path": "/dylanbai8"
        }
      }
    },
    {
      "tag": "direct",
      "protocol": "freedom",
      "settings": {}
    },
    {
      "tag": "blocked",
      "protocol": "blackhole",
      "settings": {}
    }
  ],
  "routing": {
    "domainStrategy": "IPOnDemand",
    "rules": [
      {
        "type": "field",
        "ip": [
          "geoip:private"
        ],
        "outboundTag": "direct"
      }
    ]
  },
  "policy": {},
  "reverse": {},
  "transport": {}
}
EOF
```

使用：打开手机客户端v2rayNG 右上角+号-自定义配置-剪贴板url导入自定义配置

url路径：http://youdiangan.ga/mima888888/v2ray.json

## 10.附录三

```
关闭apache2
systemctl stop apache2
systemctl disable apache2
```