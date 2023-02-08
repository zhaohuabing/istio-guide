---
title: "Envoy 内存/CPU分析"
linkTitle: ""
weight: 1
date: 2022-07-06
description: 
---

# 导出 Enovy 的 CPU 和 内存 profile
https://github.com/istio/istio/wiki/Analyzing-Istio-Performance#profile

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
