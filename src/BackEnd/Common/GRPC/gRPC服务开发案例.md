---
order: 4
date: 2025-06-22
---
# gRPC服务开发案例

## 项目结构

xxxx-api 模块 

- 定义 protobuf idl语言 (message、service)
- 并且通过protoc编译器命令将idl语言转换为对应的语言代码

xxxx-server模块

- 实现api模块中定义的服务接口
- 发布gRPC服务 (创建服务端程序)

xxxx-client模块

- 创建服务端stub(代理)
- 基于代理（stub) RPC调用  

## api 模块

### 开发过程

- .proto文件 书写protobuf的IDL

- [了解]protoc命令 把proto文件中的IDL 转换成编程语言 

```protobuf
protoc --java_out=/xxx/xxx  /xxx/xxx/xx.proto
```

- [实战] [maven插件](https://github.com/grpc/grpc-java)进行protobuf IDL文件的编译，并把他放置IDEA具体位置。

### client包结构

```shell
.
├── pom.xml
└── src
    └── main
        ├── java
        │   └── com
        │       └── heluox
        │           └── api
        │               ├── HelloProto.java           # 自动生成的message
        │               └── HelloServiceGrpc.java     # 自动生成的service
        └── proto
            └── hello.proto
```

### .proto内容

```protobuf
syntax = "proto3";

option java_multiple_files = false;
option java_package = "com.heluox.cloud.machine.api";
option java_outer_classname = "HelloProto";

message HelloRequest {
    string name = 1;
}

message HelloResponse {
    string result = 1;
}

service HelloService {
    rpc hello(HelloRequest) returns (HelloResponse) {}
}
```

### pom.xml配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
   <!--- ... --->
    <dependencies>
        <dependency> <!-- gRPC 基于netty的java原生实现 -->
            <groupId>io.grpc</groupId>
            <artifactId>grpc-netty-shaded</artifactId>
            <version>${grpc.version}</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-protobuf</artifactId>
            <version>${grpc.version}</version>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-stub</artifactId>
            <version>${grpc.version}</version>
        </dependency>
        <dependency> <!-- necessary for Java 9+ -->
            <groupId>org.apache.tomcat</groupId>
            <artifactId>annotations-api</artifactId>
            <version>${annotations-api.version}</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>

    <build>
        <extensions>
            <extension>
                <groupId>kr.motd.maven</groupId>
                <artifactId>os-maven-plugin</artifactId>
                <version>${os-maven-plugin.version}</version>
            </extension>
        </extensions>
        <plugins>
            <plugin>
                <groupId>org.xolstice.maven.plugins</groupId>
                <artifactId>protobuf-maven-plugin</artifactId>
                <version>${protobuf-maven-plugin.version}</version>
                <configuration>
                    <!-- 用于生成message-->
                    <!--suppress UnresolvedMavenProperty -->
                    <protocArtifact>com.google.protobuf:protoc:3.25.5:exe:${os.detected.classifier}</protocArtifact>
                    <pluginId>grpc-java</pluginId>
                    <!-- 用于生成服务接口 -->
                    <!--suppress UnresolvedMavenProperty -->
                    <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.73.0:exe:${os.detected.classifier}</pluginArtifact>
                    <!-- 指定生成转换后代码目录，不指定默认输出到target中 -->
                    <outputDirectory>${basedir}/src/main/java</outputDirectory>
                    <clearOutputDirectory>false</clearOutputDirectory>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal> <!-- 将生成message的命令绑定到goal -->
                            <goal>compile-custom</goal> <!-- 将生成服务接口的命令绑定到goal -->
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

### 生成后Message结构

```shell
# 因为HelloRequest和HelloResponse都是内部类，所有使用时都是HelloProto.HelloRequest这种方式
HelloProto
> HelloRequestOrBuilder
> HelloRequest
> HelloResponseOrBuilder
> HelloResponse
```

### 生成后Service结构

```shell
HelloServiceGrpc
> AsyncService
> HelloServicelmplBase       # 对应真正的服务接口，后续开发时就需要继承这个类并覆盖其中的业务方法
# 下面几个以stub结尾的类对应的就是client的代理对象，这些stub都是client代理，各个stub的区别就是网络通信方式不同
> HelloServiceStub           # 监听方式的异步stub
> HelloServiceBlockingV2Stub # 阻塞stub
> HelloServiceBlockingStub   # 阻塞stub
> HelloServiceFutureStub     # 类似netty future的同步与异步都支持的stub
```

## service模块

### 开发过程

- 实现业务接口，添加具体的功能 
- 创建服务端 （Netty)

### 实现业务接口 

```java
/**
 * 测试gRPC
 */
public class HelloServiceImpl extends HelloServiceGrpc.HelloServiceImplBase {
    /*
      1. 接受client提交的参数  request.getParameter()
      2. 业务处理 service+dao 调用对应的业务功能。
      3. 提供返回值
     */
    @Override
    public void hello(HelloProto.HelloRequest request, StreamObserver<HelloProto.HelloResponse> responseObserver) {
        //1.接受client的请求参数
        String name = request.getName();
        //2.业务处理
        System.out.println("name parameter "+name);
        //3.封装响应
        //3.1 创建相应对象的构造者
        HelloProto.HelloResponse.Builder builder = HelloProto.HelloResponse.newBuilder();
        //3.2 填充数据
        builder.setResult("hello method invoke ok");
        //3.3 封装响应
        HelloProto.HelloResponse helloResponse = builder.build();

        responseObserver.onNext(helloResponse);
        responseObserver.onCompleted();;
    }
}
```

### 创建服务端

```java
/**
 * gRPC服务端
 */
public class GRPCServer {
    public static void main(String[] args) throws IOException, InterruptedException {
        //1. 绑定端口
        ServerBuilder<?> serverBuilder = ServerBuilder.forPort(9000);
        //2. 发布服务
        serverBuilder.addService(new HelloServiceImpl());
        //serverBuilder.addService(new UserServiceImpl());
        //3. 创建服务对象
        Server server = serverBuilder.build();

        server.start();
        server.awaitTermination();
    }
}
```

## client模块

### 开发过程

- client通过代理对象完成远端对象的调用

### 远端调用

```java
/**
 * gRPC 客户端测试
 */
public class GRPCClient {
    public static void main(String[] args) {
        //1 创建通信的管道
        ManagedChannel managedChannel = ManagedChannelBuilder.forAddress("localhost", 9000).usePlaintext().build();
        //2 获得代理对象 stub
        try {
            HelloServiceGrpc.HelloServiceBlockingStub helloService = HelloServiceGrpc.newBlockingStub(managedChannel);
            // 3. 完成RPC调用
            // 3.1 准备参数
            HelloProto.HelloRequest.Builder builder = HelloProto.HelloRequest.newBuilder();
            builder.setName("rose");
            HelloProto.HelloRequest helloRequest = builder.build();
            // 3.1 进行功能rpc调用，获取相应的内容
            HelloProto.HelloResponse helloResponse = helloService.hello(helloRequest);
            String result = helloResponse.getResult();
            System.out.println("result = " + result);
        } catch (Exception e) {
            throw new RuntimeException(e);
        } finally {
            managedChannel.shutdown();
        }
    }
}
```

## 注意项

```java
// 服务端 处理返回值时
// 通过这个方法 把响应的消息回传client
responseObserver.onNext(helloResponse1);
// 通知client 整个服务结束，底层返回标记 
responseObserver.onCompleted();
                                          
// client就会监听标记 【grpc做的】
requestObserver.onNext(helloRequest1);
requestObserver.onCompleted();
```