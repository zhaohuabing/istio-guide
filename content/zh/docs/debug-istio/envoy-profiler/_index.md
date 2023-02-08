---
title: "Envoy 内存/CPU分析"
linkTitle: ""
weight: 3
date: 2023-02-08
description: 
---

# 导出 Enovy 的 CPU 和 内存 profile
https://github.com/istio/istio/wiki/Analyzing-Istio-Performance#profile

## Profile

On Istio 1.5 and older:

```bash
export POD=pod-name
export NS=istio-system
kubectl exec -n "$NS" "$POD" -c istio-proxy -- sh -c 'sudo mkdir -p /var/log/envoy && sudo chmod 777 /var/log/envoy && curl -X POST -s "http://localhost:15000/heapprofiler?enable=y"'
sleep 15
kubectl exec -n "$NS" "$POD" -c istio-proxy -- sh -c 'curl -X POST -s "http://localhost:15000/heapprofiler?enable=n"'
rm -rf /tmp/envoy
kubectl cp -n "$NS" "$POD":/var/log/envoy/ /tmp/envoy -c istio-proxy
kubectl cp -n "$NS" "$POD":/lib/x86_64-linux-gnu /tmp/envoy/lib -c istio-proxy
kubectl cp -n "$NS" "$POD":/usr/local/bin/envoy /tmp/envoy/lib/envoy -c istio-proxy
```

On Istio 1.6+

```bash
export POD=pod-name
export NS=istio-system
export PROFILER="cpu" # Can also be "heap", for a heap profile
kubectl exec -n "$NS" "$POD" -c istio-proxy -- curl -X POST -s "http://localhost:15000/${PROFILER}profiler?enable=y"
sleep 15
kubectl exec -n "$NS" "$POD" -c istio-proxy -- curl -X POST -s "http://localhost:15000/${PROFILER}profiler?enable=n"
rm -rf /tmp/envoy
kubectl cp -n "$NS" "$POD":/var/lib/istio/data /tmp/envoy -c istio-proxy
kubectl cp -n "$NS" "$POD":/lib/x86_64-linux-gnu /tmp/envoy/lib -c istio-proxy
kubectl cp -n "$NS" "$POD":/usr/local/bin/envoy /tmp/envoy/lib/envoy -c istio-proxy
```

## Visualize profile pprof installation

Install pprof, then run:

```bash
PPROF_BINARY_PATH=/tmp/envoy/lib/ pprof -pdf /tmp/envoy/lib/envoy /tmp/envoy/envoy.prof.0001.heap
```

Or, interactively

```bash
PPROF_BINARY_PATH=/tmp/envoy/lib/ pprof /tmp/envoy/lib/envoy /tmp/envoy/envoy.prof.0001.heap
```

Or, through the web UI

```bash
PPROF_BINARY_PATH=/tmp/envoy/lib/ pprof -http=localhost:8000 /tmp/envoy/lib/envoy /tmp/envoy/envoy.prof.0001.heap
```

# 采用 Envoy admin 查看内存使用情况

```bash
kubectl exec -n "$NS" "$POD" -c istio-proxy -- curl  "http://localhost:15000/memory"
```

输出
```bash
{
 "allocated": "221674328",
 "heap_size": "361693184",
 "pageheap_unmapped": "86106112",
 "pageheap_free": "21831680",
 "total_thread_cache": "9805104",
 "total_physical_bytes": "278470656"
}
```

字段含义：

https://www.envoyproxy.io/docs/envoy/latest/api-v3/admin/v3/memory.proto
