---
order: 3
date: 2025-11-02
---

#  Namespace

### 概念

Namespace提供了一种将集群资源逻辑上隔离的方式，允许在同一个集群中划分多个虚拟的、逻辑上独立的集群环境，相当于集群的“虚拟化”。 

Namespace经常用于多个团队和多个项目的场景，可以按照不同的环境划分Namespace， 或者按照不同的团队及租户划分Namespace。

### 作用

- **资源隔离**：不同团队或项目可以拥有自己独立的Namespace，以防止资源相互干扰
- **权限控制**：可以为不同的Namespace设置不同的访问权限，实现不同的用户具有不同的权限
- **环境拆分**：使用Namespace可以模拟出多个虚拟的集群环境，如开发、测试和生产环境。每个环境可以有自己的资源和服务，相互之间保持隔离，有助于简化部署和管理
- **资源配额和限制**：划分不同的Namespace可以更加有效的分配资源和限制资源的使用量
- **服务发现和负载均衡**：在同一个Namespace中服务发现和负载均衡更加简单和高效
- **简化管理**：拆分不同的Namespace，可以更加方便的对Namespace下的资源进行操作，比如删除、备份或迁移等

### 基础命名空间

- **default**： 默认命名空间，在未指定命名空间时，即表示为default
- **kube-node-lease**：此空间保存与每个节点关联的租约（Lease）对象
- **kube-public**：公开的命名空间可以被任何用户访问，包括未授权的用户
- **kube-system**： Kubernetes系统组件所在的命名空间

### 基本使用

#### 创建

```shell
kubectl create ns NAMESPACE_NAME
```

#### 通过Yaml创建

```yaml
apiVersion:v1
  kind: Namespace
  metadata: 
    name: development 
```

#### 删除

```shell
kubectl delete ns NAMESPACE_NAME
```

#### 查看

```shell
kubectl get ns NAMESPACE_NAME --show-labels
```

> Namespace名字限制：最多63个字符，只能包含字母、数字、和中横线-，并且开头和结尾只能是数字和字母