# 部署步骤

**！！！本文档未测试完成，仅作记录！！！**

步骤 1：准备环境（中转 VPS）

更新系统
`sudo apt update && sudo apt upgrade -y`

安装完整版 Nginx
`sudo apt install -y nginx-full`

验证 stream 模块
`nginx -V 2>&1 | grep -o with-stream`

应输出 `with-stream`

步骤 2：配置 Nginx SNI 路由
创建 `/etc/nginx/stream.conf`：

```nginx
load_module modules/ngx_stream_module.so;

stream {
    # SNI 路由映射表
    map $ssl_preread_server_name $backend {
        example.example.com   VPS;  # 特殊域名到目标VPS
        default                 home;       # 其他所有域名
    }

    # 上游服务器定义
    upstream VPS {  
        server <VPS公网IP>:443;  # 替换为实际目标（终点）IP
    }
    
    upstream home {
        server 10.0.0.2:443;  # WireGuard 隧道内的家庭服务器
    }

    # SNI 路由服务器
    server {
        listen 443;
        listen [::]:443;
        
        # 关键配置：启用SNI预读
        ssl_preread on;
        
        # 透明代理模式（保持原始IP）
        proxy_bind $remote_addr transparent;
        
        # 路由到上游
        proxy_pass $backend;
        
        # 连接参数优化
        proxy_connect_timeout 5s;
        proxy_timeout 1h;
        proxy_buffer_size 16k;
    }
}
```

步骤 3：修改 Nginx 主配置
/etc/nginx/nginx.conf 添加：

在 events 块后添加
`include /etc/nginx/stream.conf`;

步骤 4：配置透明路由规则

```bash

#创建路由表
echo "100 sni_route" | sudo tee -a /etc/iproute2/rt_tables


#添加透明路由规则
sudo ip rule add fwmark 1 lookup sni_route
sudo ip route add local 0.0.0.0/0 dev lo table sni_route

#启用内核参数
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv4.conf.all.route_localnet=1
sudo sysctl -w net.ipv4.tcp_timestamps=1
sudo sysctl -w net.ipv4.tcp_tw_reuse=1

# 永久保存
echo -e "net.ipv4.ip_forward=1\nnet.ipv4.conf.all.route_localnet=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

步骤 5：修改 WireGuard 配置
`/etc/wireguard/wg0.conf`：
```
ini
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = AMUU...
MTU = 1480

# 移除443端口DNAT规则
PostUp = sysctl -w net.ipv4.ip_forward=1; \
         iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE; \
         iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 566 -j DNAT --to-destination 10.0.0.2:566; \
         iptables -A FORWARD -i wg0 -j ACCEPT; \
         iptables -A FORWARD -o wg0 -j ACCEPT

PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; \
           iptables -D FORWARD -o wg0 -j ACCEPT; \
           iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE; \
           iptables -t nat -D PREROUTING -i eth0 -p tcp --dport 566 -j DNAT --to-destination 10.0.0.2:566

[Peer]
PublicKey = Gd...
AllowedIPs = 10.0.0.2/32

```
步骤 6：重启服务

```bash
# 重载配置
sudo systemctl daemon-reload

# 重启服务（顺序重要）
sudo systemctl restart nginx
sudo wg-quick down wg0
sudo wg-quick up wg0

# 验证状态
sudo systemctl status nginx
sudo wg show
```

# Xray 配置验证
1. Xray 服务端配置

```json
{
  "inbounds": [{
    "port": 443,
    "protocol": "vless",
    "settings": {
      "clients": [{
        "id": "your-uuid",
        "flow": "xtls-rprx-direct"
      }],
      "decryption": "none"
    },
    "streamSettings": {
      "network": "tcp",
      "security": "tls",
      "tlsSettings": {
        "serverName": "example.example.com",
        "certificates": [{
          "certificateFile": "/etc/xray/cert.pem",
          "keyFile": "/etc/xray/key.pem"
        }],
        "alpn": ["h2", "http/1.1"]
      }
    },
    "sniffing": {
      "enabled": true,
      "destOverride": ["http", "tls"]
    }
  }],
  "outbounds": [{
    "protocol": "freedom"
  }]
}
```
1. 关键验证点
SNI 穿透测试：

```bash
openssl s_client -connect example.example.com:443 \
-servername example.example.com
```
✅ 应显示新加坡 VPS 的证书

✅ 证书中的 CN/SAN 包含 example.example.com

Xray 日志验证：

```bash
tail -f /var/log/xray/access.log
```
期望看到：

```text
2023/07/16 12:00:00 [Info] accepted tcp:<客户端IP>:12345 -> example.example.com:443
```
流量完整性测试：

```bash
# 在成都 VPS 抓包
sudo tcpdump -i eth0 'port 443 and host <新加坡IP>' -w sg.pcap

# 在新加坡 VPS 抓包
sudo tcpdump -i eth0 'port 443' -w xray.pcap
```
使用 Wireshark 对比：

Client Hello 中的 SNI 应一致

TLS 握手过程完整

四、部署验证脚本
1. 路由测试脚本
```bash
#!/bin/bash
# /usr/local/bin/test_sni_route.sh

# 测试特殊域名
echo "测试 example.example.com："
curl -svo /dev/null \
--resolve example.example.com:443:127.0.0.1 \
https://example.example.com 2>&1 | grep -E 'HTTP|SSL|Connected'

# 测试普通域名
echo -e "\n测试 example.com："
curl -svo /dev/null \
--resolve example.com:443:127.0.0.1 \
https://example.com 2>&1 | grep -E 'HTTP|SSL|Connected'
```



# 二次方案

步骤1：添加标记规则 (关键修正)
```bash
# 在Nginx启动前标记443端口入站流量
sudo iptables -t mangle -A PREROUTING -p tcp --dport 443 -j MARK --set-mark 1
```
步骤2：创建路由表和规则
```bash
# 创建自定义路由表
echo "100 sni_route" | sudo tee -a /etc/iproute2/rt_tables

# 添加本地路由
sudo ip route add local 0.0.0.0/0 dev lo table sni_route

# 添加标记路由规则
sudo ip rule add fwmark 1 lookup sni_route
```
步骤3：Nginx配置保持透明模式
```nginx
server {
    listen 443;
    proxy_bind $remote_addr transparent;  # 保持透明代理
    ...
}
```
为什么需要fwmark和路由表？
问题本质：

当Nginx作为透明代理时，响应包的目标地址是原始客户端IP

但系统默认路由不知道如何发送给客户端（因为不是通过默认网关进来的）

具体作用：

`fwmark 1`：标识需要特殊处理的流量

`lookup sni_route`：强制使用自定义路由表

`local 0.0.0.0/0 dev lo`：将响应导向本地协议栈

完整修正配置流程
1. 创建路由脚本
`/etc/network/if-up.d/sni-route`:

```bash
#!/bin/bash
# 创建路由表
echo "100 sni_route" >> /etc/iproute2/rt_tables

# 添加路由规则
ip rule add fwmark 1 lookup sni_route
ip route add local 0.0.0.0/0 dev lo table sni_route

# 标记入站443流量
iptables -t mangle -A PREROUTING -p tcp --dport 443 -j MARK --set-mark 1

# 启用内核参数
sysctl -w net.ipv4.ip_forward=1
sysctl -w net.ipv4.conf.all.route_localnet=1
```
2. 设置权限
```bash
sudo chmod +x /etc/network/if-up.d/sni-route
sudo /etc/network/if-up.d/sni-route  # 立即生效
```
3. 验证配置
```bash
# 检查标记规则
sudo iptables -t mangle -L -v

# 检查路由表
sudo ip rule list
sudo ip route show table sni_route

# 测试数据包流程
sudo conntrack -E -p tcp --dport 443
```
新加坡返回流量验证
2. 验证方法
```bash
# 在成都VPS查看连接跟踪
sudo conntrack -L -d example.example.com

# 示例输出：
# tcp 6 431996 ESTABLISHED src=Client_IP dst=CD_IP sport=54321 dport=443 
#     src=SG_IP dst=CD_IP sport=443 dport=54321 [ASSURED] mark=1 use=1
```
3. 关键保障点
连接跟踪(conntrack)：自动维护连接状态

标记保持：mark=1 随连接保持

路由决策：出站时使用sni_route表

源地址恢复：Nginx自动处理源地址

最终配置检查点
标记设置：

```bash
sudo iptables -t mangle -L PREROUTING -v
# 应有：tcp dpt:443 MARK set 0x1
```
路由规则：

```bash
sudo ip rule list
# 应有：1:	from all fwmark 0x1 lookup sni_route
```
路由表内容：

```bash
sudo ip route show table sni_route
# 应有：local default dev lo scope host
```
Nginx透明模式：

```nginx
proxy_bind $remote_addr transparent;
```
内核参数：

```bash
sysctl net.ipv4.conf.all.route_localnet
# 应为：net.ipv4.conf.all.route_localnet = 1
```