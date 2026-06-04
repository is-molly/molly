---
order: 11
date: 2025-06-23
---

# gRPC与Springboot高级整合

## 拦截器

### 开发拦截器

```java
public class CustomBootInterceptor implements ClientInterceptor {
    @Override
    public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(MethodDescriptor<ReqT, RespT> method, CallOptions callOptions, Channel next) {
        System.out.println("这是拦截器 起作用了....");
        return next.newCall(method, callOptions);
    }
}
```

### 引入拦截器

```java
@Configuration
public class CustomBootInterceptorConfiguration {
    //@Bean 让spring容器 认识这个Bean ,那Spring容器会不会把他当成grpc的拦截器呢？
    @GrpcGlobalClientInterceptor
    public CustomBootInterceptor customBootInterceptor() {
        return new CustomBootInterceptor();
    }
}
```

### 第二种引入拦截器的方式

```java
@GrpcGlobalClientInterceptor
public class CustomBootInterceptor implements ClientInterceptor {
    @Override
    public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(MethodDescriptor<ReqT, RespT> method, CallOptions callOptions, Channel next) {
        System.out.println("这是拦截器 起作用了....");
        return next.newCall(method, callOptions);
    }
}
```

## consul注册中心

> grpc替换 springcloud中 openFeign

### 服务端开发

#### 依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
    <version>3.1.2</version>
 </dependency>

<!--健康检查  http协议 -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-actuator</artifactId>
</dependency>
```

#### application.yml

```yml
spring:
  application:
    name: boot-server
  cloud:
    consul:
      discovery:
        register: true
        hostname: 127.0.0.1
        port: 8500
        heartbeat:
          enabled: true

#  main:
#    web-application-type: none


grpc:
  server:
    port: 9000
```

#### 启动类

```java
@EnableDiscoveryClient
```

#### 注意事项

- 进行健康检查 `heartbeat.enabled = true`

- 服务端做集群

- 替换注册中心（consul zk nacos）https://github.com/yidongnan/grpc-spring-boot-starter

  - 替换cloud 依赖jar

  - application.yml：ip+port 

### 客户端开发

#### 依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
    <version>3.1.2</version>
</dependency>

<!--健康检查  http协议 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-actuator</artifactId>
</dependency>
```

#### application.yml

```yml
grpc:
  client:
    cloud-grpc-server:
      address: 'discovery://boot-server'
      negotiation-type: plaintext
      enable-keep-alive: true
      keep-alive-without-calls: true
```

#### 注入

通过以下注解注入

```java
@GrpcClient("cloud-grpc-server")
```

