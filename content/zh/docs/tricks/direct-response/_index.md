---
title: "directResoponse"
weight: 1
date: 2022-10-17
description: >
  代理对符合某个条件的 HTTP 请求直接返回一个响应。
---

有时候我们希望 gateway/sidecar 能直接向客户端返回一个 HTTP response，而不用交给应用程序处理。

## 1.15 版本之前

可以通过 EnvoyFilter 来修改 HTTP Route，为匹配某个条件的 HTTP 请求直接返回指定的内容。例如，下面的 EnvoyFilter 为 ingress gateway 收到的 http://*:80/direct 直接返回一个 200 response，消息体为 ```hello world```。：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: direct
spec:
  workloadSelector:
    labels:
      istio: ingressgateway
  configPatches:
  - applyTo: HTTP_ROUTE
    match:
      context: GATEWAY
      routeConfiguration:
        portNumber: 80
    patch:
      operation: INSERT_FIRST
      value:
        name: direct
        match:
          path: /direct
        directResponse:
          body:
            inlineString: 'hello world'
          status: 200
```

## 1.15 及之后版本

1.15 版本开始，VS 支持设置 directResponse，如下所示：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-route
spec:
  hosts:
  - ratings.prod.svc.cluster.local
  http:
  - match:
    - uri:
        exact: /v1/getProductRatings
    directResponse:
      status: 503
      body:
        string: "unknown error"
  ...
```

## 参考文档

* EnvoyFilter https://istio.io/latest/docs/reference/config/networking/envoy-filter/
* HTTPDirectResponse https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPDirectResponse
