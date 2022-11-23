---
title: "通过 Ingress Gateway 访问集群外部服务 503 UC 错误"
linkTitle: "通过 Ingress Gateway 访问集群外部服务 503 UC 错误"
weight: 12
date: 2022-11-23
description: 当采用和外部服务的域名不同的 sni 来请求外部 https 服务时，envoy 返回 503 UC 错误。 
---

## 故障现象

该使用场景比较特殊，用户通过 Ingress Gateway 来访问一个集群外部的 HTTPS 服务，Ingress Gateway 返回 503 UC 错误。

用户的访问路径如下：

Browser --> Ingress Gateway（foo.bar.org） --> External Service（dev.bar.org）

Ingress Gateway 配置的 VS 如下：

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: nginx-vs-dev
  namespace: foo-dev
spec:
  gateways:
  - foo-dev/barl-org
  hosts:
  - foo.bar.org
  http:
  - match:
    - uri:
        prefix: /api/test
    route:
    - destination:
        host: dev.bar.org
        port:
          number: 443
```

外部服务对应的 ServiceEntry 定义如下：

```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: dev-test
  namespace: foo-dev
spec:
  hosts:
  - dev.bar.org
  location: MESH_EXTERNAL
  ports:
  - name: https
    number: 443
    protocol: HTTPS
  resolution: DNS
```

Ingress Gateway 中的错误日志如下：

```json
{
   "upstream_cluster":"outbound|443||dev.bar.org",
   "response_flags":"UC",
   "authority":"foo.bar.org",
   "upstream_host":"47.107.45.209:443",
   "bytes_sent":95,
   "downstream_remote_address":"182.140.153.175:2223",
   "downstream_local_address":"192.168.32.49:443",
   "upstream_transport_failure_reason":null,
   "istio_policy_status":null,
   "response_code":503,
   "duration":19,
   "request_id":"054c265b-46eb-4524-a892-810ceeb26e64",
   "path":"/api/test",
   "protocol":"HTTP/2",
   "requested_server_name":"foo.bar.org",
   "upstream_local_address":"192.168.32.49:53382",
   "x_forwarded_for":"182.140.153.175",
   "start_time":"2022-11-23T08:08:52.303Z",
   "upstream_service_time":null,
   "bytes_received":0,
   "user_agent":"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/107.0.0.0 Safari/537.36",
   "method":"GET",
   "route_name":null
}
```

## 故障原因

用户通过 Ingress Gateway 访问时，sni 是 Ingress Gateway 的域名，即 ```foo.bar.org```，Envoy 在和 upstream 进行 tls 握手时，在没有进行配置的情况下，缺省会使用 downstream 的 sni。而该用例中，upstream 的正确 sni 应该是 ```dev.bar.org```。由于 SNI 不匹配，导致 Ingress Gateway 和 外部服务 TLS 握手失败，Ingress Gateway 报 503 UC 错误。

## 解决方案

创建下面的 DR 指定访问该外部服务时使用的 SNI。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: dev-test
spec:
  host: dev.bar.org
  trafficPolicy:
    tls:
      mode: SIMPLE
      sni: dev.bar.org
```