---
order: 2
date: 2025-08-23
---

# OpenSSH服务

## 理解

OpenSSH是SSH（Secure Shell）协议的免费开源实现，一般在各种Linux版本中会默认安装，基于C/S架构

OpenSSH软件相关包

- openssh
- openssh-clients
- openssh-server

服务器：

- sshd

客户端

- Linux Client：ssh、scp、sftp、slogin
- Windows Client：xshell、MobaXterm

## 客户端ssh命令

ssh命令是ssh客户端，允许实现对远程系统经验证地加密安全访问。

当用户远程连接ssh服务器时，会复制ssh服务器`/etc/ssh/ssh_host*key.pub`文件中的公钥到客户机的`~/.ssh/know_hosts`中。下次连接时，会自动匹配相应私钥，不能匹配将拒绝连接。

ssh客户端配置文件：/etc/ssh/ssh_config

### 主要配置

```shell
#StricHostKeyChecking ask
# IdentityFile ~/.ssh/id_rsa
# IdentityFile ~/.ssh/id_dsa
# IdentityFile ~/.ssh/id_ecdsa
# IdentityFile ~/.ssh/id_ed25519
# Port 22
```

### 格式

```shell
ssh [user@]host [COMMAND]
ssh [-l user] host [COMMAND]
```

### 常见选项

```shell
-p   # port: 远程服务器监听的端口
-b   # 指定连接的源IP
-v   # 调试模式
-C   # 压缩方式
-x   # 支持x11转发
-t   # 强制伪tty分配，如 ssh -t remoteserver1 ssh -t remoteserver 2 ssh remoteserver3
-o   # option 如 -o StricHostKeyChecking=no
```

### 范例-在远程主机上执行命令

语法：

`ssh user@hostname command ` 即直接在远程登录的语法后添加要执行的命令即可。

示例：

```shell
ssh user@192.168.205.xx "ls -alh | grep root"
```

说明：

在IP地址为192.168.205.x的远程主机上执行命令`ls -alh | grep root`，并将结果返回本地。
