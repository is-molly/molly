---
order: 3
date: 2025-06-29
---

# bind-utils

## 理解

bind-utils是bind软件提供的一组DNS工具包，里面有一些DNS相关的工具。主要：dig，host，nslookup，nsupdate。使用这些工具可以进行域名解析和DNS调试工作。

## host

### 简介

host命令是一款用于查询主机相关信息的命令。它可以用来查询主机的IP地址、域名的IP地址、反向查询IP地址对应的域名等。

### 参数

```shell
[root@dev named]# host --help
host: illegal option -- -
Usage: host [-aCdilrTvVw] [-c class] [-N ndots] [-t type] [-W time]
            [-R number] [-m flag] [-p port] hostname [server]
       -a is equivalent to -v -t ANY
       -A is like -a but omits RRSIG, NSEC, NSEC3
       -c specifies query class for non-IN data
       -C compares SOA records on authoritative nameservers
       -d is equivalent to -v
       -l lists all hosts in a domain, using AXFR
       -m set memory debugging flag (trace|record|usage)
       -N changes the number of dots allowed before root lookup is done
       -p specifies the port on the server to query
       -r disables recursive processing
       -R specifies number of retries for UDP packets
       -s a SERVFAIL response should stop query
       -t specifies the query type
       -T enables TCP/IP mode
       -U enables UDP mode
       -v enables verbose output
       -V print version number and exit
       -w specifies to wait forever for a reply
       -W specifies how long to wait for a reply
       -4 use IPv4 query transport only
       -6 use IPv6 query transport only
```


### 语法

基本语法

```shell
host [option] [params]
```

选项

```shell
-a             # 显示详细的DNS信息；
-c<类型>        # 指定查询类型，默认值为“IN“；
-C             # 查询指定主机的完整的SOA记录；
-r             # 在查询域名时，不使用递归的查询方式；
-t<类型>        # 指定查询的域名信息类型；
-v             # 显示指令执行的详细信息；
-w             # 如果域名服务器没有给出应答信息，则总是等待，直到域名服务器给出应答；
-W<时间>        # 指定域名查询的最长时间，如果在指定时间内域名服务器没有给出应答信息，则退出指令；
-4             # 使用IPv4；
-6             # 使用IPv6.三、host命令的基本使用

```

### 基本使用

#### 查询域名的IP地址

```shell
[root@dev named]# host www.baidu.com
www.baidu.com is an alias for www.a.shifen.com.
www.a.shifen.com has address 36.152.44.132
www.a.shifen.com has address 36.152.44.93
www.a.shifen.com has IPv6 address 2409:8c20:6:1794:0:ff:b080:87f0
www.a.shifen.com has IPv6 address 2409:8c20:6:123c:0:ff:b0f6:b2d
```

#### 查看ip地址对应的域名

查看ip地址对应的域名，例如查询DNS `8.8.8.8` 对应的域名。

```shell 
[root@dev named]# host 8.8.8.8
8.8.8.8.in-addr.arpa domain name pointer dns.google.
```


#### 查询NS记录

查询NS记录（域名服务器）

```shell
[root@dev named]# host -t  NS www.baidu.com
www.baidu.com is an alias for www.a.shifen.com.
```

#### 查询详细信息

使用-a选项，查询详细信息

```shell
[root@dev named]# host -at A www.baidu.com
Trying "www.baidu.com"
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12260
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;www.baidu.com.			IN	A

;; ANSWER SECTION:
www.baidu.com.		161	IN	A	36.152.44.132
www.baidu.com.		161	IN	A	36.152.44.93

Received 63 bytes from fe80::1%2#53 in 10 ms
```

#### 显示命令执行过程

使用-v选项，显示命令的执行过程。

```shell
root@dev named]# host -v 8.8.8.8
Trying "8.8.8.8.in-addr.arpa"
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 7198
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;8.8.8.8.in-addr.arpa.		IN	PTR

;; ANSWER SECTION:
8.8.8.8.in-addr.arpa.	2996	IN	PTR	8.8.8.8.in-addr.arpa.

Received 72 bytes from fe80::1%2#53 in 0 ms
```

### host命令使用注意事项

- host命令用于查询和解析主机名和IP地址之间的关系，可以用来查看主机名对应的IP地址或者IP地址对应的主机名。
- host命令默认使用本地DNS解析，可以通过使用"-a"参数来强制使用指定的DNS服务器进行解析。
- host命令可以接受一个或多个参数，每个参数可以是一个主机名或者一个IP地址。
- 如果输入参数为主机名，host命令会返回该主机名对应的IP地址。如果输入参数为IP地址，host命令会返回该IP地址对应的主机名。
- 当host命令无法解析主机名或者IP地址时，会返回相应的错误信息。
- 使用host命令时，可以通过添加额外的选项来控制输出格式，例如使用"-t"参数来指定查询的类型，使用"-v"参数来显示更详细的信息。
- host命令可以用来检测DNS解析是否正常，以及查询某个主机名对应的IP地址是否正确。
- 在某些情况下，host命令可能无法解析某些特殊的域名或者IP地址，这时可以尝试使用其他的解析工具，如nslookup或dig命令。

## nslookup

### 简介
nslookup 是一个用于查询 DNS（域名系统）记录的命令行工具。它可以帮助用户获取域名解析信息，例如域名对应的 IP 地址、DNS 服务器的记录等，常用于网络故障排查、域名解析分析以及网络安全渗透测试等场景。

nslookup 可以在交互模式和非交互模式下运行：

- 交互模式：允许用户输入多个查询命令并查看结果。

- 非交互模式：适用于单次查询命令，直接返回查询结果。

nslookup 支持查询多种类型的 DNS 记录，如 A 记录、MX 记录、NS 记录等，可以指定使用的 DNS 服务器进行查询。

### 常用命令
以下是 nslookup 工具的常用语法和命令，按功能分类并附有说明：

| **命令/选项**               | **描述**                                                     | **示例命令**                               |
| --------------------------- | ------------------------------------------------------------ | ------------------------------------------ |
| **nslookup [domain]**       | 查询指定域名的 A 记录（即域名对应的 IP 地址）。              | `nslookup www.example.com`                 |
| **nslookup [domain] [dns]** | 查询指定域名的解析结果，使用指定的 DNS 服务器进行查询。      | `nslookup www.example.com 8.8.8.8`         |
| **set type=[type]**         | 设置查询的记录类型，如 A、MX、NS、CNAME、TXT 等。            | `set type=MX` `nslookup www.example.com`   |
| **set q=[type]**            | 设置查询类型，类似于 `set type`。例如，可以查询 MX 记录、NS 记录等。 | `set q=NS` `nslookup example.com`          |
| **set debug**               | 开启调试模式，显示 DNS 查询过程中的详细信息。                | `set debug` `nslookup www.example.com`     |
| **set all**                 | 显示所有配置信息，包括默认的查询类型和 DNS 服务器。          | `set all` `nslookup`                       |
| **exit**                    | 退出 `nslookup` 命令行界面。                                 | `exit`                                     |
| **server [dns]**            | 指定要查询的 DNS 服务器，可以在交互模式下指定新的服务器。    | `server 8.8.8.8` `nslookup`                |
| **set timeout=[seconds]**   | 设置查询超时时间（秒）。                                     | `set timeout=5` `nslookup www.example.com` |
| **set retry=[count]**       | 设置查询重试次数，默认是 4 次。                              | `set retry=2` `nslookup www.example.com`   |
| **set port=[port]**         | 设置 DNS 服务器的端口号，默认为 53。                         | `set port=5353` `nslookup www.example.com` |
| **set norecurse**           | 禁用递归查询，仅查询指定的 DNS 服务器的直接答案，不做递归查询。 | `set norecurse` `nslookup www.example.com` |
| **set recurse**             | 启用递归查询，默认启用。                                     | `set recurse` `nslookup www.example.com`   |
| **ls -d [domain]**          | 列出域名的 DNS 记录（例如 A、MX、NS、TXT 记录），需要管理员权限。 | `ls -d example.com`                        |
| **set vc**                  | 强制使用 TCP 协议进行 DNS 查询（而非默认的 UDP 协议）。      | `set vc` `nslookup www.example.com`        |

### 常见的 DNS 记录类型（set type 或 set q）
| **记录类型** | **描述**                                                     |
| ------------ | ------------------------------------------------------------ |
| **A**        | 域名到 IPv4 地址的映射。                                     |
| **AAAA**     | 域名到 IPv6 地址的映射。                                     |
| **MX**       | 域名的邮件交换记录（Mail Exchange），用于指定接收邮件的服务器。 |
| **NS**       | 域名的名称服务器记录，指定负责解析该域名的 DNS 服务器。      |
| **CNAME**    | 域名的别名记录，将一个域名指向另一个域名。                   |
| **TXT**      | 域名的文本记录，常用于 SPF、DKIM 等邮件验证机制。            |
| **SOA**      | 域名的起始授权机构记录，包含该域名的主要 DNS 信息，如主服务器和序列号等。 |
| **PTR**      | 反向解析记录，将 IP 地址映射到域名。                         |
| **SRV**      | 服务记录，指定某个服务（例如 SIP、LDAP）的具体服务器和端口。 |

### 常用示例及应用建议

| **功能**                                 | **命令示例**                          | **使用建议**                                                 |
| ---------------------------------------- | ------------------------------------- | ------------------------------------------------------------ |
| **查询域名的 IP 地址（A 记录）**         | `nslookup www.example.com`            | 通过该命令可以快速查看域名对应的 IP 地址，用于故障排查、网络连接测试等场景。 |
| **查询某个域名的 MX 记录（邮件服务器）** | `nslookup -type=MX example.com`       | 可用于检查邮件服务器配置，特别是在配置邮件服务时。           |
| **查询域名的 NS 记录（域名服务器）**     | `nslookup -type=NS example.com`       | 用于查询某个域名的权威 DNS 服务器，帮助定位 DNS 配置问题。   |
| **查询特定类型的 TXT 记录**              | `nslookup -type=TXT example.com`      | 用于检查 SPF、DKIM 等邮件验证机制的配置，验证邮件服务器是否正确设置。 |
| **指定 DNS 服务器进行查询**              | `nslookup www.example.com 8.8.8.8`    | 在 DNS 故障排查时，可以尝试使用不同的 DNS 服务器进行查询，确保问题不在本地 DNS 服务器。 |
| **开启调试模式，显示详细查询过程**       | `nslookup -debug www.example.com`     | 在排查 DNS 配置问题时，开启调试模式可以看到查询过程的详细信息，帮助分析具体的故障原因。 |
| **查询 DNS 记录时禁用递归查询**          | `nslookup -norecurse www.example.com` | 使用时可以减少 DNS 服务器的负担，适用于查询时不需要递归查找的场景，通常用于高级排查。 |

### 易错点注意事项
查询类型设置不当：在查询时，如果没有明确指定查询的记录类型（如 A、MX），则默认查询类型为 A 记录。确保查询类型正确，避免得到不相关的结果。

- 错误示例：nslookup example.com（默认查询 A 记录）

- 正确示例：nslookup -type=MX example.com（查询 MX 记录）

DNS 服务器选择：如果没有指定 DNS 服务器，nslookup 将默认使用本地 DNS 服务器。为了确保查询结果的准确性，可以明确指定一个公共 DNS 服务器（如 Google DNS、Cloudflare DNS）。

- 错误示例：nslookup www.example.com（使用本地 DNS 服务器）

- 正确示例：nslookup www.example.com 8.8.8.8（使用 Google 公共 DNS）

递归与非递归查询：在某些情况下，nslookup 的递归查询可能导致请求返回大量数据。若不需要递归查询，请使用 set norecurse 以避免不必要的查询。

- 错误示例：使用递归查询时未考虑 DNS 服务器负担。

- 正确示例：set norecurse（避免递归查询）

命令拼写错误：nslookup 中常见的命令选项如 set type，拼写错误或遗漏可能导致不正确的查询结果。

- 错误示例：set q=MX（不常见的拼写错误，正确命令为 set type=MX）

DNS 缓存问题：某些 DNS 服务器会缓存 DNS 记录，导致查询结果延迟更新。可以尝试清空缓存或使用不同的 DNS 服务器进行查询。

### 总结
nslookup 是一个强大的 DNS 查询工具，广泛用于网络故障排查、域名解析分析等。掌握其常用命令和选项能够帮助用户快速定位 DNS 问题。在

使用时，需注意命令拼写、查询类型、DNS 服务器选择等细节，确保得到准确的查询结果。

## dig

### 简介

- dig（domain information groper）是域名查询工具。

- dig 是一个灵活的 DNS 查询工具，它会打印出 DNS 域名服务器的回应，主要用来从 DNS 域名服务器查询主机地址信息。

- dig 命令与 [nslookup](https://so.csdn.net/so/search?q=nslookup&spm=1001.2101.3001.7020) 命令功能基本相同，但是 dig 命令灵活性好、易用、输出清晰。

### 常用命令

#### 查看本机公网IP

```shell
dig ANY +short @resolver2.opendns.com myip.opendns.com
```

#### 查看本机使用的DNS地址

```shell
dig
```

#### 查询A记录

```shell
dig baidu.com
```

#### 指定DNS服务器查询域名

```shell
dig @1.1.1.1 google.com
```

##### 指定DNS服务器端口

有些DNS使用了非标准的端口。

```shell
# +short：只显示查询结果中的 IP 地址
# @ 指定DNS服务器地址
# -p 指定dns服务器的端口
dig +short google.com @208.67.222.222 -p 5353
```

##### 使用TCP协议查询解析

```shell
# +tcp：使用 TCP 协议进行 DNS 查询
dig +tcp google.com @8.8.8.8
```

##### 指定查询来源子网

```shell
# 在指定的DNS服务器上查询域名example.com的IP地址，并指定查询来源的子网地址和子网掩码长度。
#   @server选项表示指定DNS服务器的IP地址或主机名
#   addr表示查询来源的子网地址
#   prefix-length表示查询来源的子网掩码长度
dig @server example.com +subnet=addr[/prefix-length]

# 示例
dig @8.8.8.8 google.com +subnet=192.168.1.0/24
```

#### 使用dot或doh查询域名解析

使用dot或doh能一定程度上减少`dns污染`的概率

```shell
# 使用dot或doh查询域名解析
dig +short myip.opendns.com @resolver1.opendns.com
```

#### 使用DNSSEC查询域名解析

```shell
# +dnssec: 启用 DNSSEC 功能，表示查询结果需要进行数字签名验证
# +short: 显示查询结果中的简洁信息，只显示域名和 IP 地址
dig +dnssec +short google.com
```

#### 查询dns所有记录值any

```shell
# 查询dns所有记录值
dig @223.5.5.5 aliyun.com ANY +noall +answer
```

#### 从ip地址反查询域名`dig -x`

```shell
# 从ip地址反查询域名
dig -x 192.168.101.2

# 还有一种原始写法
dig -t ptr 2.101.168.192.in-addr.arpa
```

#### 使用tcp协议进行查询

```shell
# 使用tcp协议
dig aliyun.com +tcp
```

#### 查询域名的NS记录

NS记录（Name Server Record）是DNS中的一种记录类型，用于指定该域名的DNS服务器。NS记录通常由域名注册商或DNS服务提供商设置，用于将域名与DNS服务器关联起来。

当用户在浏览器中输入一个域名时，浏览器会向本地DNS服务器发送一个DNS查询请求，本地DNS服务器会根据该域名的NS记录来确定该域名的DNS服务器，并向该DNS服务器发起查询。如果该DNS服务器无法解析该域名，则会向上一级DNS服务器发起查询，直到找到能够解析该域名的DNS服务器为止。

NS记录通常包含两个部分：域名和DNS服务器地址。例如，一个NS记录可能是：

| 域名         |      | 记录类型 | DNS服务器地址    |
| ------------ | ---- | -------- | ---------------- |
| example.com. | IN   | NS       | ns1.example.com. |

`example.com`是`域名`，`NS`表示该`记录类型`为`NS记录`，`ns1.example.com`是`DNS服务器地址`。

```shell
# 查询域名的NS记录
dig aliyun.com +nssearch
```

#### 检查txt记录是否生效

```shell
# dig检查txt记录是否生效
dig -t txt _acme-challenge.arzar.net
```

#### 查看`DNS`是否开启`AXFR`协议全量区传输功能

```shell
# 查看DNS是否开启AXFR协议全量区传输功能
dig dns.google axfr
```

### dig诊断`DNS污染`

查询`权威dns`和`缓存dns`,判断`递归解析`过程`哪个环节`被 `污染`

#### 只显示域名的解析ip

`dig +short`提供简要答复,只返回解析的ip

```shell
# 对域名进行两次DNS查询
# dig +short 要解析的域名 @dns地址
dig +short aliyun.com @208.67.222.222

# 对域名进行两次DNS查询--使用自定义dns端口
# dig +short 要解析的域名 @dns地址 -p dns的端口
dig +short aliyun.com @208.67.222.222 -p 5353

# 只查看A记录
dig A +short 1dot1dot1dot1.cloudflare-dns.com
```

#### 递归查询

递归查询的过程可以类比为一个人在询问路线的过程。当一个人询问路线时，如果对方不知道具体的路线，就会向其他人或者地图上查找相关信息，直到找到能够指引路线的人或者信息为止。

递归查询的过程也是类似的，DNS服务器会向其他DNS服务器发起查询请求，直到找到能够解析目标域名的DNS服务器为止。

递归查询的过程可以分为以下几个步骤：

- 本地DNS服务器接收到DNS查询请求，并根据该域名的NS记录来确定该域名的DNS服务器。
- 本地DNS服务器向该DNS服务器发起查询请求，如果该DNS服务器无法解析该域名，则会向上一级DNS服务器发起查询。
- 查询过程会一直向上级DNS服务器发起查询请求，直到找到能够解析该域名的DNS服务器为止。
- 当找到能够解析该域名的DNS服务器时，该DNS服务器会返回查询结果给本地DNS服务器。
- 本地DNS服务器将查询结果返回给用户。

```shell
# 递归查询
dig google.com +recurse
```

查询递归查询过程dig +trace(从权威dns查询解析)
从根域名服务器开始，逐步追踪域名解析的过程;
+trace参数将会显示域名查询的完整递归路径，跟踪每个过程的返回内容，包括每个DNS服务器的IP地址和响应时间。

权威解析需要从根开始去迭代查询,
每次去查询NS的迭代工作就由本机完成，而不是递归服务器完成递归解析的过程:

```shell
# 递归解析
dig www.aliyun.com @223.5.5.5 +trace

## -4 只查看IPV4
dig -4 www.aliyun.com @223.5.5.5 +trace
```

#### 从指定dns查询域名解析(查询缓存dns服务器)

```shell
# 从指定的dns查询域名解析
dig aliyun.com @223.5.5.5

# Windows指定dns要加引号'@dns地址'
# 或使用\防止转义
dig aliyun.com \@1.1.1.1
```

### dig的debug

```shell
# debug模式
dig google.com -d
```

