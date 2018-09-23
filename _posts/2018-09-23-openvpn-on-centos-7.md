---
toc: true
date: 2018-09-23 21:35:00 +08:00
---
## OpenVPN on CentOS 7.x <sub>{{ page.date | date: '%Y-%m-%d' }}</sub>

### 概述


### 环境

1. Server: CentOS 7.x
1. Client: Windows 10 / Ubuntu 18.04 / macOS 10.13 

### OpenVPN Server 安装

#### 1. 安装 openvpn

```bash
$ sudo yum install -y openvpn  # 我的版本是 openvpn-2.4.6
```

#### 2. 安装 easy-rsa 3.x

```bash
$ sudo yum install easy-rsa  # 我的版本是 easy-rsa-3.0.3
```

#### 3. 使用 easy-rsa 生成自签名 CA 证书

```bash
# 为了方便，将证书生成环境放到了 /etc/openvpn 目录下
$ sudo -s  # 为了操作方便，直接切换到 root 下来执行如下的命令
$ cp -R /usr/share/easy-rsa /etc/openvpn/
$ cd /etc/openvpn/easy-rsa/3/
$ ./easyrsa init-pki
# 如下命令，在要求输入 Common Name(CN) 的时候直接回车 使用默认 Common Name(CN) 就可以了
# 如果需要配置自己的 Common Name(CN), 可以输入公司英文简称样的名称，比如: Google VPN CA
$ ./easyrsa build-ca # 生成 私钥 和 CA 证书
# 或者 如果需要生成无密码的私钥，可以使用如下的命令
$ ./easyrsa build-ca nopass
```

#### 4. 签发 Server 证书

```bash
$ sudo -s  # 如果已经切换到 root 用户，就不需要执行这个命令了
$ cd /etc/openvpn/easy-rsa/3/
$ mkdir -p server
$ cp -R /usr/share/easy-rsa server/
$ cd server/easy-rsa/3/
$ ./easyrsa init-pki
# 如下命令中的 server 可以根据需要修改成期望的名称，比如: vpn.example.com
# 在要求输入 Common Name(CN) 的时候，可以直接回车使用默认的  Common Name(CN) 就可以了
# 如果需要配置自己的 Common Name(CN)，可以使用服务器的域名，比如: vpn.example.com
$ ./easyrsa gen-req server nopass
$ cd /etc/openvpn/easy-rsa/3/
$ ./easyrsa import-req server/easy-rsa/3/pki/reqs/server.req server
# 下面命令中的 第二个 server 同 gen-req 命令中的 server
$ ./easyrsa sign-req server server
$ ls pki/issued/server.crt  # 这个文件就是 server 证书了
$ ls server/easy-rsa/3/pki/private/server.key  # 这个文件是 server 证书的私钥了
```

#### 5. 签发 Client 证书

```bash
$ sudo -s  # 如果已经切换到 root 用户，就不需要执行这个命令了
$ cd /etc/openvpn/easy-rsa/3/
$ mkdir -p client
$ cp -R /usr/share/easy-rsa client/
$ cd client/easy-rsa/3/
$ ./easyrsa init-pki
# 如下命令中的 client 可以根据需要修改成期望的名称，比如: jackma
# 在要求输入 Common Name(CN) 的时候，可以直接回车使用默认的  Common Name(CN) 就可以了
# 如果需要配置自己的 Common Name(CN), 可以使用用户名，比如: jackma
$ ./easyrsa gen-req client nopass
$ cd /etc/openvpn/easy-rsa/3/
$ ./easyrsa import-req client/easy-rsa/3/pki/reqs/client.req client
# 下面命令中的 第二个 client 同 gen-req 命令中的 client
$ ./easyrsa sign-req client client
$ ls pki/issued/client.crt  # 这个文件就是 client 证书了
$ ls server/easy-rsa/3/pki/private/client.key  # 这个文件是 client 证书的私钥了
```

#### 6. 生成 DH 参数

```bash
$ sudo -s  # 如果已经切换到 root 用户，就不需要执行这个命令了
$ cd /etc/openvpn/easy-rsa/3/
$ ./easyrsa gen-dh
```

#### 7. 生成 ta.key

```bash
$ sudo -s  # 如果已经切换到 root 用户，就不需要执行这个命令了
$ cd /etc/openvpn/easy-rsa/3/
$ openvpn --genkey --secret ta.key
```

#### 8. 整理文件 

```bash
$ sudo -s  # 如果已经切换到 root 用户，就不需要执行这个命令了
# 整理 OpenVPN Server 端需要的文件
$ cp /etc/openvpn/easy-rsa/3/pki/ca.crt /etc/openvpn/
$ cp /etc/openvpn/easy-rsa/3/pki/dh.pem /etc/openvpn/
$ cp /etc/openvpn/easy-rsa/3/ta.key /etc/openvpn/
$ cp /etc/openvpn/easy-rsa/3/pki/issued/server.crt /etc/openvpn/
$ cp /etc/openvpn/easy-rsa/3/server/easy-rsa/3/pki/private/server.key /etc/openvpn/
# 服务器端需要的证书，私钥文件如下
$ ls ca.crt server.crt server.key ta.key dh.pem
# 整理客户端需要的文件
$ mkdir /etc/openvpn/client
$ cp /etc/openvpn/easy-rsa/3/pki/ca.crt /etc/openvpn/client/
$ cp /etc/openvpn/easy-rsa/3/client/easy-rsa/3/pki/private/client.key /etc/openvpn/client/
$ cp /etc/openvpn/easy-rsa/3/pki/issued/client.crt /etc/openvpn/client/
$ cp /etc/openvpn/easy-rsa/3/ta.key /etc/openvpn/client/
# 客户端需要的文件如下
$ ls ca.crt client.key client.crt ta.key 
$ cd /etc/openvpn
$ tar -czf client.tgz client  # 压缩以备客户端使用
```

#### 9. 修改 server.conf

```bash
$ sudo -s  # 如果已经切换到 root 用户，就不需要执行这个命令了
$ rpm -ql openvpn|grep '/server.conf'
# 拷贝上面输出的文件到 /etc/openvpn, 作为 OpenVPN Server 的配置文件
$ cp /usr/share/doc/openvpn-2.4.6/sample/sample-config-files/server.conf /etc/openvpn/
$ vim /etc/openvpn/server.conf
# 查找如下的配置项，修改或者取消 注释
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 8.8.4.4"
duplicate-cn
user nobody 
group nobody
log-append  openvpn.log
```

最终生成的文件有效的内容如下

```bash
# grep -v '^[#;]' server.conf |grep -v '^$'
port 1194
proto udp
dev tun
ca ca.crt
cert server.crt
key server.key  # This file should be kept secret
dh dh.pem
server 10.8.0.0 255.255.255.0
ifconfig-pool-persist ipp.txt
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 8.8.4.4"
duplicate-cn
keepalive 10 120
tls-auth ta.key 0 # This file is secret
cipher AES-256-CBC
user nobody
group nobody
persist-key
persist-tun
status openvpn-status.log
log-append  openvpn.log
verb 3
explicit-exit-notify 1
```

#### 10. 修改防火墙 / 路由 / 转发 规则

1. 如果使用了硬件的防火墙或者云上的安全组等，首先需要修改配置以允许访问 UDP:1194 端口
1. 在服务器上还需要使用 iptables 或者 firewalld 来设置防火墙
1. 编辑 /etc/sysctl.conf 增加 net.ipv4.ip_forward=1 配置
  1. sudo bash -c 'echo net.ipv4.ip_forward=1 >> /etc/sysctl.conf'
  1. sudo sysctl -p  # 使上面的修改生效

```bash
# 使用 iptables 配置防火墙
$ sudo -s  # 如果已经切换到 root 用户，就不需要执行这个命令了
$ yum install -y iptables
$ iptables -A INPUT -p udp --dport 1194 -i eth0 -j ACCEPT
$ iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
# 如果上面的命令没有作用的话，可以试试下面的命令
# $ iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -j SNAT --to <server ip>
$ iptables-save > /etc/sysconfig/iptablesvpn
$ systemctl start iptables
$ systemctl enable iptables
#
# 使用 firewalld 配置防火墙
```

#### 11. 启动 openvpn

```bash
$ sudo systemctl enable openvpn@server
$ sudo systemctl start openvpn@server
```

### 客户端配置

1. 将在上面打包的 client.tgz 文件传输到客户端所在的操作系统

#### Ubuntu 18.04


```bash
$ sudo -s
$ apt install openvpn
$ tar -xzf client.tgz
$ cp client/{ca.crt,client.crt,client.key,ta.key} /etc/openvpn/
$ cat <<EOF > /etc/openvpn/client.conf
client
dev tun
#dev-node OpenVPN_TAP
proto udp
# 根据实际情况修改为 OpenVPN Server 的域名或者 IP
remote vpn.example.com 1194
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
cert client.crt
key client.key
tls-auth ta.key 1
remote-cert-tls server
log-append openvpn.log
verb 3
auth-nocache
EOF
$ systemctl start openvpn@client
# 如果没有意外，现在就可以通过VPN访问了
```


#### macOS

Mac 上可以使用 brew 安装 openvpn 或者使用 tunnelblick 来使用openvpn

```bash
# 如果使用 brew 安装 openvpn
$ brew install openvpn
$ tar -xzf client.tgz
$ cp -r client /usr/local/etc/openvpn/
$ cat <<EOF > /usr/local/etc/openvpn/openvpn.conf
client
dev tun
#dev-node OpenVPN_TAP
proto udp
# 根据需要修改如下的域名为实际的 OpenVPN Server 的域名或者IP
remote vpn.example.com 1194
resolv-retry infinite
nobind
persist-key
persist-tun
ca /usr/local/etc/openvpn/client/ca.crt
cert /usr/local/etc/openvpn/client/client.crt
key /usr/local/etc/openvpn/client/client.key
tls-auth /usr/local/etc/openvpn/client/ta.key 1
remote-cert-tls server
log-append /usr/local/var/log/openvpn.log
verb 3
auth-nocache
EOF
$ sudo brew services start openvpn
# 如果没有就可以通过 OpenVPN 访问了
```

```bash
# 使用 tunnelblick 来安装
$ tar -xzf client.tgz
$ cd client
$ cat <<EOF > openvpn.ovpn
client
dev tun
#dev-node OpenVPN_TAP
proto udp
# 根据需要修改如下的域名为实际的 OpenVPN Server 的域名或者IP
remote vpn.example.com 1194
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
cert client.crt
key client.key
tls-auth ta.key 1
remote-cert-tls server
#log-append openvpn.log
verb 3
auth-nocache
EOF
# 然后直接将 openvpn.ovpn 拖动到 tunnelblick 中。点击连接就可以了
```
#### Windows 10

### 参考文档

1. https://community.openvpn.net/openvpn/wiki/EasyRSA3-OpenVPN-Howto
1. https://www.howtoforge.com/tutorial/how-to-install-openvpn-on-centos-7/
1. https://blog.rj-bai.com/post/136.html
