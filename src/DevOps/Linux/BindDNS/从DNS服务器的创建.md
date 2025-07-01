---
order: 6
date: 2025-07-01
---

# 从DNS服务器的创建

## 简介

### 引入

只有一台主DNS服务器，存在单点失败的问题。可以建立主DNS服务器的备份服务器，即从服务器来实现DNS服务的容错机制。从服务器可以自动和主服务器进行单向的数据同步，从而和主DNS服务器一样也可以对外提供查询服务，但从服务器不提供数据更新服务。

### 要求

- 应该为一台独立的名称服务器
- 主服务器的区域解析库文件中必须有一条NS记录指向从服务器
- 从服务器只需要定义区域，而无须手动书写从服务器的解析库文件 (解析库文件同步后会自动放置于`/var/named/slaves/`目录中）
- 同步过后的解析库文件是加密的，无法读取与修改
- 主服务器得允许从服务器作区域传送
- 主从服务器时间应该同步，可通过ntp进行
- bind程序的版本应该保持一致；否则，应该从高，主低

### 从服务器定义区域

格式：

```shell
[root@dev named]# cat /etc/named.rfc1912.zones
zone "ZONE_NAME" IN{
  type slave;
  masters { MASTER_IP; };
  file "slaves/ZONE_NAME.zone";
}；
```

## 实现从服务器案例

> 注意修改主配置文件 `/etc/named.conf `，保持与主服务器一致

### 修改区域配置文件

`/etc/named.rfc1912.zones`

```c
zone "monap.cn" IN {
  type slave;
  masters { 192.168.1.149; };
  file "named.monap.cn";
};

zone "1.168.192.in-addr.arpa." IN {
  type slave;
  masters { 192.168.1.149; };
  file "named.1.168.192.in-addr.arpa";
};
```

### 主服务器推变更配置

主服务器中必须要配置从服务器的NS记录，否则主服务无法知道从服务器的地址，也就无法实现主服务配置变更实时推送到从服务，只能靠从服务器自己定时拉取同步了。

### 重启从服务器

重启从服务器后，观察`/var/named/slaves/`目录是否有从主服务器同步过来的解析库文件
