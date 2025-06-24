---
order: 1
date: 2025-06-20
---
# 简介

## 理解

gRPC 是由google开源的一个高性能的RPC框架。Stubby Google内部的RPC,演化而来的，2015正式开源。云原生时代是一个RPC标准。

## 设计思路

- 网络通信：gRPC自己封装网络通信的部分，提供多种语言的网络通信的封装（C Java[Netty] GO）
- 协议：HTTP2 传输数据的时候是二进制数据内容。 支持双向流（双工）连接的多路复用。
- 序列化：基于二进制实现序列化，使用`protobuf (Protocol Buffers) `实现序列化（google开源一种序列化方式，时间效率和空间效率大概是JSON的3到5倍）。
- 代理的创建：让调用者像调用本地方法那样 去调用远端的服务方法。

## gRPC与ThriftRPC区别

### 共性

支持异构语言的RPC

### 区别

|          | GRPC   | ThriftRPC |
| -------- | ------ | --------- |
| 网络通信 | Http2  | 专属协议  |
| 性能角度 | 快     | 很快      |
| 社区     | 很活跃 | 正常      |

### gRPC好处

- 高效的进行进程间通信。
- 支持多种语言 原生支持 C  Go Java实现。C语言版本上扩展 C++ C# NodeJS Python Ruby PHP..
- 支持多平台运行 Linux Android IOS MacOS Windows。
- gPRC序列化方式采用protobuf，效率高。
- 使用Http2协议
- 大厂的背书
