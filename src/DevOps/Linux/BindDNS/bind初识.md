---
order: 2
date: 2025-06-29
---

# bind 初识

## 主流DNS软件

- BindDNS
- PowerDNS
- Unbound
- CoreDNS

## 安装Bind

### Bind相关程序包

- bind：服务器
- bind-libs：相关库
- bind-utils：客户端
- bind-chroot：安全包，将dns相关文件放至 /var/named/chroot/

### Bind安装包相关文件

- Bind主程序：/usr/sbin/named
- 服务脚本和Unit名称：/etc/rc.d/init.d/named、/usr/lib/systemd/system/named.service
- 主配置文件：/etc/named.conf、/etc/named.rfc1912.zones、/etc/rndc.key
- 数据库文件目录：/var/named/
- 管理工具：/usr/sbin/rndc（remote name domain controlle），默认与bind安装在同一主机，且只能通过127.0.0.1连接named进程，提供辅助性的管理功能；运行端口为953/tcp
- 解析库文件：/var/named/ZONE_NAME.ZONE

> 注意：
>
> 1. 一台物理服务器可同时为多个区域提供解析
> 2. 必须要有根区域文件；named.ca
> 3. 应该有两个（如果包括ipv6的，应该更多）实现localhost和本地回环地址的解析库

### 主配置文件

- 全局配置：options {};
- 日志子系统配置：logging {}；
- 区域定义：zone "." IN {};   本机能够为哪些zone进行解析，就要定义哪些zone

```c
//
// named.conf
//
// Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
// server as a caching only nameserver (as a localhost DNS resolver only).
//
// See /usr/share/doc/bind*/sample/ for example named configuration files.
//

options {
        // listen-on port 53 { 127.0.0.1; };
        // ** 必改配置：127.0.0.1只能本机访问，bind不支持0.0.0.0这种方式开启远程访问，
        //    而是用localhost这个关键字表示当前主机的所有IP都能被访问
        listen-on port 53 { localhost; };
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        secroots-file   "/var/named/data/named.secroots";
        recursing-file  "/var/named/data/named.recursing";
        // ** 必改配置：localhost表示只允许本机的所有IP查询，需要改成目标网段
        // allow-query     { localhost; };
        // allow-query     { localhost;6.0.0.0/24; };
        //    使用any关键字表示允许所有网段查询
        allow-query     { any; };

        /*
         - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
         - If you are building a RECURSIVE (caching) DNS server, you need to enable
           recursion.
         - If your recursive DNS server has a public IP address, you MUST enable access
           control to limit queries to your legitimate users. Failing to do so will
           cause your server to become part of large scale DNS amplification
           attacks. Implementing BCP38 within your network would greatly
           reduce such attack surface
        */
        recursion yes;

        dnssec-validation yes;

        managed-keys-directory "/var/named/dynamic";
        geoip-directory "/usr/share/GeoIP";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";

        /* https://fedoraproject.org/wiki/Changes/CryptoPolicy */
        include "/etc/crypto-policies/back-ends/bind.config";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
        type hint;
        file "named.ca";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

> 注意：
>
> 1. 任何服务程序如果期望其能够通过网络被其它主机访问，至少应该监听在一个能与外部主机通信的IP地址上
> 2. 缓存名称服务器的配置：监听外部地址即可
> 3. dnssec：建议关闭dnssec，设为no

### 安装与启动

yum系安装

```shell
yum install bind -y
```

版本

```shell
[root@dev ~]# rpm -qi bind
Name        : bind
Epoch       : 32
Version     : 9.16.23
Release     : 29.el9_6
Architecture: aarch64
Install Date: Sun Jun 29 20:04:17 2025
Group       : Unspecified
Size        : 1775034
License     : MPLv2.0
Signature   : RSA/SHA256, Wed Jun 25 10:48:02 2025, Key ID 702d426d350d275d
Source RPM  : bind-9.16.23-29.el9_6.src.rpm
Build Date  : Wed Jun 25 10:42:47 2025
Build Host  : pb-0449412f-0dcf-4b26-aff4-b6b217eb67cf-b-aarch64
Packager    : Rocky Linux Build System (Peridot) <releng@rockylinux.org>
Vendor      : Rocky Enterprise Software Foundation
URL         : https://www.isc.org/downloads/bind/
Summary     : The Berkeley Internet Name Domain (BIND) DNS (Domain Name System) server
Description :
BIND (Berkeley Internet Name Domain) is an implementation of the DNS
(Domain Name System) protocols. BIND includes a DNS server (named),
which resolves host names to IP addresses; a resolver library
(routines for applications to use when interfacing with DNS); and
tools for verifying that the DNS server is operating properly.
```

目录

```shell
[root@dev ~]# rpm -ql bind
/etc/logrotate.d/named
/etc/named
/etc/named.conf
/etc/named.rfc1912.zones
/etc/named.root.key
/etc/rndc.conf
/etc/rndc.key
/etc/rwtab.d/named
/etc/sysconfig/named
/run/named
/usr/bin/mdig
/usr/bin/named-rrchecker
/usr/lib/.build-id
/usr/lib/.build-id/39
/usr/lib/systemd/system/named-setup-rndc.service
/usr/lib/systemd/system/named.service
/usr/lib/tmpfiles.d/named.conf
/usr/lib64/bind
/usr/lib64/named
/usr/lib64/named/filter-aaaa.so
/usr/libexec/generate-rndc-key.sh
/usr/sbin/named
/usr/sbin/named-checkconf
/usr/sbin/named-journalprint
/usr/sbin/rndc
/usr/sbin/rndc-confgen
/usr/share/doc/bind
/usr/share/man/man1/mdig.1.gz
/var/log/named.log
/var/named
/var/named/data
/var/named/dynamic
/var/named/named.ca
/var/named/named.empty
/var/named/named.localhost
/var/named/named.loopback
/var/named/slaves
[root@dev ~]#
```

启动

```shell
systemctl enable --now named
```

运行端口

```shell
# 主程序端口： 53/(tcp|udp)
# rndc管理端口：953/tcp
root@dev ~]# ss -nutl | grep -E '(53|953)'
udp   UNCONN 0      0                              127.0.0.1:53         0.0.0.0:*
udp   UNCONN 0      0                              127.0.0.1:53         0.0.0.0:*
udp   UNCONN 0      0                                  [::1]:53            [::]:*
udp   UNCONN 0      0                                  [::1]:53            [::]:*
tcp   LISTEN 0      10                             127.0.0.1:53         0.0.0.0:*
tcp   LISTEN 0      10                             127.0.0.1:53         0.0.0.0:*
tcp   LISTEN 0      4096                           127.0.0.1:953        0.0.0.0:*
tcp   LISTEN 0      10                                 [::1]:53            [::]:*
tcp   LISTEN 0      10                                 [::1]:53            [::]:*
tcp   LISTEN 0      4096                               [::1]:953           [::]:*
```

## DNS 服务基本功能实现

改网络配置文件

```shell
[root@dev ~]# cat /etc/NetworkManager/system-connections/ens160.nmconnection
[connection]
id=ens160
uuid=c551de10-c168-3347-9f2d-e211049a3098
type=ethernet
autoconnect-priority=-999
interface-name=ens160
timestamp=1739924028

[ethernet]

[ipv4]
address1=192.168.1.149/24,192.168.1.1
dns=127.0.0.1;   # 设置为本机
method=manual

[ipv6]
addr-gen-mode=eui64
method=auto

[proxy]
```

重载网络配置文件

```shell
[root@dev ~]# nmcli connection reload
[root@dev ~]# nmcli connection up ens160
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/5)
```

测试

```shell
[root@dev ~]# ping baidu.com
PING baidu.com (182.61.244.181) 56(84) bytes of data.
64 bytes from 182.61.244.181 (182.61.244.181): icmp_seq=1 ttl=51 time=12.7 ms
64 bytes from 182.61.244.181 (182.61.244.181): icmp_seq=2 ttl=51 time=11.8 ms
```

为何能ping通公网域名？

- 任何符合规范的DNS软件天生都知道根服务器地址

```shell
[root@dev ~]# cat /var/named/named.ca
;; ANSWER SECTION:
.			518400	IN	NS	a.root-servers.net.
.			518400	IN	NS	b.root-servers.net.
.			518400	IN	NS	c.root-servers.net.
.			518400	IN	NS	d.root-servers.net.
.			518400	IN	NS	e.root-servers.net.
.			518400	IN	NS	f.root-servers.net.
.			518400	IN	NS	g.root-servers.net.
.			518400	IN	NS	h.root-servers.net.
.			518400	IN	NS	i.root-servers.net.
.			518400	IN	NS	j.root-servers.net.
.			518400	IN	NS	k.root-servers.net.
.			518400	IN	NS	l.root-servers.net.
.			518400	IN	NS	m.root-servers.net.

;; ADDITIONAL SECTION:
a.root-servers.net.	518400	IN	A	198.41.0.4
b.root-servers.net.	518400	IN	A	170.247.170.2
c.root-servers.net.	518400	IN	A	192.33.4.12
d.root-servers.net.	518400	IN	A	199.7.91.13
e.root-servers.net.	518400	IN	A	192.203.230.10
f.root-servers.net.	518400	IN	A	192.5.5.241
g.root-servers.net.	518400	IN	A	192.112.36.4
h.root-servers.net.	518400	IN	A	198.97.190.53
i.root-servers.net.	518400	IN	A	192.36.148.17
j.root-servers.net.	518400	IN	A	192.58.128.30
k.root-servers.net.	518400	IN	A	193.0.14.129
l.root-servers.net.	518400	IN	A	199.7.83.42
m.root-servers.net.	518400	IN	A	202.12.27.33
a.root-servers.net.	518400	IN	AAAA	2001:503:ba3e::2:30
b.root-servers.net.	518400	IN	AAAA	2801:1b8:10::b
c.root-servers.net.	518400	IN	AAAA	2001:500:2::c
d.root-servers.net.	518400	IN	AAAA	2001:500:2d::d
e.root-servers.net.	518400	IN	AAAA	2001:500:a8::e
f.root-servers.net.	518400	IN	AAAA	2001:500:2f::f
g.root-servers.net.	518400	IN	AAAA	2001:500:12::d0d
h.root-servers.net.	518400	IN	AAAA	2001:500:1::53
i.root-servers.net.	518400	IN	AAAA	2001:7fe::53
j.root-servers.net.	518400	IN	AAAA	2001:503:c27::2:30
k.root-servers.net.	518400	IN	AAAA	2001:7fd::1
l.root-servers.net.	518400	IN	AAAA	2001:500:9f::42
m.root-servers.net.	518400	IN	AAAA	2001:dc3::35
```

