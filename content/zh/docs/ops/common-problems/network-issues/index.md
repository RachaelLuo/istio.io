---
title: 流量管理问题
description: 定位常见的 Istio 流量管理和网络问题的技术。
force_inline_toc: true
weight: 10
aliases:
  - /zh/help/ops/traffic-management/troubleshooting
  - /zh/help/ops/troubleshooting/network-issues
  - /zh/docs/ops/troubleshooting/network-issues
owner: istio/wg-networking-maintainers
test: no
---

## 请求被 Envoy 拒绝{#requests-are-rejected-by-envoy}

请求被拒绝有许多原因。弄明白为什么请求被拒绝的最好方式是检查 Envoy 的访问日志。默认情况下，访问日志被输出到容器的标准输出中。运行下列命令可以查看日志：

{{< text bash >}}
$ kubectl logs PODNAME -c istio-proxy -n NAMESPACE
{{< /text >}}

在默认的访问日志输出格式中，Envoy 响应标志位于响应状态码之后，如果您使用自定义日志输出格式，请确保包含 `%RESPONSE_FLAGS%`。

参考 [Envoy 响应标志](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#config-access-log-format-response-flags)查看更多有关响应标志的细节。

通用响应标志如下：

- `NR`：没有配置路由，请检查您的 `DestinationRule` 或者 `VirtualService` 配置。
- `UO`：上游溢出导致断路，请在 `DestinationRule` 检查您的熔断器配置。
- `UF`：未能连接到上游，如果您正在使用 Istio 认证，请检查[双向 TLS 配置冲突](#service-unavailable-errors-after-setting-destination-rule)。

## 路由规则似乎没有对流量生效{#route-rules-don't-seem-to-affect-traffic-flow}

在当前版本的 Envoy Sidecar 实现中，加权版本分发被观测到至少需要 100 个请求。

如果路由规则在 [Bookinfo](/zh/docs/examples/bookinfo/) 这个例子中完美地运行，但在您自己的应用中相似版本的路由规则却没有生效，可能因为您的 Kubernetes service 需要被稍微地修改。为了利用 Istio 的七层路由特性 Kubernetes service 必须严格遵守某些限制。参考 [Pods 和 Services 的要求](/zh/docs/ops/deployment/requirements/)查看详细信息。

另一个潜在的问题是路由规则可能只是生效比较慢。在 Kubernetes 上实现的 Istio 利用一个最终一致性算法来保证所有的 Envoy Sidecar 有正确的配置包括所有的路由规则。一个配置变更需要花一些时间来传播到所有的 sidecar。在大型的集群部署中传播将会耗时更长并且可能有几秒钟的延迟时间。

## 设置 destination rule 之后出现 503 异常{#service-unavailable-errors-after-setting-destination-rule}

{{< tip >}}
只有在安装期间禁用了 [自动双向 TLS](/zh/docs/tasks/security/authentication/authn-policy/#auto-mutual-TLS) 时，才会看到此错误。
{{< /tip >}}

如果在您应用了一个 `DestinationRule` 之后请求一个服务立即发生了 HTTP 503 异常，并且这个异常状态一直持续到您移除或回滚了这个 `DestinationRule`，那么这个 `DestinationRule` 大概为这个服务引起了一个 TLS 冲突。

举个例子，如果在您的集群里配置了全局的 mutual TLS，这个 `DestinationRule` 肯定包含下列的 `trafficPolicy`：

{{< text yaml >}}
trafficPolicy:
  tls:
    mode: ISTIO_MUTUAL
{{< /text >}}

否则，这个 TLS mode 默认被设置成 `DISABLE` 会使客户端 sidecar 代理发起明文 HTTP 请求而不是 TLS 加密了的请求。因此，请求和服务端代理冲突，因为服务端代理期望的是加密了的请求。

任何时候您应用一个 `DestinationRule`，请确保 `trafficPolicy` TLS mode 和全局的配置一致。

## 路由规则没有对 ingress gateway 请求生效{#route-rules-have-no-effect-on-ingress-gateway-requests}

假设您正在使用一个 Ingress `Gateway` 和相应的 `VirtualService` 来访问一个内部的服务。举个例子，您的 `VirtualService` 配置可能和如下配置类似：

{{< text yaml >}}
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - "myapp.com" # 或者您正在通过 IP 而不是 DNS 测试 ingress-gateway（例如 http://1.2.3.4/hello），也可以配置成 "*"
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

您还有一个 `VirtualService` 将访问 helloworld 服务的流量路由至该服务的一个特定子集：

{{< text yaml >}}
apiVersion: networking.istio.io/v1beta1
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

此时您会发现，通过 ingress 网关访问 helloworld 服务的请求没有直接路由到服务实例子集 v1，而是仍然使用默认的轮询调度路由。

Ingress 请求经由网关主机（如：`myapp.com`）进行路由，网关主机将激活 myapp `VirtualService` 中的规则，将请求路由至 helloworld 服务的任何一个实例端点。
只有通过主机 `helloworld.default.svc.cluster.local` 访问的内部请求才会使用 helloworld `VirtualService`，其中的规则直接将流量路由至服务实例子集 v1。

为了控制从 gateway 过来的流量，您需要在 myapp `VirtualService` 的配置中包含 subset 规则配置：

{{< text yaml >}}
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - "myapp.com" # 或者您正在通过 IP 而不是 DNS 测试 ingress-gateway（例如 http://1.2.3.4/hello），也可以配置成 "*"
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

或者，您可以尽可能地将两个 `VirtualService` 配置合并成一个：

{{< text yaml >}}
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp.com # 这里不能使用“*”，因为这是与网格服务关联在一起的。
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

## Envoy 在负载下崩溃{#envoy-is-crashing-under-load}

检查您的 `ulimit -a`。许多系统有一个默认只能有打开 1024 个文件描述符的限制，它将导致 Envoy 断言失败并崩溃：

{{< text plain >}}
[2017-05-17 03:00:52.735][14236][critical][assert] assert failure: fd_ != -1: external/envoy/source/common/network/connection_impl.cc:58
{{< /text >}}

请确保增大您的 ulimit。例如: `ulimit -n 16384`

## Envoy 不能连接到 HTTP/1.0 服务{#envoy-won't-connect-to-my-http/1.0-service}

Envoy 要求上游服务使用 `HTTP/1.1` 或者 `HTTP/2` 协议流量。举个例子，当在 Envoy 之后使用 [NGINX](https://www.nginx.com/) 来代理您的流量，您将需要在您的 NGINX 配置里将 [proxy_http_version](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_http_version) 设置为 "1.1"，因为 NGINX 默认的设置是 1.0。

示例配置为：

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

## TLS 配置错误{#TLS-configuration-mistakes}

许多流量管理问题是由于错误的 [TLS 配置](/zh/docs/ops/configuration/traffic-management/tls-configuration/) 而导致的。
以下各节描述了一些最常见的错误配置。

### 将 HTTPS 流量发送到 HTTP 端口{#sending-HTTPS-to-an-HTTP-port}

如果您的应用程序向声明为 HTTP 的服务发送 HTTPS 请求，Envoy Sidecar 将在转发请求时尝试将请求解析为 HTTP，
这会使 HTTP 被意外加密，从而导致失败。

{{< text yaml >}}
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: httpbin
spec:
  hosts:
  - httpbin.org
  ports:
  - number: 443
    name: http
    protocol: HTTP
  resolution: DNS
{{< /text >}}

虽然如果您有意在端口 443 上发送明文（如，`curl http://httpbin.org:443`），上述配置可能是正确的，但是一般情况下，443 端口专用于 HTTPS 流量。

发送像 `curl https://httpbin.org` 这样的 HTTPS 请求（默认端口为443）将导致类似于 `curl: (35) error:1408F10B:SSL routines:ssl3_get_record:wrong version number` 的错误。
访问日志也可能显示如 `400 DPE` 的错误。

要解决这个问题，您应该将端口协议改为 HTTPS:

{{< text yaml >}}
spec:
  ports:
  - number: 443
    name: https
    protocol: HTTPS
{{< /text >}}

### 网关到 `VirtualService` 的 TLS 不匹配{#gateway-mismatch}

将 `VirtualService` 绑定到网关时，可能会发生两种常见的 TLS 不匹配。

1. 网关终止了 TLS，而 `VirtualService` 配置 TLS 路由。
1. 网关启用 TLS 透传，而 `VirtualService` 配置了 HTTP 路由。

#### 网关和 TLS termination{#gateway-with-TLS-termination}

{{< text yaml >}}
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
      - "*"
    tls:
      mode: SIMPLE
      credentialName: sds-credential
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "*.example.com"
  gateways:
  - istio-system/gateway
  tls:
  - match:
    - sniHosts:
      - "*.example.com"
    route:
    - destination:
        host: httpbin.org
{{< /text >}}

在此示例中，当 `VirtualService` 使用基于 TLS 的路由时，网关将终止TLS。
因为在计算路由规则时 TLS 已经终止，所以 TLS 路由规则将无效。

使用这种错误配置，您将最终获得404响应，因为请求将发送到 HTTP 路由，但未配置 HTTP 路由。
您可以使用 `istioctl proxy-config routes` 命令确认这一点。

要解决这个问题，您应该切换 `VirtualService` 来指定 `http` 路由，而不是 `tls`:

{{< text yaml >}}
spec:
  ...
  http:
  - match: ...
{{< /text >}}

#### 网关和 TLS passthrough{#gateway-with-TLS-passthrough}

{{< text yaml >}}
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - "*"
    port:
      name: https
      number: 443
      protocol: HTTPS
    tls:
      mode: PASSTHROUGH
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: virtual-service
spec:
  gateways:
  - gateway
  hosts:
  - httpbin.example.com
  http:
  - route:
    - destination:
        host: httpbin.org
{{< /text >}}

在此配置中，`VirtualService` 试图将 HTTP 流量与通过网关的 TLS 流量进行匹配。
这将导致 `VirtualService` 配置无效。您可以使用 `istioctl proxy-config listener` 和 `istioctl proxy-config route` 命令观察到未应用 HTTP 路由。

要解决这个问题，您应该切换 `VirtualService` 以配置 TLS 路由。

{{< text yaml >}}
spec:
  tls:
  - match:
    - sniHosts: ["httpbin.example.com"]
    route:
    - destination:
        host: httpbin.org
{{< /text >}}

另外，您可以通过在网关中切换 `tls` 配置来终止 TLS，而不是通过它：

{{< text yaml >}}
spec:
  ...
    tls:
      credentialName: sds-credential
      mode: SIMPLE
{{< /text >}}

### 双 TLS（TLS 源发起 TLS 连接）{#double-tls}

将 Istio 配置为执行 {{< gloss >}}TLS origination{{< /gloss >}} 时，您需要确保应用程序将纯文本请求发送到 sidecar，sidecar 将随后发起 TLS。

下述 `DestinationRule` 向 `httpbin.org` 服务发起 TLS 连接，但相应的 `ServiceEntry` 在端口 443 上将协议定义为 HTTPS。

{{< text yaml >}}
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: httpbin
spec:
  hosts:
  - httpbin.org
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: originate-tls
spec:
  host: httpbin.org
  trafficPolicy:
    tls:
      mode: SIMPLE
{{< /text >}}

使用此配置，Sidecar 期望应用程序在端口443上发送TLS通信 (如，`curl https://httpbin.org`)，但它也将在转发请求之前发起 TLS 连接。
这将导致对请求进行双重加密。

例如，发送 `curl https://httpbin.org` 之类的请求将导致错误：
`(35) error:1408F10B:SSL routines:ssl3_get_record:wrong version number`。

您可以通过将 `ServiceEntry` 中的端口协议更改为 HTTP 来解决此示例：

{{< text yaml >}}
spec:
  hosts:
  - httpbin.org
  ports:
  - number: 443
    name: http
    protocol: HTTP
{{< /text >}}

请注意，使用此配置，您的应用程序将需要向端口 443 发送纯文本请求，例如 `curl http://httpbin.org:443`，因为 TLS 连接不会更改端口。
但是，从 Istio 1.8 开始，您可以将 HTTP 端口 80 暴露给应用程序（例如，`curl http://httpbin.org`），然后将请求重定向到 `targetPort` 443 以用于发起 TLS：

{{< text yaml >}}
spec:
  hosts:
  - httpbin.org
  ports:
  - number: 80
    name: http
    protocol: HTTP
    targetPort: 443
{{< /text >}}

### 当为多个 Gateway 配置了相同的 TLS 证书导致 404 异常{#not-found-errors-occur-when-multiple-gateways-configured-with-same-TLS-certificate}

多个网关配置同一 TLS 证书会导致浏览器在与第一台主机建立连接之后访问第二台主机时利用 [HTTP/2 连接复用](https://httpwg.org/specs/rfc7540.html#reuse)（例如，大部分浏览器）从而导致 404 异常产生。

举个例子，假如您有 2 个主机共用相同的 TLS 证书，如下所示：

- 通配证书 `*.test.com` 被安装到 `istio-ingressgateway`
- `Gateway` 将 `gw1` 配置为主机 `service1.test.com`，选择器 `istio: ingressgateway`，并且 TLS 使用 gateway 安装的（通配）证书
- `Gateway` 将 `gw2` 配置为主机 `service2.test.com`，选择器 `istio: ingressgateway`，并且 TLS 使用 gateway 安装的（通配）证书
- `VirtualService` 将 `vs1` 配置为主机 `service1.test.com` 并且 gateway 配置为 `gw1`
- `VirtualService` 将 `vs2` 配置为主机 `service2.test.com` 并且 gateway 配置为 `gw2`

因为两个网关都由相同的工作负载提供服务（例如，选择器 `istio: ingressgateway`），到两个服务的请求（`service1.test.com` 和 `service2.test.com`）将会解析为同一 IP。
如果 `service1.test.com` 首先被接受了，它将会返回一个通配证书（`*.test.com`）使得到 `service2.test.com` 的连接也能够使用相同的证书。
因此，Chrome 和 Firefox 等浏览器会自动使用已建立的连接来发送到 `service2.test.com` 的请求。
因为 gateway（`gw1`）没有到 `service2.test.com` 的路由信息，它会返回一个 404 (Not Found) 响应。

您可以通过配置一个单独的通用 `Gateway` 来避免这个问题，而不是两个（`gw1` 和 `gw2`）。
然后，简单地绑定两个 `VirtualService` 到这个单独的网关，比如这样：

- `Gateway` 将 `gw` 配置为主机 `*.test.com`，选择器 `istio: ingressgateway`，并且 TLS 使用网关挂载的（通配）证书
- `VirtualService` 将 `vs1` 配置为主机 `service1.test.com` 并且 gateway 配置为 `gw`
- `VirtualService` 将 `vs2` 配置为主机 `service2.test.com` 并且 gateway 配置为 `gw`

### 不发送 SNI 时配置 SNI 路由{#configuring-SNI-routing-when-not-sending-SNI}

指定 `hosts` 字段的 HTTPS `Gateway` 将对传入请求执行 [SNI](https://en.wikipedia.org/wiki/Server_Name_Indication) 匹配。
例如，以下配置仅允许在 SNI 中匹配 `*.example.com` 的请求：

{{< text yaml >}}
servers:
- port:
    number: 443
    name: https
    protocol: HTTPS
  hosts:
  - "*.example.com"
{{< /text >}}

这可能会导致某些请求失败。

例如，如果您没有设置 DNS，而是直接设置主机标头，例如 `curl 1.2.3.4 -H "Host: app.example.com"`，则 SNI 不会被设置，从而导致请求失败。
相反，您可以设置 DNS 或使用 `curl` 的 `--resolve` 标志。 有关更多信息，请参见[安全网关](/zh/docs/tasks/traffic-management/ingress/secure-ingress/)。

另一个常见的问题是 Istio 前面的负载均衡器。
大多数云负载均衡器不会转发 SNI，因此，如果您要终止云负载均衡器中的 TLS，则可能需要执行以下操作之一：

- 将云负载均衡器改为 TLS 连接方式
- 通过将 hosts 字段设置为 `*` 来禁用 `Gateway` 中的 SNI 匹配

常见的症状是负载均衡器运行状况检查成功，而实际流量失败。
