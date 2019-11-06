<!-- TOC -->

- [1. 验证是否支持OpenVPN](#1-验证是否支持openvpn)
- [2. 服务器端](#2-服务器端)
    - [2.1. 创建CA、签发证书](#21-创建ca签发证书)
        - [2.1.1. 安装easy-rsa并设置配置文件](#211-安装easy-rsa并设置配置文件)
        - [2.1.2. 生成CA和服务器端证书](#212-生成ca和服务器端证书)
        - [2.1.3. 生成测试客户端证书并签名](#213-生成测试客户端证书并签名)
        - [2.1.4. 整理证书](#214-整理证书)
            - [2.1.4.1. 服务器](#2141-服务器)
            - [2.1.4.2. 测试客户端](#2142-测试客户端)
    - [2.2. 安装openvpn并配置服务器](#22-安装openvpn并配置服务器)
        - [2.2.1. 安装以及配置防火墙](#221-安装以及配置防火墙)
        - [2.2.2. 开启内核转发](#222-开启内核转发)
        - [2.2.3. 启动openvpn](#223-启动openvpn)
        - [2.2.4. 固定客户端地址](#224-固定客户端地址)
- [3. 客户端连接](#3-客户端连接)
    - [3.1. windows](#31-windows)
    - [3.2. Linux](#32-linux)
        - [3.2.1. 设置自启动](#321-设置自启动)
- [4. 后期维护](#4-后期维护)

<!-- /TOC -->
# 1. 验证是否支持OpenVPN
```bash
cat /dev/net/tun
# cat:/dev/net/tun:Filedescriptor in bad state（开启了tun/tap，支持）
# cat:/dev/net/tun:Permissiondenied（未开启tun/tap，不支持）
```
# 2. 服务器端
## 2.1. 创建CA、签发证书
### 2.1.1. 安装easy-rsa并设置配置文件
```bash
# 安装easy-rsa
yum -y install easy-rsa
# 拷贝easyrsa为两份，分别用作客户端和服务器，可能需要更改版本号
cp -av /usr/share/easy-rsa/3.0.6 /etc/openvpn/easyrsa_server
cp -av /usr/share/easy-rsa/3.0.6 /etc/openvpn/easyrsa_testclient
# 编辑拷贝出来的两份vars，该文件存储了组织信息等
vim /etc/openvpn/easyrsa_server/vars
vim /etc/openvpn/easyrsa_testclient/vars
# vars文件内容如下
if [ -z "$EASYRSA_CALLER" ]; then
    echo "You appear to be sourcing an Easy-RSA 'vars' file." >&2
    echo "This is no longer necessary and is disallowed. See the section called" >&2
    echo "'How to use this file' near the top comments for more details." >&2
    return 1
fi
set_var EASYRSA    "$PWD"
set_var EASYRSA_PKI        "$EASYRSA/pki"
set_var EASYRSA_DN    "cn_only"
set_var EASYRSA_REQ_COUNTRY     "CN" #国家
set_var EASYRSA_REQ_PROVINCE    "BEIJING" #省份
set_var EASYRSA_REQ_CITY        "BEIJING" #城市
set_var EASYRSA_REQ_ORG         "OpenVPN CERTIFICATE AUTHORITY" #组织
set_var EASYRSA_REQ_EMAIL       "110@openvpn.com" #管理员邮箱
set_var EASYRSA_REQ_OU          "OpenVPN EASY CA" #部门
set_var EASYRSA_KEY_SIZE        2048              #key长度
set_var EASYRSA_ALGO            rsa               #key 类型
set_var EASYRSA_CA_EXPIRE       7000
set_var EASYRSA_CERT_EXPIRE     3650
set_var EASYRSA_NS_SUPPORT      "no"
set_var EASYRSA_NS_COMMENT      "OpenVPN CERTIFICATE AUTHORITY"
set_var EASYRSA_EXT_DIR "$EASYRSA/x509-types"
set_var EASYRSA_SSL_CONF        "$EASYRSA/openssl-1.0.cnf"
set_var EASYRSA_DIGEST          "sha256"
```
### 2.1.2. 生成CA和服务器端证书
```bash
# 切换目录
cd /etc/openvpn/easyrsa_server
# 初始化，在当前目录创建PKI目录，用于存储一些中间变量及最终生成的证书
./easyrsa init-pki
# 创建根证书，首先会提示设置CA密码（用于CA对之后生成的server和client证书签名），之后设置各种信息，可以直接键入回车默认 
./easyrsa build-ca
# 创建server端证书和private key，nopass表示不加密private key，之后设置各种信息，可以直接键入回车默认 
./easyrsa gen-req vpnserver nopass
# 给server端证书做签名，首先是对一些信息的确认，可以输入yes，然后输入CA密码 
./easyrsa sign server vpnserver
# 创建Diffie-Hellman，时间会有点长，耐心等待 
./easyrsa gen-dh 
```
### 2.1.3. 生成测试客户端证书并签名
```bash
# 切换目录
cd /etc/openvpn/easyrsa_testclient
# 初始化，在当前目录创建PKI目录，用于存储一些中间变量及最终生成的证书 
./easyrsa init-pki 
# 创建client端证书和private key，nopass表示不加密private key，后设置各种信息，可以直接键入回车默认，testclient为自定义客户端名称
./easyrsa gen-req testclient nopass
# 回到服务器的easyrsa3目录，导入client端证书
cd /etc/openvpn/easyrsa_server
./easyrsa import-req /etc/openvpn/easyrsa_testclient/pki/reqs/testclient.req testclient
# 给client端证书做签名，首先是对一些信息的确认，可以输入yes，然后输入CA密码
./easyrsa sign client testclient
```
### 2.1.4. 整理证书
#### 2.1.4.1. 服务器
```bash
mkdir -p /etc/openvpn/server_keys
cp /etc/openvpn/easyrsa_server/pki/ca.crt /etc/openvpn/server_keys/
cp /etc/openvpn/easyrsa_server/pki/private/vpnserver.key /etc/openvpn/server_keys/
cp /etc/openvpn/easyrsa_server/pki/issued/vpnserver.crt /etc/openvpn/server_keys/
cp /etc/openvpn/easyrsa_server/pki/dh.pem /etc/openvpn/server_keys/
```
#### 2.1.4.2. 测试客户端
```bash
mkdir /etc/openvpn/testclient_keys
cp /etc/openvpn/easyrsa_server/pki/ca.crt /etc/openvpn/testclient_keys/
cp /etc/openvpn/easyrsa_server/pki/issued/testclient.crt  /etc/openvpn/testclient_keys/
cp /etc/openvpn/easyrsa_testclient/pki/private/testclient.key /etc/openvpn/testclient_keys/
```
## 2.2. 安装openvpn并配置服务器
```bash
# 安装
yum install epel-release -y
yum install  openssh-server lzo openssl openssl-devel openvpn NetworkManager-openvpn openvpn-auth-ldap -y
# 复制配置文件
cp /usr/share/doc/openvpn-*/sample/sample-config-files/server.conf /etc/openvpn/
# 切换目录
cd /etc/openvpn
# 如果开启tls-auth执行这条命令，并将key复制到整理的证书目录下
openvpn --genkey --secret ta.key
cp /etc/openvpn/ta.key /etc/openvpn/server_keys/
cp /etc/openvpn/ta.key /etc/openvpn/testclient_keys/
# 编辑服务器端配置文件
vim /etc/openvpn/server.conf
# 服务器端配置文件内容如下
local 192.168.1.1                             # 监听网卡地址
port 1290                                     # 端口
proto tcp                                     # 协议，可以是tcp或者udp，但是需要服务器与客户端保持一致
dev tun
ca /etc/openvpn/server_keys/ca.crt            # 根据情况修改
cert /etc/openvpn/server_keys/vpnserver.crt   # 根据服务端证书修改     
key /etc/openvpn/server_keys/vpnserver.key    # 根据服务端密钥修改 
dh /etc/openvpn/server_keys/dh.pem
tls-auth /etc/openvpn/ta.key 0                # 如果开启tls-auth，解除注释
server 10.8.0.0 255.255.255.0                 # 分配的IP段
ifconfig-pool-persist ipp.txt                 # 存储IP地址对应关系的文件
keepalive 10 120                              # 存活时间段
comp-lzo                                      # 启用允许数据压缩，客户端配置文件也需要有这项
persist-key                                   # 通过keepalive检测超时后，重新启动VPN，不重新读取keys，保留第一次使用的keys
persist-tun                                   # 通过keepalive检测超时后，重新启动VPN，一直保持tun或者tap设备是linkup的。否则会先linkdown然后再linkup
status openvpn-status.log                     # 日志文件，相对路径
log-append  openvpn.log                       # 日志文件，相对路径
verb 3                                        # 设置日志记录冗长级别
mute 20                                       # 重复日志记录限额
client-config-dir /etc/openvpn/ip             # 固定客户端IP地址，设置存储对应关系的路径
push "route 192.168.1.0 255.255.255.0"        # 推送路由信息
push "dhcp-option DNS 114.114.114.114"        # 推送dns信息
max-clients 50                                # 最大客户端数量
user openvpn                                  # 运行软件的的用户
group openvpn                                 # 运行软件的的用户组
```
### 2.2.1. 安装以及配置防火墙
```
# 安装iptables防火墙
yum install iptables-services –y 
systemctl enable iptables.service
# 禁用firewalled防火墙
systemctl mask firewalld.service
systemctl stop firewalld.service
systemctl disable firewalld.service
# 添加NAT转换，使得客户端连接上VPN之后可以通过地址转换访问VPN服务器同网段内网主机
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
```
### 2.2.2. 开启内核转发
编辑 `/etc/sysctl.conf`文件,将`net.ipv4.ip_forward = 0`修改为`net.ipv4.ip_forward = 1`，然后执行`sysctl -p`
### 2.2.3. 启动openvpn
`systemctl start openvpn@server`
### 2.2.4. 固定客户端地址
客户端的的IP地扯变动和openvpn的版本变动会导致分配的IP地址变动，可能会导致客户端之间无法正常通讯。`mkdir -p /etc/openvpn/ip`，`vim /etc/openvpn/ip/testclient`，名称对应客户端名称，内容形如`ifconfig-push 10.8.0.17 10.8.0.18`，ifconfig-push后面跟着的是两个连续的成组IP地址
```bash
[  1,  2] [  5,  6] [  9, 10] [ 13, 14] [ 17, 18]
[ 21, 22] [ 25, 26] [ 29, 30] [ 33, 34] [ 37, 38]
[ 41, 42] [ 45, 46] [ 49, 50] [ 53, 54] [ 57, 58]
[ 61, 62] [ 65, 66] [ 69, 70] [ 73, 74] [ 77, 78]
[ 81, 82] [ 85, 86] [ 89, 90] [ 93, 94] [ 97, 98]
[101,102] [105,106] [109,110] [113,114] [117,118]
[121,122] [125,126] [129,130] [133,134] [137,138]
[141,142] [145,146] [149,150] [153,154] [157,158]
[161,162] [165,166] [169,170] [173,174] [177,178]
[181,182] [185,186] [189,190] [193,194] [197,198]
[201,202] [205,206] [209,210] [213,214] [217,218]
[221,222] [225,226] [229,230] [233,234] [237,238]
[241,242] [245,246] [249,250] [253,254]
```
# 3. 客户端连接
## 3.1. windows
下载[客户端](https://openvpn.net/community-downloads/)并安装，将证书放到C:\Program Files (x86)\OpenVPN\config\目录下，再在该目录下新建一个open.ovpn，内容如下
```bash
client
dev tun                                            //openvpn运行的模式，和Server端保持一致
proto tcp                                          //协议，和Server端保持一致
nobind
remote 1.1.1.1 1290                                //这里填写公网地址和对应端口
ns-cert-type server
tls-auth ta.key 1
ca ca.crt
cert hui_client.crt
key hui_client.key
keepalive 10 120
persist-key
persist-tun
comp-lzo
verb 3
status hui-status.log
log-append hui.log
route-nopull        //局部代理
route 10.8.0.0 255.255.255.0 vpn_gateway
```
保存，启动openvpn客户端即可
## 3.2. Linux
安装openvpn后，复制证书，创建配置文件open.ovpn，直接openvpn open.ovpn即可
### 3.2.1. 设置自启动
`vim /etc/crontab` 添加一行`*/1 * * * * root /etc/openvpn/crond.sh`，代表每分钟执行一次该脚本，脚本内容如下
```bash
#!/bin/bash
# if openvpn not running , then start it.
LOG_PATH=/var/log/crond.log
TIME_NOW=`date "+%Y-%m-%d %H:%M:%S"`
PIDS=`ps -ef |grep "openvpn open.ovpn" |grep -v grep | grep -v crond`
if [ "$PIDS" != "" ]; then
echo "[****INFO*******]$TIME_NOW openvpn is runing!" >> $LOG_PATH
else
echo "[****WARNING****]$TIME_NOW openvpn is stopped, restart it!" >> $LOG_PATH
cd /etc/openvpn/client/ && openvpn open.ovpn &
fi
```
# 4. 后期维护
* openvpn目录下的ipp.txt文件夹存储了客户端名称和相应的地址
* openvpn目录下的easyrsa3_server/pki/index.txt存储了签发的证书，其中包括客户端名称
