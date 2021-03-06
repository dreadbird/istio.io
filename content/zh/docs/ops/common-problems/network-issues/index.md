---
title: 网络问题
description: 定位 Istio 流量管理和网络问题的技术。
force_inline_toc: true
weight: 10
aliases:
  - /zh/help/ops/traffic-management/troubleshooting
  - /zh/help/ops/troubleshooting/network-issues
  - /zh/docs/ops/troubleshooting/network-issues
---

## 请求被 Envoy 拒绝{#requests-are-rejected-by-envoy}

请求被拒绝有许多原因。弄明白为什么请求被拒绝的最好方式是通过检查 Envoy 的访问日志。默认情况下，访问日志被输出到容器的标准输出中。运行下列命令可以查看日志：

{{< text bash >}}
$ kubectl logs PODNAME -c istio-proxy -n NAMESPACE
{{< /text >}}

在默认的访问日志输出格式中，Envoy 响应标志和 Mixer 策略状态位于响应状态码之后，如果你使用自定义日志输出格式，请确保包含 `%RESPONSE_FLAGS%` 和 `%DYNAMIC_METADATA(istio.mixer:status)%`。

参考 [Envoy 响应标志](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log#config-access-log-format-response-flags)查看更多有关响应标志的细节。

通用响应标志如下：

- `NR`：没有配置路由，请检查你的 `DestinationRule` 或者 `VirtualService` 配置。
- `UO`：上游溢出导致断路，请在 `DestinationRule` 检查你的熔断器配置。
- `UF`：未能连接到上游，如果你正在使用 Istio 认证，请检查 [mutual-tls 配置冲突](#service-unavailable-errors-after-setting-destination-rule)。

如果一个请求的响应标志是 `UAEX` 并且 Mixer 策略状态不是 `-`，表示这个请求被 Mixer 拒绝。

通用 Mixer 策略状态如下：

- `UNAVAILABLE`：Envoy 不能连接到 Mixer 并且策略被配置为失败自动关闭。
- `UNAUTHENTICATED`：请求被 Mixer 认证组件拒绝。
- `PERMISSION_DENIED`：请求被 Mixer 认证组件拒绝。
- `RESOURCE_EXHAUSTED`：请求被 Mixer 指标组件拒绝。
- `INTERNAL`：因为 Mixer 内部错误请求被拒绝。

## 路由规则似乎没有对流量生效{#route-rules-don't-seem-to-affect-traffic-flow}

在当前版本的 Envoy sidecar 实现中，加权版本分发被观测到至少需要 100 个请求。

如果路由规则在 [Bookinfo](/zh/docs/examples/bookinfo/) 这个例子中完美地运行，但在你自己的应用中相似版本的路由规则却没有生效，可能因为你的 Kubernetes service 需要被稍微地修改。为了利用 Istio 的七层路由特性 Kubernetes service 必须严格遵守某些限制。参考 [Pods 和 Services 的要求](/zh/docs/ops/prep/requirements/)查看详细信息。

另一个潜在的问题是路由规则可能只是生效比较慢。在 Kubernetes 上实现的 Istio 利用一个最终一致性算法来保证所有的 Envoy sidecar 有正确的配置包括所有的路由规则。一个配置变更需要花一些时间来传播到所有的 sidecar。在大型的集群部署中传播将会耗时更长并且可能有几秒钟的延迟时间。

## Destination rule 策略未生效{#destination-rule-policy-not-activated}

尽管 destination rule 和特定的目标主机关联，subset-specific 策略的激活最终依赖于路由规则。

当路由一个请求，Envoy 首先会评估 virtual service 中的 route rule 来决定是否路由到一个特定的 subset。如此，它才会激活任意与 subset 相对应的 destination rule 策略。所以，如果你希望流量路由到正确的 subset，只有你定义明确的 subset 策略 Istio 才会应用。

举个例子，假设下面是为 *reviews* 服务定义的唯一的 destination rule，因此，这里没有与 `VirtualService` 定义相对应的路由规则：

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
    trafficPolicy:
      connectionPool:
        tcp:
          maxConnections: 100
{{< /text >}}

如果某些情况下 Istio 默认采用 round-robin 策略来访问 "v1" 实例，可能最终 "v1" 实例是唯一运行的版本，那么上面的流量策略将永远不会被使用到。

你可以通过下面两种方式之一来使上面的例子生效：

1. 将 destination rule 中的流量策略上移一级以使策略应用到任意 subset，例如：

    {{< text yaml >}}
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: reviews
    spec:
      host: reviews
      trafficPolicy:
        connectionPool:
          tcp:
            maxConnections: 100
      subsets:
      - name: v1
        labels:
          version: v1
    {{< /text >}}

1. 用 `VirtualService` 为这个服务定义一个正确的路由规则。例如，为 `reviews` 服务的 `v1` subset 添加一个简单的路由规则：

    {{< text yaml >}}
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: reviews
    spec:
      hosts:
      - reviews
      http:
      - route:
        - destination:
            host: reviews
            subset: v1
    {{< /text >}}

Istio 默认可以方便地将流量从任意源传输到目标服务的任意版本而不需要设置任何规则。因为你需要区分一个服务的不同版本，所以你需要定义不同的路由规则。因此，我们考虑从一开始就为每个服务基于最佳实践设置一个默认的路由规则。

## 设置 destination rule 之后出现 503 异常{#service-unavailable-errors-after-setting-destination-rule}

如果在你应用了一个 `DestinationRule` 之后请求一个服务立即产生了一个 503 异常，并且这个异常持续产生直到你移除了这个 `DestinationRule`，那么这个 `DestinationRule` 大概为这个服务引起了一个 TLS 冲突。

举个例子，如果在你的集群里配置了全局的 mutual TLS，这个 `DestinationRule` 肯定包含下列的 `trafficPolicy`：

{{< text yaml >}}
trafficPolicy:
  tls:
    mode: ISTIO_MUTUAL
{{< /text >}}

否则，这个 TLS mode 默认被设置成 `DISABLE` 会使客户端 sidecar 代理发起普通的 HTTP 请求而不是 TLS 加密了的请求。因此，请求和服务端代理冲突，因为服务端代理期望的是加密了的请求。

为了确认是否存在冲突，请检查你的服务 [`istioctl authn tls-check`](/zh/docs/reference/commands/istioctl/#istioctl-authn-tls-check) 命令输出中的 `STATUS` 字段是否被设置为 `CONFLICT`。举个例子，一个和如下命令类似的命令可以用来为 `httpbin` 服务检查冲突：

{{< text bash >}}
$ istioctl authn tls-check istio-ingressgateway-db454d49b-lmtg8.istio-system httpbin.default.svc.cluster.local
HOST:PORT                                  STATUS       SERVER     CLIENT     AUTHN POLICY     DESTINATION RULE
httpbin.default.svc.cluster.local:8000     CONFLICT     mTLS       HTTP       default/         httpbin/default
{{< /text >}}

任何时候你应用一个 `DestinationRule`，请确保 `trafficPolicy` TLS mode 和全局的配置一致。

## 路由规则没有对 ingress gateway 请求生效{#route-rules-have-no-effect-on-ingress-gateway-requests}

让我们假设你正在使用一个 ingress `Gateway` 和相应的 `VirtualService` 来访问一个内部的服务。举个例子，你的 `VirtualService` 配置可能和如下配置类似：

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - "myapp.com" # 或者，你正在测试 ingress-gateway IP 而不是 DNS 也可以配置成“*”（例如，http://1.2.3.4/hello）
  gateways:
  - myapp-gateway
  http:
  - match:
    - uri:
        prefix: /hello
    route:
    - destination:
        host: helloworld.default.svc.cluster.local
  - match:
    ...
{{< /text >}}

并且你有一个路由 helloworld 服务到一个普通 subset 的 `VirtualService` 配置：

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: helloworld
spec:
  hosts:
  - helloworld.default.svc.cluster.local
  http:
  - route:
    - destination:
        host: helloworld.default.svc.cluster.local
        subset: v1
{{< /text >}}

在这种情况下你会注意到向 helloworld 服务发起的请求通过 ingress gateway 将会被直接路由到 subset v1 而不是使用默认的 round-robin 路由规则。

gateway 主机（例如，`myapp.com`）配置将会激活 myapp `VirtualService` 里的规则使 ingress 请求被路由到 helloworld 服务的任意端点。只有服务 `helloworld.default.svc.cluster.local` 内部的请求将会使用 helloworld `VirtualService` 配置直接将流量定向传输到 subset v1。

为了控制从 gateway 过来的流量，你需要在 myapp `VirtualService` 的配置中包含 subset 规则配置：

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - "myapp.com" # 或者，你正在测试 ingress-gateway IP 而不是 DNS 也可以配置成”*“（例如，http://1.2.3.4/hello）
  gateways:
  - myapp-gateway
  http:
  - match:
    - uri:
        prefix: /hello
    route:
    - destination:
        host: helloworld.default.svc.cluster.local
        subset: v1
  - match:
    ...
{{< /text >}}

或者，你可以尽可能地将连个 `VirtualServices` 配置合并成一个：

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp.com # 这里不能使用“*”，因为这是与网格服务结合在一起的。
  - helloworld.default.svc.cluster.local
  gateways:
  - mesh # 内部和外部都可以应用
  - myapp-gateway
  http:
  - match:
    - uri:
        prefix: /hello
      gateways:
      - myapp-gateway # 只对 ingress gateway 严格应用这条规则
    route:
    - destination:
        host: helloworld.default.svc.cluster.local
        subset: v1
  - match:
    - gateways:
      - mesh # 应用到网格中的所有服务
    route:
    - destination:
        host: helloworld.default.svc.cluster.local
        subset: v1
{{< /text >}}

## Headless TCP 服务失去连接{#headless-tcp-services-losing-connection}

如果部署了 `istio-citadel`，Envoy 每 45 天会重启一次来使证书刷新。这会导致 TCP 数据流失去连接或者服务之间的长连接。

你应该在你的应用中为这种失去连接异常构建快速恢复的能力，但你仍然想阻止这种失去连接异常的发生，你应该禁用 mutual TLS 配置并且禁止 `istio-citadel` 部署。

首先，编辑你的 `istio` 配置来禁用 mutual TLS：

{{< text bash >}}
$ kubectl edit configmap -n istio-system istio
$ kubectl delete pods -n istio-system -l istio=pilot
{{< /text >}}

然后，逐一下线 `istio-citadel` 部署来禁止 Envoy 重启：

{{< text bash >}}
$ kubectl scale --replicas=0 deploy/istio-citadel -n istio-system
{{< /text >}}

这将会使 Istio 停止重启 Envoy 并且不再产生失去 TCP 连接的异常。

## Envoy 在负载下崩溃{#envoy-is-crashing-under-load}

检查你的 `ulimit -a`。许多系统有一个默认只能有打开 1024 个文件的 descriptor 的限制，它将导致 Envoy 断言失败并崩溃：

{{< text plain >}}
[2017-05-17 03:00:52.735][14236][critical][assert] assert failure: fd_ != -1: external/envoy/source/common/network/connection_impl.cc:58
{{< /text >}}

请确保增大你的 ulimit。例如: `ulimit -n 16384`

## Envoy 不能连接到 HTTP/1.0 服务{#envoy-won't-connect-to-my-http/1.0-service}

Envoy 要求 `HTTP/1.1` 或者 `HTTP/2` 协议的流量作为上游服务。举个例子，当在 Envoy 之后使用 [NGINX](https://www.nginx.com/) 来代理你的流量，你将需要在你的 NGINX 配置里直接设置 [proxy_http_version](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_http_version) 为 "1.1"，因为 NGINX 默认的设置是 1.0。

例如配置为：

{{< text plain >}}
upstream http_backend {
    server 127.0.0.1:8080;

    keepalive 16;
}

server {
    ...

    location /http/ {
        proxy_pass http://http_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        ...
    }
}
{{< /text >}}

## 当为多个 gateway 配置了相同的 TLS 证书导致 404 异常{#not-found-errors-occur-when-multiple-gateways-configured-with-same-TLS-certificate}

多个网关配置同一 TLS 证书会导致浏览器在与第一台主机建立连接之后访问第二台主机时利用 [HTTP/2 连接复用](https://httpwg.org/specs/rfc7540.html#reuse)（例如，大部分浏览器）从而导致 404 异常产生。

举个例子，假如你有 2 个主机共用相同的 TLS 证书，如下所示：

- 通配证书 `*.test.com` 被安装到 `istio-ingressgateway`
- `Gateway` 配置 `gw1` 到主机 `service1.test.com`，选择器 `istio: ingressgateway`，并且 TLS 使用 gateway 安装的（通配）证书
- `Gateway` 配置 `gw2` 到主机 `service2.test.com`，选择器 `istio: ingressgateway`，并且 TLS 使用 gateway 安装的（通配）证书
- `VirtualService` 配置 `vs1` 到主机 `service1.test.com` 并且 gateway 配置为 `gw1`
- `VirtualService` 配置 `vs2` 到主机 `service2.test.com` 并且 gateway 配置为 `gw2`

因为两个网关都由相同的工作负载提供服务（例如，选择器 `istio: ingressgateway`）到两个服务的请求（`service1.test.com` 和 `service2.test.com`）将会解析为同一 IP。如果 `service1.test.com` 首先被接受了，它将会返回一个通配证书（`*.test.com`）表明到 `service2.test.com` 的连接也可以使用相同的证书。浏览器比如 Chrome 和 Firefox 因此会自动使用已建立的连接来发送到 `service2.test.com` 的请求。因此 gateway（`gw1`）没有到 `service2.test.com` 的路由信息，它会返回一个 404 (Not Found) 响应。

你可以通过配置一个单独的 `Gateway` 来阻止这个问题产生，而不是两个（`gw1` 和 `gw2`）。然后，简单地绑定两个 `VirtualServices` 到这个单独的网关，比如这样：

- `Gateway` 配置 `gw` 到主机 `*.test.com`，选择器 `istio: ingressgateway`，并且 TLS 使用 gateway 安装的（通配）证书
- `VirtualService` 配置 `vs1` 到主机 `service1.test.com` 并且 gateway 配置为 `gw`
- `VirtualService` 配置 `vs2` 到主机 `service2.test.com` 并且 gateway 配置为 `gw`
