Istio路由规则配置：VirtualService
=============

## 一、理解 Istio 的服务模型
Istio 支持将由服务、服务版本和服务实例构造的抽象模型映射到不同的平台上，基于Kubernetes 的场景, Istio 的几个资源对象就是基于 Kubernetes 的相应资源对象构建的，加上部分约束来满足 Istio 服务模型的要求。

* **端口命名**：对 Istio 的服务端口必须进行命名，而且名称只允许是 ＜protocol＞[-＜suffix＞] 这种格式，其中 ＜protocol＞ 可以是 tcp、http、http2、https、grpc、tls、mongo、mysql、redis 等，Istio 根据在端口上定义的协议来提供对应的路由能力。例如“name：http2-forecast”和“name：http”是合法的端口名，如果端口未命名或者没有基于这种格式进行命名，则端口的流量会被当作 TCP 流量来处理。
* **服务关联**：Pod 需要关联到服务，如果一个 Pod 属于多个 Kubernetes 服务，则要求服务不能在同一个端口上使用不同的协议。
* **Deployment 使用 app 和 version 标签**：建议 Kubernetes Deployment 显式地包含 app 和 version 标签。每个 Deployment 都需要有一个有业务意义的 app 标签和一个表示版本的 version 标签。在分布式追踪时可以通过 app 标签来补齐上下文信息，还可以通过 app 和 version 标签为遥测数据补齐上下文信息。


### 1、Istio的服务
从逻辑上看，服务是 Istio 主要管理的资源对象，是一个抽象概念，主要包含 HostName 和 Ports 等属性，并指定了 Service 的域名和端口列表。每个端口都包含端口名称、端口号和端口的协议。

从物理层面看，Istio 服务的存在形式就是 Kubernetes 的 Service，在启用了 Istio 的集群中创建 Kubernetes 的 Service 时只要满足以上约束，就可以转换为 Istio 的 Service 并配置规则进行流量治理。
> Service 是 Kubernetes 的一个核心资源，用户通过一个域名或者虚拟的 IP 就能访问到后端 Pod，避免向用户暴露 Pod 地址的问题，特别是在 Kubernetes 中，Pod 作为一个资源创建、调度和管理的最小部署单元的封装，本来就是动态变化的，在节点删除、资源变化等多种情况下都可能被重新调度，Pod 的后端地址也会随之变化。

**一个Istio Service 示例**

```
apiversion: v1
kind: Service
metadata:
  name: forecast
spec:
  ports:
  - port: 3002
    tagetPort: 3002
    name: http        #Istio 服务的约束，在端口名称上指定协议
  selector:
    app: forecast
```
在 Istio 中，Service 是治理的对象，是 Istio 中的核心管理实体，所以在 Istio 中，Service 是一个提供了对外访问能力的执行体，可以将其理解为一个定义了服务的工作负载，没有访问方式的工作负载不是 Istio 的管理对象，Kubernetes 的 Service 定义就是 Istio 服务的元数据。

### Istio 的服务版本
在 Istio 中多个版本的定义是将一个 Service 关联到多个 Deployment ，每个Deployment 都对应服务的一个版本
> forecast-v1 和 forecast-v2 这两个 Deployment 分别对应服务的两个版本
```
apiVersion: app/v1
kind: Deployment
metadata:
  name: forecast-v1
  labels:
    app: forecast
    version: v1
  spec:
    replicas: 2
    template:
      metadata:
        labels:
          app: forecast
          version: v1
      spec:
        containers:
        - name: forecasts
          image: ***
          ports:
          - containerPort: 3002 
```
```
apiVersion: app/v1
kind: Deployment
metadata:
  name: forecast-v2
  labels:
    app: forecast
    version: v2
  spec:
    replicas: 2
    template:
      metadata:
        labels:
          app: forecast
          version: v2
      spec:
        containers:
        - name: forecast
          image: ***
          ports:
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
Istio 的配置都是通过 Kubernetes 的 CRD（CustomerResourceDefinition 用户自定义资源）方式表达。此示例所配置的规则是：对于 forecast 服务的访问，如果在请求的Header 中 location 取值是 north，则将该请求转发到服务的 v2 版本上，将其他请求都转发到服务的 v1 版本上。

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

#### 1.1 HTTPRoute规则解析
HTTPRoute 规则的功能是：满足 HTTPMatchRequest 条件的流量都被路由到HTTPRouteDestination，执行重定向（HTTPRedirect）、重写（HTTPRewrite）、重试（HTTPRetry）、故障注入（HTTPFaultInjection）、跨站（CorsPolicy）策略等。HTTP 不仅可以做路由匹配，还可以做一些写操作来修改请求本身。
<div align=center>
<img src="image/HTTPRoute规则.png" style="zoom:20%" />
</div>
<p align="center">图 HTTPRoute规则 （图源 《云原生服务网格Istio》）</p>

#### 1.2 HTTP匹配规则（HTTPMatchRequest）
<div align=center>
<img src="image/Authority的标准定义.png"  />
</div>

（1）**uri、scheme、method、authority**：4个字段都是 StringMatch 类型，在匹配请求时都支持 exact、prefix 和 regex 三种模式的匹配，分别表示完全匹配输入的字符串，前缀方式匹配和正则表达式匹配。

（2）**headers**：匹配请求中的 Header，是一个 Map 类型。对于每一个 Header 的值，都可以使用精确、前缀和正则三种方式进行匹配。
```
- match:
  - headers:
      source:
        exact: north
    uri:
      prefix: "/advertisement/"
```
（3）**sourceLabels**：是一个 map 类型的键值对，表示请求来源的负载匹配标签。对于每一个 Header 的值，都可以使用精确、前缀和正则三种方式进行匹配。
```
http:
- match:
  - sourceLabels:
      app: frontend
      version: v2
```
要注意的是，在 VirtualService 中 match 字段都是数组类型。HTTPMatchRequest 中的诸多属性如 uri、headers、method 等是“与”逻辑，而数组中几个元素间的关系是“或”逻辑。

在下面的示例中，match 包含两个 HTTPMatchRequest 元素，其条件的语义是：headers中的 source 取值为 “north”，并且 uri 以“/advertisement”开头的请求，或者 uri 以“/forecast”开头的请求。
```
- match:
  - headers:
      source:
        exact: north
    uri:
      prefix: "/advertisement/"
  - uri:
      prefix: "/forecast/"
```
#### 1.3 HTTP路由目标（HTTPRouteDestination）
在 HTTPRouteDestination 中主要有三个字段：**destination（请求目标）、weight（权重）和headers（HTTP头操作）**，destination 和 weight 是必选字段。

>如果一个route只有一个destination，那么可以不用配置weight，默认就是100。如下示例为将全部流量都转到这一个destination上：
```
……
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
>  从原有的v1版本中切分20%的流量到v2版本(也是灰度发布常用的一种流量策略)，即不区分内容，平等地从总流量中切出一部分流量给新版本：
```
……
spec:
  hosts:
  - forecast
  http:
  - route:
    - destination:
        host: forecast
        subset: v2
      weight: 20
    - destination:
        host: forecast
        subset: v1
      weight: 80  
```
#### 1.4 HTTP重定向（HTTPRedirect）
```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: forecast
  namespace: weather
spec:
  hosts:
  - forecast
  http:
  - match:
    - uri:
       prefix: /advertisement
    redirect:
      uri: /recommendation/activity
      authority: new-forecast
```
#### 1.5 HTTP重写（HTTPRewrite）
```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: forecast
  namespace: weather
spec:
  hosts:
  - forecast
  http:
  - match:
    - uri:
       prefix: /advertisement
    rewrite:
      uri: /recommendation/activity
    route:
    - destination:
        host: forecast
```
#### 1.6 HTTP重试（HTTPRetry）
HTTPRetry可以定义请求失败时的重试策略。重试策略包括重试次数、超时、重试条件等，这里分别描述相应的三个字段。
* **attempts**：必选字段，定义重试的次数。
* **perTryTimeout**：每次重试的超时时间，单位可以是毫秒（ms）、秒（s）、分钟（m）和小时（h）。
* **retryOn**：进行重试的条件，可以是多个条件，以逗号分隔。其中，重试条件retryOn的取值包括以下几种。
* **5xx**：在上游服务返回5xx应答码，或者在没有返回时重试。
* **gateway-error**：类似5xx异常，只对502、503和504应答码进行重试。
* **connect-failure**：在连接上游服务失败时重试。
```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: forecast
  namespace: weather
spec:
  hosts:
  - forecast
  http:
  - route:
    - destination:
        host: forecast
    retries:
      attempts: 5
      perTryTimeout: 3s
      retryOn: 5xx,connect-failure
```

#### 1.7 HTTP流量镜像（Mirror）
> 如下示例：把到forcast v1 版本的流量镜像到 v2版本上
```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: forecast
  namespace: weather
spec:
  hosts:
  - forecast
  http:
  - route:
    - destination:
        host: forecast
        subset: v1
    mirror:
      host: forecast
      subset: v2
```

#### 1.8 HTTP故障注入（HTTPFaultInjection）
##### 1. 延迟故障注入
HTTPFaultInjection 中的延迟故障使用 HTTPFaultInjection.Delay 类型描述延时故障，表示在发送请求前进行一段延时，模拟网络、远端服务负载等各种原因导致的失败，主要有如下两个字段。
* **fixedDelay**：一个必选字段，表示延迟时间，单位可以是毫秒、秒、分钟和小时，要求时间必须大于1毫秒。
* **percentage**：配置的延迟故障作用在多少比例的请求上，通过这种方式可以只让部分请求发生故障。
> 如下例所示将 forecast 服务 v1 版本上百分之10的请求产生5秒的延迟
```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: forecast
  namespace: weather
spec:
  hosts:
  - forecast
  http:
  - route:
    - destination:
        host: forecast
        subset: v1
    fault:
      delay: 
        percentage:
          value: 10
        fixedDelay: 5s
```
##### 2. 请求中止故障注入
HTTPFaultInjection 使用 HTTPFaultInjection.Abort 描述中止故障，模拟服务端异常，给调用的客户端返回预先定义的错误状态码，主要有如下两个字段。
* **httpStatus**：是一个必选字段，表示中止的HTTP 状态码。
* **percentage**：配置的中止故障作用在多少比例的请求上，通过这种方式可以只让部分请求发生故障，用法同延迟故障。
> 如下例所示将 forecast 服务 v1 版本上百分之10的请求返回500错误
```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: forecast
  namespace: weather
spec:
  hosts:
  - forecast
  http:
  - route:
    - destination:
        host: forecast
        subset: v1
    fault:
      delay: 
        percentage:
          value: 10
        httpStatus: 500
```

##### 1.9 HTTP跨域资源共享（CorsPolicy）
……

















