---
title: "为 TCP 长链接周期输出访问日志"
weight: 3
date: 2023-05-16
description: >
  为 TCP 长链接周期输出访问日志。
---

缺省情况下，TCP 的访问日志只会在链接结束后再输出，对于长链接来说，会在链接建立后很长时间都无法看到访问日志。我们可以通过下面的 EnvoyFilter 来实现周期性的输出访问日志。

```yaml

apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: periodical-access-log
  namespace: istio-system # apply to all sidecars
spec:
  configPatches:
  - applyTo: NETWORK_FILTER
    match:
      listener:
        filterChain:
          filter:
            name: "envoy.filters.network.tcp_proxy"
    patch:
      operation: MERGE
      value:
        name: "envoy.filters.network.tcp_proxy"
        typed_config:
          "@type": "type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy"
          access_log_flush_interval: 5s
```

应用该 EnvoyFilter 后，Sidecar Proxy 每隔 5s 就会输出一次访问日志。如下所示：

```bash
[2023-05-15T10:37:08.842Z] "- - -" 0 - - - "-" 3 0 - - "-" "-" "-" "-" "10.244.0.70:9080" outbound|9080||productpage.default.svc.cluster.local 10.244.0.72:41238 10.96.219.213:9080 10.244.0.72:53492 - -
[2023-05-15T10:37:08.842Z] "- - -" 0 - - - "-" 3 0 - - "-" "-" "-" "-" "10.244.0.70:9080" outbound|9080||productpage.default.svc.cluster.local 10.244.0.72:41238 10.96.219.213:9080 10.244.0.72:53492 - -
[2023-05-15T10:37:08.842Z] "- - -" 0 - - - "-" 3 0 - - "-" "-" "-" "-" "10.244.0.70:9080" outbound|9080||productpage.default.svc.cluster.local 10.244.0.72:41238 10.96.219.213:9080 10.244.0.72:53492 - -
[2023-05-15T10:37:08.842Z] "- - -" 0 - - - "-" 3 0 - - "-" "-" "-" "-" "10.244.0.70:9080" outbound|9080||productpage.default.svc.cluster.local 10.244.0.72:41238 10.96.219.213:9080 10.244.0.72:53492 - -
```