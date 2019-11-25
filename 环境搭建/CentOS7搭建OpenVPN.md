<!-- TOC -->

- [1. 验证是否支持OpenVPN](#1-验证是否支持openvpn)
- [2. 服务器端](#2-服务器端)
    - [2.1. 安装easy-rsa并设置配置文件](#21-安装easy-rsa并设置配置文件)
    - [2.2. 生成CA和服务器端证书](#22-生成ca和服务器端证书)
    - [2.3. 生成测试客户端证书](#23-生成测试客户端证书)
    - [2.4. 整理证书](#24-整理证书)
        - [2.4.1. 服务器](#241-服务器)
        - [2.4.2. 测试客户端](#242-测试客户端)
    - [2.5. 安装openvpn并配置服务器](#25-安装openvpn并配置服务器)
    - [2.6. 安装以及配置防火墙](#26-安装以及配置防火墙)
    - [2.7. 开启内核转发](#27-开启内核转发)
    - [2.8. 启动openvpn并设置自启动](#28-启动openvpn并设置自启动)
    - [2.9. 固定客户端地址（可选）](#29-固定客户端地址可选)
- [3. 客户端连接](#3-客户端连接)
    - [3.1. windows](#31-windows)
    - [3.2. Linux](#32-linux)
        - [3.2.1. 设置自启动（进程终结自动连接VPN）](#321-设置自启动进程终结自动连接vpn)
- [4. 后期维护](#4-后期维护)
- [5. 附件：配置文件详解](#5-附件配置文件详解)
    - [5.1. 服务器端](#51-服务器端)
    - [5.2. 客户端](#52-客户端)

<!-- /TOC -->
# 1. 验证是否支持OpenVPN
```bash
cat /dev/net/tun
# cat:/dev/net/tun:Filedescriptor in bad state（开启了tun/tap，支持）
# cat:/dev/net/tun:Permissiondenied（未开启tun/tap，不支持）
```
# 2. 服务器端
## 2.1. 安装easy-rsa并设置配置文件
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
## 2.2. 生成CA和服务器端证书
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
## 2.3. 生成测试客户端证书
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
## 2.4. 整理证书
### 2.4.1. 服务器
```bash
mkdir -p /etc/openvpn/server_keys
cp /etc/openvpn/easyrsa_server/pki/ca.crt /etc/openvpn/server_keys/
cp /etc/openvpn/easyrsa_server/pki/private/vpnserver.key /etc/openvpn/server_keys/
cp /etc/openvpn/easyrsa_server/pki/issued/vpnserver.crt /etc/openvpn/server_keys/
cp /etc/openvpn/easyrsa_server/pki/dh.pem /etc/openvpn/server_keys/
```
### 2.4.2. 测试客户端
```bash
mkdir /etc/openvpn/testclient_keys
cp /etc/openvpn/easyrsa_server/pki/ca.crt /etc/openvpn/testclient_keys/
cp /etc/openvpn/easyrsa_server/pki/issued/testclient.crt  /etc/openvpn/testclient_keys/
cp /etc/openvpn/easyrsa_testclient/pki/private/testclient.key /etc/openvpn/testclient_keys/
```
## 2.5. 安装openvpn并配置服务器
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
```
服务器端配置文件内容如下：
```
local 192.168.1.1                             # 监听网卡地址
port 1290                                     # 端口
proto tcp                                     # 协议，可以是tcp或者udp，但是需要服务器与客户端保持一致
dev tun                                       # VPN模式，tun为路由模式，tap为隧道模式
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
duplicate-cn                                  # 允许单证书多点登录
```
## 2.6. 安装以及配置防火墙
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
service iptables save
```
## 2.7. 开启内核转发
编辑 `/etc/sysctl.conf`文件,将`net.ipv4.ip_forward = 0`修改为`net.ipv4.ip_forward = 1`，然后执行`sysctl -p`
## 2.8. 启动openvpn并设置自启动
```bash
systemctl start openvpn@server
systemctl enable openvpn@server
```
## 2.9. 固定客户端地址（可选）
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
下载[客户端](https://openvpn.net/community-downloads/)并安装，将证书放到C:\Program Files (x86)\OpenVPN\config\目录下，再在该目录下新建一个open.ovpn，内容如下：
```
client
dev tun                                            # openvpn运行的模式，和Server端保持一致
proto tcp                                          # 协议，和Server端保持一致
nobind                                             # 不监听
remote 1.1.1.1 1290                                # 这里填写公网地址和对应端口
ns-cert-type server                                # 验证证书，安全措施
tls-auth ta.key 1                                  # tls认证，安全措施
ca ca.crt                                          # CA证书
cert testclient.crt                                # 证书
key testclient.key                                 # key
keepalive 10 120
persist-key
persist-tun
comp-lzo                                           # 启用数据压缩
verb 3                                             # 日志冗余级别
status testclient-status.log                       # 状态日志
log-append testclient.log                          # 运行日志
route-nopull                                       # 局部代理
route 10.8.0.0 255.255.255.0 vpn_gateway           # 路由
```
保存，启动openvpn客户端即可
## 3.2. Linux
安装openvpn后，复制证书，创建配置文件open.ovpn，直接openvpn open.ovpn即可
### 3.2.1. 设置自启动（进程终结自动连接VPN）
vim /etc/crontab 添加一行*/1 * * * * root /etc/openvpn/crond.sh，代表每分钟执行一次该脚本，脚本内容如下：
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
* openvpn目录下的easyrsa_server/pki/index.txt存储了签发的证书，其中包括客户端名称
# 5. 附件：配置文件详解
## 5.1. 服务器端
```
#################################################
# 针对多客户端的OpenVPN 2.0 的服务器端配置文件示例
#
# 本文件用于多客户端<->单服务器端的OpenVPN服务器端配置
#
# OpenVPN也支持单机<->单机的配置(更多信息请查看网站上的示例页面)
#
# 该配置支持Windows或者Linux/BSD系统。此外，在Windows上，记得将路径加上双引号，
# 并且使用两个反斜杠，例如："C:\\Program Files\\OpenVPN\\config\\foo.key"
#
# '#' or ';'开头的均为注释内容
#################################################

#OpenVPN应该监听本机的哪些IP地址？
#该命令是可选的，如果不设置，则默认监听本机的所有IP地址。
;local a.b.c.d

# OpenVPN应该监听哪个TCP/UDP端口？
# 如果你想在同一台计算机上运行多个OpenVPN实例，你可以使用不同的端口号来区分它们。
# 此外，你需要在防火墙上开放这些端口。
port 1194

#OpenVPN使用TCP还是UDP协议?
;proto tcp
proto udp

# 指定OpenVPN创建的通信隧道类型。
# "dev tun"将会创建一个路由IP隧道，
# "dev tap"将会创建一个以太网隧道。
#
# 如果你是以太网桥接模式，并且提前创建了一个名为"tap0"的与以太网接口进行桥接的虚拟接口，则你可以使用"dev tap0"
#
# 如果你想控制VPN的访问策略，你必须为TUN/TAP接口创建防火墙规则。
#
# 在非Windows系统中，你可以给出明确的单位编号(unit number)，例如"tun0"。
# 在Windows中，你也可以使用"dev-node"。
# 在多数系统中，除非你部分禁用或者完全禁用了TUN/TAP接口的防火墙，否则VPN将不起作用。
;dev tap
dev tun

# 如果你想配置多个隧道，你需要用到网络连接面板中TAP-Win32适配器的名称(例如"MyTap")。
# 在XP SP2或更高版本的系统中，你可能需要有选择地禁用掉针对TAP适配器的防火墙
# 通常情况下，非Windows系统则不需要该指令。
;dev-node MyTap

# 设置SSL/TLS根证书(ca)、证书(cert)和私钥(key)。
# 每个客户端和服务器端都需要它们各自的证书和私钥文件。
# 服务器端和所有的客户端都将使用相同的CA证书文件。
#
# 通过easy-rsa目录下的一系列脚本可以生成所需的证书和私钥。
# 记住，服务器端和每个客户端的证书必须使用唯一的Common Name。
#
# 你也可以使用遵循X509标准的任何密钥管理系统来生成证书和私钥。
# OpenVPN 也支持使用一个PKCS #12格式的密钥文件(详情查看站点手册页面的"pkcs12"指令)
ca ca.crt
cert server.crt
key server.key  # 该文件应该保密

# 指定迪菲·赫尔曼参数。
# 你可以使用如下名称命令生成你的参数：
#   openssl dhparam -out dh1024.pem 1024
# 如果你使用的是2048位密钥，使用2048替换其中的1024。
dh dh1024.pem

# 设置服务器端模式，并提供一个VPN子网，以便于从中为客户端分配IP地址。
# 在此处的示例中，服务器端自身将占用10.8.0.1，其他的将提供客户端使用。
# 如果你使用的是以太网桥接模式，请注释掉该行。更多信息请查看官方手册页面。
server 10.8.0.0 255.255.255.0

# 指定用于记录客户端和虚拟IP地址的关联关系的文件。
# 当重启OpenVPN时，再次连接的客户端将分配到与上一次分配相同的虚拟IP地址
ifconfig-pool-persist ipp.txt

# 该指令仅针对以太网桥接模式。
# 首先，你必须使用操作系统的桥接能力将以太网网卡接口和TAP接口进行桥接。
# 然后，你需要手动设置桥接接口的IP地址、子网掩码；
# 在这里，我们假设为10.8.0.4和255.255.255.0。
# 最后，我们必须指定子网的一个IP范围(例如从10.8.0.50开始，到10.8.0.100结束)，以便于分配给连接的客户端。
# 如果你不是以太网桥接模式，直接注释掉这行指令即可。
;server-bridge 10.8.0.4 255.255.255.0 10.8.0.50 10.8.0.100

# 该指令仅针对使用DHCP代理的以太网桥接模式，
# 此时客户端将请求服务器端的DHCP服务器，从而获得分配给它的IP地址和DNS服务器地址。
#
# 在此之前，你也需要先将以太网网卡接口和TAP接口进行桥接。
# 注意：该指令仅用于OpenVPN客户端，并且该客户端的TAP适配器需要绑定到一个DHCP客户端上。
;server-bridge

# 推送路由信息到客户端，以允许客户端能够连接到服务器背后的其他私有子网。
# (简而言之，就是允许客户端访问VPN服务器自身所在的其他局域网)
# 记住，这些私有子网也要将OpenVPN客户端的地址池(10.8.0.0/255.255.255.0)反馈回OpenVPN服务器。
;push "route 192.168.10.0 255.255.255.0"
;push "route 192.168.20.0 255.255.255.0"

# 为指定的客户端分配指定的IP地址，或者客户端背后也有一个私有子网想要访问VPN，
# 那么你可以针对该客户端的配置文件使用ccd子目录。
# (简而言之，就是允许客户端所在的局域网成员也能够访问VPN)

# 举个例子：假设有个Common Name为"Thelonious"的客户端背后也有一个小型子网想要连接到VPN，该子网为192.168.40.128/255.255.255.248。
# 首先，你需要去掉下面两行指令的注释：
;client-config-dir ccd
;route 192.168.40.128 255.255.255.248
# 然后创建一个文件ccd/Thelonious，该文件的内容为：
#     iroute 192.168.40.128 255.255.255.248
#这样客户端所在的局域网就可以访问VPN了。
# 注意，这个指令只能在你是基于路由、而不是基于桥接的模式下才能生效。
# 比如，你使用了"dev tun"和"server"指令。

# 再举个例子：假设你想给Thelonious分配一个固定的IP地址10.9.0.1。
# 首先，你需要去掉下面两行指令的注释：
;client-config-dir ccd
;route 10.9.0.0 255.255.255.252
# 然后在文件ccd/Thelonious中添加如下指令：
#   ifconfig-push 10.9.0.1 10.9.0.2

# 如果你想要为不同群组的客户端启用不同的防火墙访问策略，你可以使用如下两种方法：
# (1)运行多个OpenVPN守护进程，每个进程对应一个群组，并为每个进程(群组)启用适当的防火墙规则。
# (2) (进阶)创建一个脚本来动态地修改响应于来自不同客户的防火墙规则。
# 关于learn-address脚本的更多信息请参考官方手册页面。
;learn-address ./script

# 如果启用该指令，所有客户端的默认网关都将重定向到VPN，这将导致诸如web浏览器、DNS查询等所有客户端流量都经过VPN。
# (为确保能正常工作，OpenVPN服务器所在计算机可能需要在TUN/TAP接口与以太网之间使用NAT或桥接技术进行连接)
;push "redirect-gateway def1 bypass-dhcp"

# 某些具体的Windows网络设置可以被推送到客户端，例如DNS或WINS服务器地址。
# 下列地址来自opendns.com提供的Public DNS 服务器。
;push "dhcp-option DNS 208.67.222.222"
;push "dhcp-option DNS 208.67.220.220"

# 去掉该指令的注释将允许不同的客户端之间相互"可见"(允许客户端之间互相访问)。
# 默认情况下，客户端只能"看见"服务器。为了确保客户端只能看见服务器，你还可以在服务器端的TUN/TAP接口上设置适当的防火墙规则。
;client-to-client

# 如果多个客户端可能使用相同的证书/私钥文件或Common Name进行连接，那么你可以取消该指令的注释。
# 建议该指令仅用于测试目的。对于生产使用环境而言，每个客户端都应该拥有自己的证书和私钥。
# 如果你没有为每个客户端分别生成Common Name唯一的证书/私钥，你可以取消该行的注释(但不推荐这样做)。
;duplicate-cn

# keepalive指令将导致类似于ping命令的消息被来回发送，以便于服务器端和客户端知道对方何时被关闭。
# 每10秒钟ping一次，如果120秒内都没有收到对方的回复，则表示远程连接已经关闭。
keepalive 10 120

# 出于SSL/TLS之外更多的安全考虑，创建一个"HMAC 防火墙"可以帮助抵御DoS攻击和UDP端口淹没攻击。
# 你可以使用以下命令来生成：
#   openvpn --genkey --secret ta.key
#
# 服务器和每个客户端都需要拥有该密钥的一个拷贝。
# 第二个参数在服务器端应该为'0'，在客户端应该为'1'。
;tls-auth ta.key 0 # 该文件应该保密

# 选择一个密码加密算法。
# 该配置项也必须复制到每个客户端配置文件中。
;cipher BF-CBC        # Blowfish (默认)
;cipher AES-128-CBC   # AES
;cipher DES-EDE3-CBC  # Triple-DES

# 在VPN连接上启用压缩。
# 如果你在此处启用了该指令，那么也应该在每个客户端配置文件中启用它。
comp-lzo

# 允许并发连接的客户端的最大数量
;max-clients 100

# 在完成初始化工作之后，降低OpenVPN守护进程的权限是个不错的主意。
# 该指令仅限于非Windows系统中使用。
;user nobody
;group nobody

# 持久化选项可以尽量避免访问那些在重启之后由于用户权限降低而无法访问的某些资源。
persist-key
persist-tun

# 输出一个简短的状态文件，用于显示当前的连接状态，该文件每分钟都会清空并重写一次。
status openvpn-status.log

# 默认情况下，日志消息将写入syslog(在Windows系统中，如果以服务方式运行，日志消息将写入OpenVPN安装目录的log文件夹中)。
# 你可以使用log或者log-append来改变这种默认情况。
# "log"方式在每次启动时都会清空之前的日志文件。
# "log-append"这是在之前的日志内容后进行追加。
# 你可以使用两种方式之一(但不要同时使用)。
;log         openvpn.log
;log-append  openvpn.log

# 为日志文件设置适当的冗余级别(0~9)。冗余级别越高，输出的信息越详细。
#
# 0 表示静默运行，只记录致命错误。
# 4 表示合理的常规用法。
# 5 和 6 可以帮助调试连接错误。
# 9 表示极度冗余，输出非常详细的日志信息。
verb 3

# 重复信息的沉默度。
# 相同类别的信息只有前20条会输出到日志文件中。
;mute 20
```
## 5.2. 客户端
```
##############################################
# 针对多个客户端的OpenVPN 2.0 的客户端配置文件示例
#
# 该配置文件可以被多个客户端使用，当然每个客户端都应该有自己的证书和密钥文件
#
# 在Windows上此配置文件的后缀应该是".ovpn"，在Linux/BSD系统中则是".conf"
##############################################

# 指定这是一个客户端，我们将从服务器获取某些配置文件指令
client

# 在大多数系统中，除非你部分禁用或者完全禁用了TUN/TAP接口的防火墙，否则VPN将不起作用。
;dev tap
dev tun

# 在Windows系统中，如果你想配置多个隧道，则需要该指令。
# 你需要用到网络连接面板中TAP-Win32适配器的名称(例如"MyTap")。
# 在XP SP2或更高版本的系统中，你可能需要禁用掉针对TAP适配器的防火墙。
;dev-node MyTap

# 指定连接的服务器是采用TCP还是UDP协议。
# 这里需要使用与服务器端相同的设置。
;proto tcp
proto udp

# 指定服务器的主机名(或IP)以及端口号。
# 如果有多个VPN服务器，为了实现负载均衡，你可以设置多个remote指令。
remote my-server-1 1194
;remote my-server-2 1194

# 如果指定了多个remote指令，启用该指令将随机连接其中的一台服务器，
# 否则，客户端将按照指定的先后顺序依次尝试连接服务器。
;remote-random

# 启用该指令，与服务器连接中断后将自动重新连接，这在网络不稳定的情况下(例如：笔记本电脑无线网络)非常有用。
resolv-retry infinite

# 大多数客户端不需要绑定本机特定的端口号
nobind

# 在初始化完毕后，降低OpenVPN的权限(该指令仅限于非Windows系统中使用)
;user nobody
;group nobody

# 持久化选项可以尽量避免访问在重启时由于用户权限降低而无法访问的某些资源。
persist-key
persist-tun

# 如果你是通过HTTP代理方式来连接到实际的VPN服务器，请在此处指定代理服务器的主机名(或IP)和端口号。
# 如果你的代理服务器需要身份认证，请参考官方手册页面。
;http-proxy-retry # 连接失败时自动重试
;http-proxy [proxy server] [proxy port #]

# 无线网络通常会产生大量的重复数据包。设置此标识将忽略掉重复数据包的警告信息。
;mute-replay-warnings

# SSL/TLS 参数配置。
# 更多描述信息请参考服务器端配置文件。
# 最好为每个客户端单独分配.crt/.key文件对。
# 单个CA证书可以供所有客户端使用。
ca ca.crt
cert client.crt
key client.key

# 指定通过检查证书的nsCertType字段是否为"server"来验证服务器端证书。
# 这是预防潜在攻击的一种重要措施。
#
# 为了使用该功能，你需要在生成服务器端证书时，将其中的nsCertType字段设为"server"
# easy-rsa文件夹中的build-key-server脚本文件可以达到该目的。
ns-cert-type server

# 如果服务器端使用了tls-auth密钥，那么每个客户端也都应该有该密钥。
;tls-auth ta.key 1

# 指定密码的加密算法。
# 如果服务器端启用了cipher指令选项，那么你必须也在这里指定它。
;cipher x

# 在VPN连接中启用压缩。
# 该指令的启用/禁用应该与服务器端保持一致。
comp-lzo

# 设置日志文件冗余级别(0~9)。
# 0 表示静默运行，只记录致命错误。
# 4 表示合理的常规用法。
# 5 和 6 可以帮助调试连接错误。
# 9 表示极度冗余，输出非常详细的日志信息。
verb 3

# 忽略过多的重复信息。
# 相同类别的信息只有前20条会输出到日志文件中。
;mute 20
```