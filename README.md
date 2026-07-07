# Hysteria协议安装

- Project Heysteria：https://github.com/apernet/hysteria
- Hysteria Offical Blog: https://v2.hysteria.network

## 服务端部署
- 更新系统

Debian
```
sudo apt update && sudo apt upgrade -y
```

- 安装 Hysteria

切换到 root
```
sudo -i
```
Official Script 
```
bash <(curl -fsSL https://get.hy2.sh/)
```

Modify
```
bash <(curl -fsSL https://raw.githubusercontent.com/Felix-zf/Hysteria-Scripts/main/hysteria.sh)
```

## SSL证书申请
- 自签证书
```
openssl req -x509 -nodes -newkey ec:<(openssl ecparam -name prime256v1) -keyout /etc/hysteria/server.key -out /etc/hysteria/server.crt -subj "/CN=bing.com" -days 36500 && sudo chown hysteria /etc/hysteria/server.key && sudo chown hysteria /etc/hysteria/server.crt
```
- CA证书
```
wget -N --no-check-certificate https://raw.githubusercontent.com/Felix-zf/ACME-Scripts/main/acme.sh && bash acme.sh
```

***Hysteria2 启动***

启动hy2
```
systemctl start hysteria-server.service
```
开机自启
```
systemctl enable hysteria-server.service
```

## 服务端配置

- 编辑配置文件
```
vi /etc/hysteria/config.yaml
```
- 配置文件内容
```
listen: :443 #默认端口443，可以修改为其他端口

#使用CA证书
#acme:
#  domains:
#    - your.domain.net #已经解析到服务器的域名
#  email: your@email.com #你的邮箱

#使用自签证书
#tls:
#  cert: /etc/hysteria/server.crt 
#  key: /etc/hysteria/server.key 

auth:
  type: password
  password: 123456 #认证密码，使用一个强密码进行替换

resolver:
  type: udp
  tcp:
    addr: 8.8.8.8:53 
    timeout: 4s 
  udp:
    addr: 8.8.4.4:53 
    timeout: 4s
  tls:
    addr: 1.1.1.1:853 
    timeout: 10s
    sni: cloudflare-dns.com 
    insecure: false 
  https:
    addr: 1.1.1.1:443 
    timeout: 10s
    sni: cloudflare-dns.com
    insecure: false

masquerade:
  type: proxy
  proxy:
    url: https://cn.bing.com/ #伪装网址
    rewriteHost: true
```

**Tips**: 

1. 伪装网址推荐使用个人网盘的网址，个人网盘比较符合单节点大流量的特征，可以通过谷歌搜索 `intext:登录 cloudreve` 来查找别人搭建好的网盘网址

2. 服务端的配置要和客户端状态一致，也就是说，服务端配置的是无域名的配置，客户端也必须选择无域名的配置参数，必须一致，反之一样，服务端配置的是有域名的，那稍后客户端也必须选择有域名的参数

***Hysteria2 重啓***

重启Hysteria2
```
systemctl restart hysteria-server.service
```
查看状态，q退出
```
systemctl status hysteria-server.service
```

## 客户端配置 

**sing-box 配置文件**
```
{
  "log": {
    "disabled": false,
    "level": "error"
  },
  "dns": {
    "servers": [
      {
        "tag": "cloudflare",
        "address": "https://1.1.1.1/dns-query",
		"detour": "proxy"
      },
      {
        "tag": "local",
        "address": "223.5.5.5",
        "detour": "direct"
      },
      {
        "tag": "block",
        "address": "rcode://success"
      }
    ],
    "rules": [
		{
		  "geosite": [
			"cn"
		  ],
		  "server": "local",
		  "disable_cache": true
		},
		{
		  "geosite": [
			"category-ads-all"
		  ],
		  "server": "block",
		  "disable_cache": true
		}
    ],
    "strategy": "ipv4_only"
  },
  "inbounds": [
    {
      "type": "tun",
	  "tag": "tun-in",
      "inet4_address": "172.19.0.1/30",
	  "inet6_address": "fdfe:dcba:9876::1/126",
      "auto_route": true,
      "strict_route": false,
      "sniff": true
    }
  ],
  "outbounds": [
    {
      "type": "hysteria2",
      "tag": "proxy",
      "server": "111.111.111.111", #服务器地址
      "server_port": 443, #服务器端口
      "up_mbps": 20, #最大上传速率
      "down_mbps": 50, #最大下载速率
      "password": "123456", #密码和服务端一致
      "tls": {
        "enabled": true,
        "server_name": "your.domain.net", #没有域名的填伪装网址
        "insecure": false #使用自签证书需要改成true
      }
    },
    {
      "type": "direct",
      "tag": "direct"
    },
    {
      "type": "block",
      "tag": "block"
    },
    {
      "type": "dns",
      "tag": "dns-out"
    }
  ],
  "route": {
    "rules": [
      {
        "protocol": "dns",
        "outbound": "dns-out"
      },
      {
        "geosite": "cn",
        "geoip": [
          "private",
          "cn"
        ],
        "outbound": "direct"
      },
      {
        "geosite": "category-ads-all",
        "outbound": "block"
      }
    ],
    "auto_detect_interface": true
  }
}
```
*请看清楚以上配置文件中的注释，根据自己的需要，自行更改。*

如果显示：{"error": "invalid config: tls: open /etc/hysteria/server.crt: permission denied"} 或者 failed to load server conf 的错误，则说明 Hysteria 没有访问证书文件的权限，需要执行下面的命令将 Hysteria 切换到 root 用户运行
```
sed -i '/User=/d' /etc/systemd/system/hysteria-server.service
sed -i '/User=/d' /etc/systemd/system/hysteria-server@.service
systemctl daemon-reload
systemctl restart hysteria-server.service
```

**UFW 防火墙**

查看防火墙状态
```
ufw status
```
开放 80 和 443 端口
```
ufw allow http && ufw allow https
```

**性能优化**

将发送、接收的两个缓冲区都设置为 16 MB：
```
sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216
```

## Hysteria2相关命令
#启动Hysteria2
```
systemctl start hysteria-server.service
```
#停止Hysteria2
```
systemctl stop hysteria-server.service
```
#设置开机自启
```
systemctl enable hysteria-server.service
```
#重启Hysteria2
```
systemctl restart hysteria-server.service
```
#查看Hysteria2状态
```
systemctl status hysteria-server.service
```
#查看日志
```
journalctl -u hysteria-server.service
```
*Tips: 部分厂商VPS如搭建完Hysteria2无法正常启动，请执行如下命令添加私钥的读写权限，完成之后执行重启Hysteria2命令，再执行查看Hysteria2状态命令，如果启动成功即可！*
```
chmod +rw /root/private.key
```
#Hysteria 2 的伪装验证  

Windows 命令提示符进入 Chrome.exe 目标地址（cd+路径），开启 Hysteria2 代理，客户端打开 TUN 模式，cmd 命令如下：
```
chrome --origin-to-force-quic-on=你的域名:443
```
访问 https://你的域名 ,若是按照上述搭建的话，会跳转到 bing.com 的网页

## 端口跳跃
- 在基于Ubuntu和基于Debian的Linux发行版上，运行以下命令来安装ifconfig：
```
sudo apt install net-tools -y
```
- 用`ifconfig`命令查看网卡名称，默认一般是eth0。然后在终端里输入这条命令：
```
# IPv4
iptables -t nat -A PREROUTING -i eth0 -p udp --dport 20000:50000 -j REDIRECT --to-ports 443
# IPv6
ip6tables -t nat -A PREROUTING -i eth0 -p udp --dport 20000:50000 -j REDIRECT --to-ports 443
```
在这个示例中，服务器监听 443 端口，但客户端可以通过 20000-50000 范围内的任何端口连接。

## Debian系统命令放行端口
- 安装iptables（通常系统都会自带，如果没有就需要安装）
```
apt-get update
```
```
apt-get install iptables
```
- 放行端口
```
iptables -I INPUT -p tcp --dport xxxx -j ACCEPT
```
- 保存放行规则
```
iptables-save
```
- 设置完就已经放行了指定的端口，但重启后会失效，下面设置持续生效规则；
- 安装iptables-persistent
```
apt-get install iptables-persistent
```
- 保存规则持续生效
```
netfilter-persistent save
```
```
netfilter-persistent reload
```
设置完成后指定端口就会持续放行了

## 鸣谢
- 油管up主波仔分享的blog教学：https://v2rayssr.com/hysteria2.html
- Play实验室的heysteria项目：https://playlab.eu.org/archives/hysteria2
- deathline94'PJ: https://github.com/deathline94/Hysteria2-Installer



