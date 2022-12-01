
---
title: "Sidecar/Gateway 优雅退出"
linkTitle: "Sidecar/Gateway 优雅退出"
weight: 2
date: 2022-12-01
description: 
---

## Istio 中 Envoy 的退出机制
缺省情况下，在收到 SIGTERM 后，Istio-agent 会在等待 terminationDrainDuration (缺省 5S）后退出，由于 Envoy 是 Istio-agent 的子进程，Envoy 也会随之退出。该缺省行为可能对于一些耗时较长的关键业务有影响，导致正在进行业务处理的链接被强制中断。

## 通过 EXIT_ON_ZERO_ACTIVE_CONNECTIONS 参数配置优雅退出
Istio 1.12 版本中为 Istio-agent 引入了 EXIT_ON_ZERO_ACTIVE_CONNECTIONS 环境变量，通过该变量可以实现 Envoy 的优雅退出。当配置该变量为 true 之后，Istio-agent 会以 1S 的固定间隔检查 Envoy 中的活动链接数，当链接数量为 0 后才会退出。Istio-agent 中该部分代码如下所示：

```go
// 配置了 EXIT_ON_ZERO_ACTIVE_CONNECTIONS 为 true 时，检查活动链接为 0 后再退出
if a.exitOnZeroActiveConnections {
		log.Infof("Agent draining proxy for %v, then waiting for active connections to terminate...", a.minDrainDuration)
		time.Sleep(a.minDrainDuration)
		log.Infof("Checking for active connections...")
		ticker := time.NewTicker(activeConnectionCheckDelay)
		for range ticker.C {
			if a.activeProxyConnections() == 0 {
				log.Info("There are no more active connections. terminating proxy...")
				a.abortCh <- errAbort
				return
			}
		}
	} else { //缺省情况下等待 5S 即退出
		log.Infof("Graceful termination period is %v, starting...", a.terminationDrainDuration)
		time.Sleep(a.terminationDrainDuration)
		log.Infof("Graceful termination period complete, terminating remaining proxies.")
		a.abortCh <- errAbort
	}
```

## 配置方法

### 全局配置

```yaml
meshConfig:
  defaultConfig:
    proxyMetadata: 
      EXIT_ON_ZERO_ACTIVE_CONNECTIONS: 'true'
```

### 按 workload 单独配置
在 deploy 中通过 annotation 为 pilot-agent 添加 EXIT_ON_ZERO_ACTIVE_CONNECTIONS 环境变量。
```yaml
annotations:
  proxy.istio.io/config: |
    proxyMetadata:
      EXIT_ON_ZERO_ACTIVE_CONNECTIONS: 'true'
```

## 配置 pod 的 terminationGracePeriodSeconds 参数

Kubernetes 在向 pod 发出 SIGTERM 信号后，会缺省等待 30S，如果 30S 后 pod 还未结束，Kubernetes 会向 pod 发出 SIGKILL 信号。因此，即使设置了 EXIT_ON_ZERO_ACTIVE_CONNECTIONS 为 true，Envoy 最多也只能等待 30S，如果你的应用需要等待更长时间，则需要设置 pod 的 terminationGracePeriodSeconds 参数。下面的示例将 terminationGracePeriodSeconds 从缺省的 30S 延长到了 60S。

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
    name: test
spec:
    replicas: 1
    template:
        spec:
            containers:
              - name: test
                image: ...
            terminationGracePeriodSeconds: 60
```

## 参考链接

* [Kubernetes best practices: terminating with grace](https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-best-practices-terminating-with-grace)
* [Istio-agent Environment variables](https://istio.io/latest/docs/reference/commands/pilot-agent/#envvars)