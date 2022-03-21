# OpenVPN-Configure
OpenVPN-Configure

## 1. 安装 OpenVPN
``` bash
# 临时关闭 selinux
setenforce 0
# 永久关闭 selinux: /etc/selinux/config
SELINUX=disabled

# epel repo
sudo wget -O /etc/yum.repos.d/epel-7.repo http://mirrors.aliyun.com/repo/epel-7.repo

# yum install openvpn
sudo yum install openvpn -y
```
## 2. 安装 EasyRSA
``` bash
# 下载 EasyRSA 3.0.8
cd ~
wget https://github.com/OpenVPN/easy-rsa/releases/download/v3.0.8/EasyRSA-3.0.8.tgz
tar xf EasyRSA-3.0.8.tgz
sudo mv EsyRSA-3.0.8 /etc/openvpn/easy-rsa-3.0.8
sudo cp /etc/openvpn/easy-rsa-3.0.8/vars.example /etc/openvpn/easy-rsa-3.0.8/vars
```
## 3. 创建服务端证书和秘钥
``` bash
cd /etc/openvpn/easy-rsa-3.0.8/
# 初始化 pki 目录
sudo ./easyrsa init-pki

# 创建根证书
sudo ./easyrsa build-ca nopass
# ./pki/ca.crt

# 创建服务端秘钥
sudo ./easyrsa gen-req server nopass
# ./pki/private/server.key

# 服务端证书签名
sudo ./easyrsa sign-req server server
# ./pki/issued/server.crt

# 创建 Diffie-Hellman
sudo ./easyrsa gen-dh
# ./pki/dh.pem

# 创建 TLS 认证密钥
sudo openvpn --genkey --secret /etc/openvpn/ta.key
# /etc/openvpn/ta.key
```
## 4. 创建客户端配置文件
``` bash
# 创建客户端秘钥
sudo ./easyrsa gen-req benben nopass
# ./pki/private/benben.key

# 客户端证书签名
sudo ./easyrsa sign-req client benben
# ./pki/issued/benben.crt
```
## 5. 服务端配置及启动服务
``` bash
# 准备服务端文件
sudo mkdir /etc/openvpn/server
sudo cp /etc/openvpn/easy-rsa-3.0.8/pki/ca.crt /etc/openvpn/easy-rsa-3.0.8/pki/dh.pem /etc/openvpn/easy-rsa-3.0.8/pki/issued/server.crt /etc/openvpn/easy-rsa-3.0.8/pki/private/server.key /etc/openvpn/easy-rsa-3.0.8/pki/ta.key /etc/openvpn/server/

# 创建配置文件
sudo cd /etc/openvpn/
sudo cp /usr/share/doc/openvpn-2.4.8/sample/sample-config-files/server.conf /etc/openvpn/server.conf
# 指定服务器地址、端口、四项配置文件绝对路径
sudo vim server.conf
# 创建客户端对应局域网地址文件
# sudo touch /etc/openvpn/allowed-client-list.txt
sudo touch /etc/openvpn/ipp.txt
# benben,10.8.0.12,udp,1194
# benben,10.8.0.12

# 修改文件目录权限
sudo chown root.openvpn /etc/openvpn/* -R
# 配置系统转发，/etc/sysctl.conf 配置文件中添加
sudo net.ipv4.ip_forward=1
# 配置系统转发生效
sudo sysctl -p 
# 配置开放端口
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
sudo iptables -I INPUT -p tcp --dport 1194 -j ACCEPT
# 保存规则并重启
sudo service iptables save
sudo systemctl restart iptables

#启动 OpenVPN 服务
sudo systemctl start openvpn@server
#确 认服务进程是否存在
netstat -lntp|grep openvpn
ps -aux|grep openvpn
```
## 6. 客户端配置及启动服务（PC/Mac/iPhone/Android）
``` bash

# 准备客户端文件
sudo mkdir /etc/openvpn/client/benben/
sudo cp /etc/openvpn/easy-rsa-3.0.8/pki/ca.crt /etc/openvpn/easy-rsa-3.0.8/pki/ta.key /etc/openvpn/easy-rsa-3.0.8/pki/issued/benben.crt /etc/openvpn/easy-rsa-3.0.8/pki/private/benben.key /etc/openvpn/client/benben/

# 创建 .ovpn 配置文件并填写上述四项配置文件的相对路径
sudo cd /etc/openvpn/client/benben/
sudo vim benben.ovpn

# 将 /etc/openvpn/client/benben/ 所有配置文件拷贝至 OpenVPN Connect: PC/Mac 的 config 目录或直接通过设置导入

# 对于 OpenVPN Connect: iPhone/Android 需要将所有配置文件整合在config.ovpn文件中
# yml 格式改为 xml 格式配置，从而将五项配置整合为新的 benben-one.ovpn 配置文件
ca ./ca.crt

<ca>
-----BEGIN CERTIFICATE-----
MIIDSzCCAjOgAwIBAgIUKG9/oRGB7hjWxm3pwGINYmhQoFcwDQYJKoZIhvcNAQEL
BQAwFjEUMBIGA1UEAwwLRWFzeS1SU0EgQ0EwHhcNMjEwODAxMDkxNzE4WhcNMzEw
NzMwMDkxNzE4WjAWMRQwEgYDVQQDDAtFYXN5LVJTQSBDQTCCASIwDQYJKoZIhvcN
AQEBBQADggEPADCCAQoCggEBAOO7CCF8wW1u0CyIAdgJUUxrWGfI+Qnkll0LR8nF
Vs1d9EplkZPmmCeQxnOTOFhZRow0A6rqcN/75vgSSXU29UA3jqhHVQk2CRa7UdIa
BQrU/2BnQMfpOf8j1QXSHMVHOGfEDlqxFtgbGNRWftaGZkMNrPuvfu9vZncU4Rni
thJKoGD3XrHBI5QjUj+nc/IFkA4Dsm7TCKwCAuGU00PjFnGCIie2klX3kumn5b7u
XAwX7/Dpg5VKdExPLta1z1Myjgzn3vmmXqC5BMHaA7ko3IRMD0tbeDKZiosEdA0l
g3NItDf4eyQAWU4fRyNfqPhkvYEw/oMoynk7hFsb+l56+LcCAwEAAaOBkDCBjTAd
BgNVHQ4EFgQUC8z1BfpgG2yMXjGgE7nxyss5Op8wUQYDVR0jBEowSIAUC8z1Bfpg
G2yMXjGgE7nxyss5Op+hGqQYMBYxFDASBgNVBAMMC0Vhc3ktUlNBIENBghQob3+h
EYHuGNbGbenAYg1iaFCgVzAMBgNVHRMEBTADAQH/MAsGA1UdDwQEAwIBBjANBgkq
hkiG9w0BAQsFAAOCAQEAHcNlRcsQXDJe9dQgoyxyaaCcW/A6NqosObp6hW8UbWoV
GsBeVayUPszR5PZHd5lEKkZd00P3qvJ3DSQh0deoaw4QSmxgc8VBiueLnPGkIchh
Nzk1Fvdpj5Ag1J4WoUFaYtcQF/S1UjKRygOtAj92/D1LhrxLv++CTJiO4/l/GILp
AyV67t+apmB2nz/cjQkwMq5jkp65y7XMzs19m0VBs/DXiLjx2VJru8jrFGc6Gjrb
B08hFmQpav8WndMqYL1oENV3uPr5n7XYlPqeddiRUy15RHR1lsKLh3NP0WhcxALm
sVaeDX+2cTbXyG/KsLMTDyt2KUKp5s6CtVNbcC/Uaw==
-----END CERTIFICATE-----
</ca>
```
## 7. 配置文件详细内容
### 7.1 server.conf
``` conf
#################################################
# Sample OpenVPN 2.0 config file for            #
# multi-client server.                          #
#                                               #
# This file is for the server side              #
# of a many-clients <-> one-server              #
# OpenVPN configuration.                        #
#                                               #
# OpenVPN also supports                         #
# single-machine <-> single-machine             #
# configurations (See the Examples page         #
# on the web site for more info).               #
#                                               #
# This config should work on Windows            #
# or Linux/BSD systems.  Remember on            #
# Windows to quote pathnames and use            #
# double backslashes, e.g.:                     #
# "C:\\Program Files\\OpenVPN\\config\\foo.key" #
#                                               #
# Comments are preceded with '#' or ';'         #
#################################################

# Which local IP address should OpenVPN
# listen on? (optional)
;local a.b.c.d

# Which TCP/UDP port should OpenVPN listen on?
# If you want to run multiple OpenVPN instances
# on the same machine, use a different port
# number for each one.  You will need to
# open up this port on your firewall.
port 1194

# TCP or UDP server?
;proto tcp
proto udp

# "dev tun" will create a routed IP tunnel,
# "dev tap" will create an ethernet tunnel.
# Use "dev tap0" if you are ethernet bridging
# and have precreated a tap0 virtual interface
# and bridged it with your ethernet interface.
# If you want to control access policies
# over the VPN, you must create firewall
# rules for the the TUN/TAP interface.
# On non-Windows systems, you can give
# an explicit unit number, such as tun0.
# On Windows, use "dev-node" for this.
# On most systems, the VPN will not function
# unless you partially or fully disable
# the firewall for the TUN/TAP interface.
;dev tap
dev tun

# Windows needs the TAP-Win32 adapter name
# from the Network Connections panel if you
# have more than one.  On XP SP2 or higher,
# you may need to selectively disable the
# Windows firewall for the TAP adapter.
# Non-Windows systems usually don't need this.
;dev-node MyTap

# SSL/TLS root certificate (ca), certificate
# (cert), and private key (key).  Each client
# and the server must have their own cert and
# key file.  The server and all clients will
# use the same ca file.
#
# See the "easy-rsa" directory for a series
# of scripts for generating RSA certificates
# and private keys.  Remember to use
# a unique Common Name for the server
# and each of the client certificates.
#
# Any X509 key management system can be used.
# OpenVPN can also use a PKCS #12 formatted key file
# (see "pkcs12" directive in man page).
ca /etc/openvpn/server/ca.crt
cert /etc/openvpn/server/server.crt
key /etc/openvpn/server/server.key  # This file should be kept secret

# Diffie hellman parameters.
# Generate your own with:
#   openssl dhparam -out dh2048.pem 2048
dh /etc/openvpn/server/dh.pem

# Network topology
# Should be subnet (addressing via IP)
# unless Windows clients v2.0.9 and lower have to
# be supported (then net30, i.e. a /30 per client)
# Defaults to net30 (not recommended)
;topology subnet

# Configure server mode and supply a VPN subnet
# for OpenVPN to draw client addresses from.
# The server will take 10.8.0.1 for itself,
# the rest will be made available to clients.
# Each client will be able to reach the server
# on 10.8.0.1. Comment this line out if you are
# ethernet bridging. See the man page for more info.
server 10.8.0.0 255.255.255.0

# Maintain a record of client <-> virtual IP address
# associations in this file.  If OpenVPN goes down or
# is restarted, reconnecting clients can be assigned
# the same virtual IP address from the pool that was
# previously assigned.
ifconfig-pool-persist ipp.txt

# Configure server mode for ethernet bridging.
# You must first use your OS's bridging capability
# to bridge the TAP interface with the ethernet
# NIC interface.  Then you must manually set the
# IP/netmask on the bridge interface, here we
# assume 10.8.0.4/255.255.255.0.  Finally we
# must set aside an IP range in this subnet
# (start=10.8.0.50 end=10.8.0.100) to allocate
# to connecting clients.  Leave this line commented
# out unless you are ethernet bridging.
;server-bridge 10.8.0.4 255.255.255.0 10.8.0.50 10.8.0.100

# Configure server mode for ethernet bridging
# using a DHCP-proxy, where clients talk
# to the OpenVPN server-side DHCP server
# to receive their IP address allocation
# and DNS server addresses.  You must first use
# your OS's bridging capability to bridge the TAP
# interface with the ethernet NIC interface.
# Note: this mode only works on clients (such as
# Windows), where the client-side TAP adapter is
# bound to a DHCP client.
;server-bridge

# Push routes to the client to allow it
# to reach other private subnets behind
# the server.  Remember that these
# private subnets will also need
# to know to route the OpenVPN client
# address pool (10.8.0.0/255.255.255.0)
# back to the OpenVPN server.
;push "route 192.168.10.0 255.255.255.0"
;push "route 192.168.20.0 255.255.255.0"
push "route 10.8.0.0 255.255.255.0"
push "route 210.38.139.148 255.255.255.0"

# To assign specific IP addresses to specific
# clients or if a connecting client has a private
# subnet behind it that should also have VPN access,
# use the subdirectory "ccd" for client-specific
# configuration files (see man page for more info).

# EXAMPLE: Suppose the client
# having the certificate common name "Thelonious"
# also has a small subnet behind his connecting
# machine, such as 192.168.40.128/255.255.255.248.
# First, uncomment out these lines:
;client-config-dir ccd
;route 192.168.40.128 255.255.255.248
# Then create a file ccd/Thelonious with this line:
#   iroute 192.168.40.128 255.255.255.248
# This will allow Thelonious' private subnet to
# access the VPN.  This example will only work
# if you are routing, not bridging, i.e. you are
# using "dev tun" and "server" directives.

# EXAMPLE: Suppose you want to give
# Thelonious a fixed VPN IP address of 10.9.0.1.
# First uncomment out these lines:
;client-config-dir ccd
;route 10.9.0.0 255.255.255.252
# Then add this line to ccd/Thelonious:
#   ifconfig-push 10.9.0.1 10.9.0.2

# Suppose that you want to enable different
# firewall access policies for different groups
# of clients.  There are two methods:
# (1) Run multiple OpenVPN daemons, one for each
#     group, and firewall the TUN/TAP interface
#     for each group/daemon appropriately.
# (2) (Advanced) Create a script to dynamically
#     modify the firewall in response to access
#     from different clients.  See man
#     page for more info on learn-address script.
;learn-address ./script

# If enabled, this directive will configure
# all clients to redirect their default
# network gateway through the VPN, causing
# all IP traffic such as web browsing and
# and DNS lookups to go through the VPN
# (The OpenVPN server machine may need to NAT
# or bridge the TUN/TAP interface to the internet
# in order for this to work properly).
push "redirect-gateway def1 bypass-dhcp"

# Certain Windows-specific network settings
# can be pushed to clients, such as DNS
# or WINS server addresses.  CAVEAT:
# http://openvpn.net/faq.html#dhcpcaveats
# The addresses below refer to the public
# DNS servers provided by opendns.com.
;push "dhcp-option DNS 208.67.222.222"
;push "dhcp-option DNS 208.67.220.220"
push "dhcp-option DNS 210.38.128.34"
push "dhcp-option DNS 210.38.137.34"

# Uncomment this directive to allow different
# clients to be able to "see" each other.
# By default, clients will only see the server.
# To force clients to only see the server, you
# will also need to appropriately firewall the
# server's TUN/TAP interface.
;client-to-client

# Uncomment this directive if multiple clients
# might connect with the same certificate/key
# files or common names.  This is recommended
# only for testing purposes.  For production use,
# each client should have its own certificate/key
# pair.
#
# IF YOU HAVE NOT GENERATED INDIVIDUAL
# CERTIFICATE/KEY PAIRS FOR EACH CLIENT,
# EACH HAVING ITS OWN UNIQUE "COMMON NAME",
# UNCOMMENT THIS LINE OUT.
;duplicate-cn

# The keepalive directive causes ping-like
# messages to be sent back and forth over
# the link so that each side knows when
# the other side has gone down.
# Ping every 10 seconds, assume that remote
# peer is down if no ping received during
# a 120 second time period.
keepalive 10 120

# For extra security beyond that provided
# by SSL/TLS, create an "HMAC firewall"
# to help block DoS attacks and UDP port flooding.
#
# Generate with:
#   openvpn --genkey --secret ta.key
#
# The server and each client must have
# a copy of this key.
# The second parameter should be '0'
# on the server and '1' on the clients.
tls-auth ta.key 0 # This file is secret

# Select a cryptographic cipher.
# This config item must be copied to
# the client config file as well.
# Note that v2.4 client/server will automatically
# negotiate AES-256-GCM in TLS mode.
# See also the ncp-cipher option in the manpage
cipher AES-256-GCM

# Enable compression on the VPN link and push the
# option to the client (v2.4+ only, for earlier
# versions see below)
;compress lz4-v2
;push "compress lz4-v2"

# For compression compatible with older clients use comp-lzo
# If you enable it here, you must also
# enable it in the client config file.
;comp-lzo

# The maximum number of concurrently connected
# clients we want to allow.
max-clients 10

# It's a good idea to reduce the OpenVPN
# daemon's privileges after initialization.
#
# You can uncomment this out on
# non-Windows systems.
;user nobody
;group nobody

# The persist options will try to avoid
# accessing certain resources on restart
# that may no longer be accessible because
# of the privilege downgrade.
persist-key
persist-tun

# Output a short status file showing
# current connections, truncated
# and rewritten every minute.
status openvpn-status.log

# By default, log messages will go to the syslog (or
# on Windows, if running as a service, they will go to
# the "\Program Files\OpenVPN\log" directory).
# Use log or log-append to override this default.
# "log" will truncate the log file on OpenVPN startup,
# while "log-append" will append to it.  Use one
# or the other (but not both).
;log         openvpn.log
;log-append  openvpn.log

# Set the appropriate level of log
# file verbosity.
#
# 0 is silent, except for fatal errors
# 4 is reasonable for general usage
# 5 and 6 can help to debug connection problems
# 9 is extremely verbose
verb 3

# Silence repeating messages.  At most 20
# sequential messages of the same message
# category will be output to the log.
;mute 20

# Notify the client that when the server restarts so it
# can automatically reconnect.
explicit-exit-notify 1
```
### 7.2 config.ovpn
``` conf
##############################################
# Sample client-side OpenVPN 2.0 config file #
# for connecting to multi-client server.     #
#                                            #
# This configuration can be used by multiple #
# clients, however each client should have   #
# its own cert and key files.                #
#                                            #
# On Windows, you might want to rename this  #
# file so it has a .ovpn extension           #
##############################################

# Specify that we are a client and that we
# will be pulling certain config file directives
# from the server.
client

# Use the same setting as you are using on
# the server.
# On most systems, the VPN will not function
# unless you partially or fully disable
# the firewall for the TUN/TAP interface.
;dev tap
dev tun

# Windows needs the TAP-Win32 adapter name
# from the Network Connections panel
# if you have more than one.  On XP SP2,
# you may need to disable the firewall
# for the TAP adapter.
;dev-node MyTap

# Are we connecting to a TCP or
# UDP server?  Use the same setting as
# on the server.
;proto tcp
proto udp

# The hostname/IP and port of the server.
# You can have multiple remote entries
# to load balance between the servers.
;remote my-server-1 1194
;remote my-server-2 1194
remote 210.38.139.148 1194

# Choose a random host from the remote
# list for load-balancing.  Otherwise
# try hosts in the order specified.
;remote-random

# Keep trying indefinitely to resolve the
# host name of the OpenVPN server.  Very useful
# on machines which are not permanently connected
# to the internet such as laptops.
resolv-retry infinite

# Most clients don't need to bind to
# a specific local port number.
nobind

# Downgrade privileges after initialization (non-Windows only)
;user nobody
;group nobody

# Try to preserve some state across restarts.
persist-key
persist-tun

# If you are connecting through an
# HTTP proxy to reach the actual OpenVPN
# server, put the proxy server/IP and
# port number here.  See the man page
# if your proxy server requires
# authentication.
;http-proxy-retry # retry on connection failures
;http-proxy [proxy server] [proxy port #]

# Wireless networks often produce a lot
# of duplicate packets.  Set this flag
# to silence duplicate packet warnings.
;mute-replay-warnings

# SSL/TLS parms.
# See the server config file for more
# description.  It's best to use
# a separate .crt/.key file pair
# for each client.  A single ca
# file can be used for all clients.
ca ca.crt
cert benben.crt
key benben.key

# Verify server certificate by checking that the
# certicate has the correct key usage set.
# This is an important precaution to protect against
# a potential attack discussed here:
#  http://openvpn.net/howto.html#mitm
#
# To use this feature, you will need to generate
# your server certificates with the keyUsage set to
#   digitalSignature, keyEncipherment
# and the extendedKeyUsage to
#   serverAuth
# EasyRSA can do this for you.
remote-cert-tls server

# If a tls-auth key is used on the server
# then every client must also have the key.
tls-auth ta.key 1

# Select a cryptographic cipher.
# If the cipher option is used on the server
# then you must also specify it here.
# Note that v2.4 client/server will automatically
# negotiate AES-256-GCM in TLS mode.
# See also the ncp-cipher option in the manpage
cipher AES-256-GCM

# Enable compression on the VPN link.
# Don't enable this unless it is also
# enabled in the server config file.
#comp-lzo

# Set log file verbosity.
verb 3

# Silence repeating messages
;mute 20
```
### 7.3 config-one.ovpn
``` conf
##############################################
# Sample client-side OpenVPN 2.0 config file #
# for connecting to multi-client server.     #
#                                            #
# This configuration can be used by multiple #
# clients, however each client should have   #
# its own cert and key files.                #
#                                            #
# On Windows, you might want to rename this  #
# file so it has a .ovpn extension           #
##############################################

# Specify that we are a client and that we
# will be pulling certain config file directives
# from the server.
client

# Use the same setting as you are using on
# the server.
# On most systems, the VPN will not function
# unless you partially or fully disable
# the firewall for the TUN/TAP interface.
;dev tap
dev tun

# Windows needs the TAP-Win32 adapter name
# from the Network Connections panel
# if you have more than one.  On XP SP2,
# you may need to disable the firewall
# for the TAP adapter.
;dev-node MyTap

# Are we connecting to a TCP or
# UDP server?  Use the same setting as
# on the server.
;proto tcp
proto udp

# The hostname/IP and port of the server.
# You can have multiple remote entries
# to load balance between the servers.
;remote my-server-1 1194
;remote my-server-2 1194
remote 210.38.139.148 1194

# Choose a random host from the remote
# list for load-balancing.  Otherwise
# try hosts in the order specified.
;remote-random

# Keep trying indefinitely to resolve the
# host name of the OpenVPN server.  Very useful
# on machines which are not permanently connected
# to the internet such as laptops.
resolv-retry infinite

# Most clients don't need to bind to
# a specific local port number.
nobind

# Downgrade privileges after initialization (non-Windows only)
;user nobody
;group nobody

# Try to preserve some state across restarts.
persist-key
persist-tun

# If you are connecting through an
# HTTP proxy to reach the actual OpenVPN
# server, put the proxy server/IP and
# port number here.  See the man page
# if your proxy server requires
# authentication.
;http-proxy-retry # retry on connection failures
;http-proxy [proxy server] [proxy port #]

# Wireless networks often produce a lot
# of duplicate packets.  Set this flag
# to silence duplicate packet warnings.
;mute-replay-warnings

# SSL/TLS parms.
# See the server config file for more
# description.  It's best to use
# a separate .crt/.key file pair
# for each client.  A single ca
# file can be used for all clients.
# ca ca.crt
# cert benben.crt
# key benben.key

<ca>
-----BEGIN CERTIFICATE-----
MIIDSzCCAjOgAwIBAgIUKG9/oRGB7hjWxm3pwGINYmhQoFcwDQYJKoZIhvcNAQEL
BQAwFjEUMBIGA1UEAwwLRWFzeS1SU0EgQ0EwHhcNMjEwODAxMDkxNzE4WhcNMzEw
NzMwMDkxNzE4WjAWMRQwEgYDVQQDDAtFYXN5LVJTQSBDQTCCASIwDQYJKoZIhvcN
AQEBBQADggEPADCCAQoCggEBAOO7CCF8wW1u0CyIAdgJUUxrWGfI+Qnkll0LR8nF
Vs1d9EplkZPmmCeQxnOTOFhZRow0A6rqcN/75vgSSXU29UA3jqhHVQk2CRa7UdIa
BQrU/2BnQMfpOf8j1QXSHMVHOGfEDlqxFtgbGNRWftaGZkMNrPuvfu9vZncU4Rni
thJKoGD3XrHBI5QjUj+nc/IFkA4Dsm7TCKwCAuGU00PjFnGCIie2klX3kumn5b7u
XAwX7/Dpg5VKdExPLta1z1Myjgzn3vmmXqC5BMHaA7ko3IRMD0tbeDKZiosEdA0l
g3NItDf4eyQAWU4fRyNfqPhkvYEw/oMoynk7hFsb+l56+LcCAwEAAaOBkDCBjTAd
BgNVHQ4EFgQUC8z1BfpgG2yMXjGgE7nxyss5Op8wUQYDVR0jBEowSIAUC8z1Bfpg
G2yMXjGgE7nxyss5Op+hGqQYMBYxFDASBgNVBAMMC0Vhc3ktUlNBIENBghQob3+h
EYHuGNbGbenAYg1iaFCgVzAMBgNVHRMEBTADAQH/MAsGA1UdDwQEAwIBBjANBgkq
hkiG9w0BAQsFAAOCAQEAHcNlRcsQXDJe9dQgoyxyaaCcW/A6NqosObp6hW8UbWoV
GsBeVayUPszR5PZHd5lEKkZd00P3qvJ3DSQh0deoaw4QSmxgc8VBiueLnPGkIchh
Nzk1Fvdpj5Ag1J4WoUFaYtcQF/S1UjKRygOtAj92/D1LhrxLv++CTJiO4/l/GILp
AyV67t+apmB2nz/cjQkwMq5jkp65y7XMzs19m0VBs/DXiLjx2VJru8jrFGc6Gjrb
B08hFmQpav8WndMqYL1oENV3uPr5n7XYlPqeddiRUy15RHR1lsKLh3NP0WhcxALm
sVaeDX+2cTbXyG/KsLMTDyt2KUKp5s6CtVNbcC/Uaw==
-----END CERTIFICATE-----
</ca>

<cert>
-----BEGIN CERTIFICATE-----
MIIDVDCCAjygAwIBAgIQSIBQYdQdxrupgpeQiH168zANBgkqhkiG9w0BAQsFADAW
MRQwEgYDVQQDDAtFYXN5LVJTQSBDQTAeFw0yMTEwMDgwODQyMDRaFw0yNDAxMTEw
ODQyMDRaMBExDzANBgNVBAMMBmJlbmJlbjCCASIwDQYJKoZIhvcNAQEBBQADggEP
ADCCAQoCggEBALj/EjOWCDy6k/h3FjZY/gg19OyO8b2WV8B+XV0I6wf0tZXlxG6p
xxSyyyneaNoM72WXJw5+m0y93rkHTrc2IdmW2mnq2eD+YIv+Chf4L1dAdnClvm+D
JSaZviYxNkqkEq6WVfhprlBFtTZ8AOpYinQDQN3sFm1S5Z1nU7ktQQ6kAG7ya4uy
VbJCnb2iKq0hNZUzlPy8JdnSLkeftBBC4oAjkfjX5eOWbN8vysL8rMLZ8dvl7Scw
Dy23swWcV+NQBGw2i9uuZyCqMipqmtr0k35+2mopAyFykMsqAijMhwUXsutdR/y/
58HXhp5GrD4nG60eUvyhHEC46/bMgJKMdq8CAwEAAaOBojCBnzAJBgNVHRMEAjAA
MB0GA1UdDgQWBBTJduAiAoTIbajGRTjNkuE33nP64TBRBgNVHSMESjBIgBQLzPUF
+mAbbIxeMaATufHKyzk6n6EapBgwFjEUMBIGA1UEAwwLRWFzeS1SU0EgQ0GCFChv
f6ERge4Y1sZt6cBiDWJoUKBXMBMGA1UdJQQMMAoGCCsGAQUFBwMCMAsGA1UdDwQE
AwIHgDANBgkqhkiG9w0BAQsFAAOCAQEAHC/BpMYFD5ujzQhc7OLljA+aKTAaKWe3
tB9K6DVnYT8OqUF3sOGhxLeuk7pDsf9QgwPcjDFp9Vhs3Dd9GfUMg/pO3a5iItEc
zWPPF/+DwI6kSfZgLc+fmQa7IYKfK59YAKN3cLrRniJq6XL6nxSsbbFm2IFqP0xf
Weagzb7fVQ2FG+szdHIqKqy78xdwBcX4dOBcnXDs3t/9+zZYpuwBAWUq2HtGA5Es
8WrnaG/S72cb/FkrCH1WH95u/1HafBEL2xqGz5arJ1riuGuh/A+jZ3SNQxDm1SlS
ua8zlM4jDgYooP2Te0lRmoWJ3LA0HfcsSJmI+B/8Bt650xTNMegEvg==
-----END CERTIFICATE-----
</cert>

<key>
-----BEGIN PRIVATE KEY-----
MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQC4/xIzlgg8upP4
dxY2WP4INfTsjvG9llfAfl1dCOsH9LWV5cRuqccUsssp3mjaDO9llycOfptMvd65
B063NiHZltpp6tng/mCL/goX+C9XQHZwpb5vgyUmmb4mMTZKpBKullX4aa5QRbU2
fADqWIp0A0Dd7BZtUuWdZ1O5LUEOpABu8muLslWyQp29oiqtITWVM5T8vCXZ0i5H
n7QQQuKAI5H41+XjlmzfL8rC/KzC2fHb5e0nMA8tt7MFnFfjUARsNovbrmcgqjIq
apra9JN+ftpqKQMhcpDLKgIozIcFF7LrXUf8v+fB14aeRqw+JxutHlL8oRxAuOv2
zICSjHavAgMBAAECggEAMihuXfA60YRg5EgdjKS6U62Vd6IWJyohJr7cP4JQfzq5
FShUBxEfOhxz+ykjUqOZMPk3jLWFE9yTC6XQkNoreVxuYbNcWaV+tdYuFGulIkoH
EunNZVywcPPUW3SSXNB5hD6clprIuVj9FgWvFdrlxyiuqLz/I6sLOI8wYw/DCN22
fmgF9TGU/bAoPtU6cXyYJTu/0IMIG51yh76CmNJQiaAnSM4VgkWHDmD2ymzfRxWC
eus5TFSUbkqVPlTBlljfEEskPRlVq10OkdRx/srt15d+all82kGJN5A6vcB0UtUC
BJEvE6M/CeuqZMlrkoTXo2i2KIo3H/lkOxzzfoFxqQKBgQDdDO65E5Ta3EXbiLBi
cwSxyb6fCbZsPmjAiukWWbs1Cda6yZ5+PzD6B0b19MD0+yrwYY/aw+9adwctF4Bn
hrREqP/ZeaCzDERDrMgfdrfcji/Jew4okUWUx2mDN+1VKpSppsjPm8iiYPTDEilE
00CbVFAwXsZf+ild4A7bFkNO8wKBgQDWPtbXsa2eznqmIKNU2DM5ybNi1aJSSMsf
B0FSpazI/rYOmEaN1I8Jh1vpsLfL1dlPs8vKCqaYsZMnXPFsuPON9GznMi9RjcSD
6JfHsC2bV3HNNVr6FvLb/HhMJEYLySziQnHeld06RKbw5PdZgE2rHjTZikUQlVTX
1OId0k/AVQKBgBEIgBSu15eNxaxG+iB78G6qtw+WNgJdRMEhcxiPzYcmvO8jvhzI
TcPWb7dgJsY53HMtcWJQGs+DwH/PAcv4a0enJh/h6WoildgJJlqWUVCjfDcwTkT9
/LicLRs5YgZgA5iXC35D6M/qXLHzYk61YJMXih5QD0UyB6H+M+bZ7lHVAoGAeFkI
OlWWn9SA1P0Ugr6H1/hTijtTWUGGyEE9En36V1WtUvl6+ITkbIfau6UHObtAvSLU
YQQmnTNy4/Ozsk0aky0wV5a7OeaW8zoeuI9grxgp1woXttBZT/W8ZZkit9AkJF0K
tewdP3P9CuizgVUvS+ZF7cVcEnqwFCWDdxkCr5kCgYEAgKAUX+VYeX7gFgwyM8B2
0bq67qDjfyqBA31ayjufy5r9lAtuRfyUmE6MlESIfs95vwMGqF+iQRJ9CIVoaQkK
dhtepm+hiSYL7zmqOAexgeM0CLX/OcTk+OEES1Sw0pHISWq/qvXjlDieikmqo320
cCzzyeK4KLg4TBt1JxYE3h4=
-----END PRIVATE KEY-----
</key>

# Verify server certificate by checking that the
# certicate has the correct key usage set.
# This is an important precaution to protect against
# a potential attack discussed here:
#  http://openvpn.net/howto.html#mitm
#
# To use this feature, you will need to generate
# your server certificates with the keyUsage set to
#   digitalSignature, keyEncipherment
# and the extendedKeyUsage to
#   serverAuth
# EasyRSA can do this for you.
remote-cert-tls server

# If a tls-auth key is used on the server
# then every client must also have the key.
# tls-auth ta.key 1

key-direction 1
<tls-auth>
-----BEGIN OpenVPN Static key V1-----
53b53c1f6c85a5d857471cd13d6f9aba
ff7eee25395164a4be626ec3d08a329f
f9691c87f273caa1093cf6eaa2bdfb07
3ec8d15951f184d75edcb8729ed52981
656918c21cf3950c69e0e0cccce23ae6
f585f51ac4bcca8a01b021b1dd9fece6
bda585978bd8daf8baabd6032dcf4e39
978878ca9638f4f53b0cf9bb03a4b945
0dabf6c4c39b168bb15f3b3e209f050a
e8f539d39594f090546c6079fbf5fdb3
0e37be6f704d51edc427400e8a2b3e43
b22a9a0521e8177f49b0ab683b150dd0
d56596f24d902493776ec67aa3f90cbd
93b4ce938d522107dc894fbbf3338982
641a84f612373fb292502245c4fd94d6
391733b070a684fce1d34ef740b91d7b
-----END OpenVPN Static key V1-----
</tls-auth>

# Select a cryptographic cipher.
# If the cipher option is used on the server
# then you must also specify it here.
# Note that v2.4 client/server will automatically
# negotiate AES-256-GCM in TLS mode.
# See also the ncp-cipher option in the manpage
cipher AES-256-GCM

# Enable compression on the VPN link.
# Don't enable this unless it is also
# enabled in the server config file.
#comp-lzo

# Set log file verbosity.
verb 3

# Silence repeating messages
;mute 20
```