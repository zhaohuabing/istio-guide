
---
title: "无法连接 gateway 上的 tcp 端口"
linkTitle: "无法连接 gateway 上的 tcp 端口"
weight: 10
date: 2022-10-17 
description: 
---

## 故障现象

通过 Gateway CRD 定义了一个 tcp 端口，但是无法连接到 gateway 上该端口。

## 故障原因

如果通过 Gateway 定义了一个 TCP 端口，但没有采用 VS 配置相应的路由，则会出现在 gateway 上找不到该 TCP 端口的情况。

例如，通过下面的 Gateway ，在 ingress gateway 上定义了一个 TCP 端口 8888。

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: ingressgw
spec:
  selector:
    app: istio-ingressgateway
    istio: ingressgateway
  servers:
  - hosts:
    - '*'
    port:
      name: TCP-8888
      number: 8888
      protocol: TCP
```

此时发现通过 ingress gateway 无法访问 8888 端口，查看 ingress gateway 中的 listener 配置，找不到在 8888 上监控的 listener。
```yaml
 -n istio-system proxy-config listeners istio-ingressgateway-74fd488699-4v4rt
ADDRESS PORT  MATCH DESTINATION
0.0.0.0 15021 ALL   Inline Route: /healthz/ready*
0.0.0.0 15090 ALL   Inline Route: /stats/prometheus*
```

此时查看 istiod 的日志，发现有下面的错误输出：

```bash
gateway omitting listener "0.0.0.0_8888" due to: must have more than 0 chains in listener "0.0.0.0_8888"
```

原因是 istiod 在试图生成 listener 时 filter chain 没有内容，导致 istiod 忽略了该 listener。

## 解决方案

采用 VS 为该 port 设置对应的路由，则 istiod 在生成 listener 时 filter chain 就不会为空，可以正常生成 listener。

创建 VS：

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ingress
spec:
  gateways:
  - ingressgw
  hosts:
  - '*'
  tcp:
  - match:
    - port: 8888
    route:
    - destination:
        host: details.default.svc.cluster.local
        port:
          number: 9080
```

此时查看 ingress gateway 的配置，可以看到 8888 对应的 listener 已经成功生成：

```yaml
istioctl -n istio-system proxy-config listeners istio-ingressgateway-74fd488699-4v4rt
ADDRESS PORT  MATCH DESTINATION
0.0.0.0 8888  ALL   Cluster: outbound|9080||details.default.svc.cluster.local
0.0.0.0 15021 ALL   Inline Route: /healthz/ready*
0.0.0.0 15090 ALL   Inline Route: /stats/prometheus*
```

> 备注：HTTP 端口和 TCP 的情况有所不同。如果采用 Gateway 定义了一个 HTTP 端口，没有配置相应的 VS，可以连接到该端口，gateway 将返回 404 HTTP 错误。