Istio路由规则配置：VirtualService
=============
- [Istio路由规则配置：VirtualService](#istio路由规则配置：virtualservice)
  - [一、理解 Istio 的服务模型](#一、理解istio的服务模型)
    - [1、Istio的服务](#1、istio的服务)
    - [Istio 的服务版本](#istio的服务版本)
  - [二、VirtualService](#二、virtualservice)
    - [路由规则配置示例](#路由规则配置示例)
    - [路由规则定义](#路由规则定义)

## 一、理解 Istio 的服务模型
Istio 支持将由服务、服务版本和服务实例构造的抽象模型映射到不同的平台上，基于Kubernetes 的场景, Istio 的几个资源对象就是基于 Kubernetes 的相应资源对象构建的，加上部分约束来满足 Istio 服务模型的要求。

* **端口命名**：对 Istio 的服务端口必须进行命名，而且名称只允许是 ＜protocol＞[-＜suffix＞] 这种格式，其中 ＜protocol＞ 可以是 tcp、http、http2、https、grpc、tls、mongo、mysql、redis 等，Istio 根据在端口上定义的协议来提供对应的路由能力。例如“name：http2-forecast”和“name：http”是合法的端口名，如果端口未命名或者没有基于这种格式进行命名，则端口的流量会被当作 TCP 流量来处理。
* **服务关联**：Pod 需要关联到服务，如果一个 Pod 属于多个 Kubernetes 服务，则要求服务不能在同一个端口上使用不同的协议。
* **Deployment 使用 app 和 version 标签**：建议 Kubernetes Deployment 显式地包含 app和 version 标签。每个 Deployment 都需要有一个有业务意义的 app 标签和一个表示版本的 version 标签。在分布式追踪时可以通过 app 标签来补齐上下文信息，还可以通过 app 和 version 标签为遥测数据补齐上下文信息。


### 1、Istio的服务
从逻辑上看，服务是 Istio 主要管理的资源对象，是一个抽象概念，主要包含 HostName 和 Ports 等属性，并指定了 Service 的域名和端口列表。每个端口都包含端口名称、端口号和端口的协议。

从物理层面看，Istio 服务的存在形式就是 Kubernetes 的 Service，在启用了 Istio 的集群中创建 Kubernetes 的 Service 时只要满足以上约束，就可以转换为 Istio 的 Service 并配置规则进行流量治理。
> Service 是 Kubernetes 的一个核心资源，用户通过一个域名或者虚拟的 IP 就能访问到后端 Pod，避免向用户暴露 Pod 地址的问题，特别是在 Kubernetes 中，Pod 作为一个资源创建、调度和管理的最小部署单元的封装，本来就是动态变化的，在节点删除、资源变化等多种情况下都可能被重新调度，Pod 的后端地址也会随之变化。

**一个Istio Service 示例**

```
apiversion: v1
kind: Service
metadata:
  name: details
spec:
  ports:
  - port: 3002
    tagetPort: 3002
    name: http        #Istio 服务的约束，在端口名称上指定协议
  selector:
   app: details
```
在 Istio 中，Service 是治理的对象，是 Istio 中的核心管理实体，所以在 Istio 中，Service 是一个提供了对外访问能力的执行体，可以将其理解为一个定义了服务的工作负载，没有访问方式的工作负载不是 Istio 的管理对象，Kubernetes 的 Service 定义就是 Istio 服务的元数据。

### Istio 的服务版本
在 Istio 中多个版本的定义是将一个 Service 关联到多个 Deployment ，每个Deployment 都对应服务的一个版本

**示例**
```
apiVersion: app/v1
kind: Deployment
metadata:
  name: details-v1
  labels:
    app: details
    version: v1
  spec:
    replica: 2
    template:
      metadata:
        labels:
          app: details
          version: v1
      spec:
        containers:
        - name: details
        - image: ***
        - ports:
          - containerPort: 3002 
```

## 二、VirtualService
VirtualService 是 Istio 流量治理的一个核心配置，也可以说是 Istio 流量治理中最重要、最复杂的规则。

### 路由规则配置示例
```
apiVersion: networking.istio.io/vlalpha3
kind: VirtualService
metadata:
  name: forecast
spec:
  hosts:
  - forecast
  http:
  - match:
    - headers:
        location:
          exact: north
    route:
    - destination:
        host: forecast
        subset: v2
  - route:
    - destination:
        host: forecast
        subset: v1  
```
Istio 的配置都是通过 Kubernetes 的 CRD（CustomerResourceDefinition 用户自定义资源）方式表达。此示例所配置的规则是：对于 forecast服务的访问，如果在请求的Header 中 location 取值是 north，则将该请求转发到服务的 v2 版本上，将其他请求都转发到服务的 v1 版本上。

### 路由规则定义
VirtualService 定义了对特定目标服务的一组流量规则。VirtualService 在形式上表示一个虚拟服务，将满足条件的流量都转发到对应的服务后端，这个服务后端可以是一个服务，也可以是在 DestinationRule 中定义的服务的子集。

<div align=center>
<img src="image/VirtualService路由规则图示.png" style="zoom:20%" />
</div>
<p align="center">图 VirtualService路由规则图示 （图源 《云原生服务网格Istio》）</p>

从上图中可以看到，除了 hosts、gateways 等通用字段，规则的主体是 http、tcp 和 tls，都是复合字段，分别对应 HTTPRoute、TCPRoute 和 TLSRoute，表示 Istio 支持的 HTTP、TCP 和 TLS 协议的流量规则。

1. **hosts**：是一个重要的必选字段，表示流量发送的目标。可以将其理解为VirtualService 定义的路由规则的标识，用于匹配访问地址，可以是一个 DNS 名称或IP 地址。
2. **gateways**：表示应用这些流量规则的 Gateway。服务只是在网格内访问的，gateways 字段可以省略。服务只是在网格外访问的，配置要关联的 Gateway，表示对应 Gateway 进来的流量执行在这个 VirtualService 上定义的流量规则。
3. **http**：是一个与 HTTPRoute 类似的路由集合，用于处理 HTTP 的流量，是 Istio中内容最丰富的一种流量规则。
4. **tls**：是一个 TLSRoute 类型的路由集合，用于处理非终结的 TLS 和 HTTPS 的流量。
5. **tcp**：是一个 TCPRoute 类型的路由集合，用于处理 TCP 的流量，应用于所有其他非 HTTP 和 TLS 端口的流量。如果在 VirtualService 中对 HTTPS 和 TLS 没有定义对应的 TLSRoute，则所有流量都会被当成 TCP 流量来处理，都会走到 TCP 路由集合上。

### 1.  HTTP路由（HTTPRoute）

<div align=center>
<img src="image/HTTPRoute规则.png" style="zoom:20%" />
</div>
<p align="center">图 HTTPRoute规则 （图源 《云原生服务网格Istio》）</p>


















