---
title: OPENVPN部署文档
date: '2019-12-20T17:58:41.000Z'
categories:
  - linux
  - network
tags:
  - openvpn
---

# openvpn配置

## 1.部署环境

* 服务器： Linux网关服务器
* IP地址： 192.168.31.1（外） 192.168.123.1（内）
* 软件版本： openvpn-2.2.2
* 配置时间同步： /usr/sbin/ntpdate pool.ntp.org

## 2.安装软件

```bash
yum -y install openvpn lzo
```

## 3.服务端配置

### 3.1 生成CA证书、服务端、客户端证书

```bash
# 进入源码包目录
cd openvpn-2.2.2/easy-rsa/2.0/
cp vars vars.bak
vim vars
#修改对应默认配置
> export KEY_COUNTRY="CN"
> export KEY_PROVINCE="BJ"
> export KEY_CITY="BeiJing"
> export KEY_ORG="gw"
> export KEY_EMAIL="zqingfa@gmail.com"
> export KEY_CN=CN
> export KEY_NAME=gw
> export KEY_OU=gw
> export PKCS11_MODULE_PATH=
> export PKCS11_PIN=
source  vars
./clean-all
#生成服务端证书和密钥文件
./build-ca
ll keys/
#生成服务端证书和密钥文件
./build-key-server server
#生成客户端证书和密钥文件
#如果退出了bash，之后需要加新密钥，需要重新source  vars，但是不要./clean-all
./build-key test
ll keys/
#如果需要加密客户端密钥：
./build-key-pass ett
#生成交换密钥协议文件
 ./build-dh
ll keys/dh1024.pem
#为防止恶意攻击如DOS，UDP flooding，生成一个“HMAC firewall”
openvpn --genkey --secret keys/ta.key
```

### 3.2 配置文件

```bash
mkdir /etc/openvpn
cd openvpn-2.2.2/easy-rsa/2.0/
cp -ap keys /etc/openvpn
cd ../../sample-config-files/
cp client.conf server.conf /etc/openvpn/
# 配置文件
cd /etc/openvpn/
cp server.conf server.conf.ori
# 配置参数：
egrep -v ";|#|^$" server.conf
> local 192.168.31.1
> port 52115
> proto tcp
> dev tun
> ca /etc/openvpn/keys/ca.crt
> cert /etc/openvpn/keys/server.crt
> key /etc/openvpn/keys/server.key
> dh /etc/openvpn/keys/dh1024.pem
> server 10.8.0.0 255.255.255.0
> push "route 192.168.123.0 255.255.255.0"
> push "route 192.168.2.0 255.255.255.0"
> push "route 192.168.3.0 255.255.255.0"
> push "route 192.168.4.0 255.255.255.0"
> ifconfig-pool-persist ipp.txt
> keepalive 10 120
> comp-lzo
> persist-key
> persist-tun
> status openvpn-status.log
> verb 3
> client-to-client
> duplicate-cn
> log /var/log/openvpn.log
```

### 3.3 修改iptables

```bash
vim /etc/sysconfig/iptables
# 添加：
-A INPUT -p tcp --dport 52115 -j ACCEPT
-A POSTROUTING -s  10.8.0.0/24  -o enp11s0f0 -j MASQUERADE
# 重启iptables
service iptables restart
```

或者直接运行如下命令

```bash
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o enp11s0f0 -j MASQUERADE
```

### 3.4 打开Linux系统内核转发

```bash
vim /etc/sysctl.conf
net.ipv4.ip_forward = 1
# 重载配置生效
sysctl -p
```

### 3.4 启动openvpn

启动VPN，注意/etc/openvpn/下的\*.conf文件，不能有server配置以外的配置文件。

```bash
/etc/init.d/openvpn start
#编译安装启动方法：
/usr/local/sbin/openvpn --config  /etc/openvpn/server.conf &
#开机自启动
echo '/usr/local/sbin/openvpn --config  /etc/openvpn/server.conf &' >> /etc/rc.local
#openvpn自带脚本的启动方法：
/usr/local/sbin/openvpn --daemon --writepid /var/run/openvpn/server.pid --config server.conf  --cd /etc/openvpn
```

## 4. 客户端安装配置

```bash
# 自行百度安装对应操作系统的OPENVPN客户端
# 配置
cd openvpn-2.2.2/easy-rsa/2.0/keys
# 拷贝生成的客户端证书
sz -y ca.crt test.crt test.key
# 将上述文件拷贝到openvpn客户端安装路径下的config目录,新建一个文件夹存放，例如新建一个test文件夹
# 在test文件夹新建openvpn的配置文件test.ovpn，配置如下：
> client
> dev tun
> proto tcp
> remote 140.206.0.0 52115
> resolv-retry infinite
> nobind
> persist-key
> persist-tun
> ca ca.crt
> cert test.crt
> key test.key
> ns-cert-type server
> comp-lzo
> verb 3
```

配置完成，打开openvpn客户端拨号连接。



