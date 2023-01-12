---
title: "503 UC upstream_reset_before_response_started{connection_termination}"
linkTitle: "503 UC upstream_reset_before_response_started{connection_termination}"
weight: 13
date: 2023-01-12
description: Upstream 断开链路导致 503 UC。 
---

## 故障现象

客户端直接访问服务器正常，但在 service mesh 中经过 envoy 访问服务器则会出现一定几率的 503 错误。查看客户端侧 envoy 的访问日志，发现日志中有下面的异常信息：

```bash
[2023-01-05T04:21:37.764Z] "POST /foo/bar" 503 UC upstream_reset_before_response_started{connection_termination} - "-" 291 95 0 - "116.211.195.11,116.211.195.11" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/108.0.0.0 Safari/537.36" "06a39679-a8d4-47f7-baf3-d688ea3e67c4" "foo.bar.com" "30.183.173.155:1984" outbound|1984||foor-service.bar-ns.svc.cluster.local 30.169.11.123:46894 30.169.11.123:443 116.211.195.11:21663 - -
```

## 故障原因

从访问日志中 `503 UC upstream_reset_before_response_started{connection_termination}` 的输出，我们可以初步推断出 503 的原因是链接被 upstream 侧中断了。

通过 Envoy 管理端口打开 debug 日志，可以看到在出现 503 UC 时，envoy 从 connection pool 中拿出了一个 upstream 的链接，但拿出该链接后，envoy 打印了一个 "remote close" 日志，说明该链接被对端关闭了。

![](debug.png)

Envoy 的 HTTP Router 会在第一次和 Upstream 建立 TCP 链接并使用后将链接释放到一个链接池中，而不是直接关闭该链接。这样下次 downstream 请求相同的 Upstream host 时可以重用该链接，可以避免频繁创建/关闭链接带来的开销。 

但是如果服务器端关闭了链接，而 Envoy 侧尚未感知到该链接的状态变化，就会导致 Envoy 将该链接从连接池中取出以用于发送来着 downstream 的请求。此时就会出现 `503 UC upstream_reset_before_response_started{connection_termination}` 异常。

## 解决方案

采用 Virtual Service 为出现该问题的服务设置重试策略，在重试策略的 retryOn 中增加 `reset` 条件。

备注：
Istio 缺省为服务设置了重试策略，但缺省的重试策略中并不会对链接重置这种情况进行重试。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-route
spec:
  hosts:
  - ratings.prod.svc.cluster.local
  http:
  - route:
    - destination:
        host: ratings.prod.svc.cluster.local
        subset: v1
    retries:
      attempts: 3
      retryOn: reset,connect-failure,refused-stream,unavailable,cancelled,retriable-status-codes
```

