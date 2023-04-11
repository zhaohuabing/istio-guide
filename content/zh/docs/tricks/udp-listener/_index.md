---
title: "支持 UDP Listener"
weight: 1
date: 2023-04-11
description: >
  在 Ingress Gateway 上对外提供 UDP 服务。
---

Istio 并不会处理 UDP 类型的服务，当我们需要在 Ingress Gateway 上对外提供 UDP 服务时，可以通过 EnvoyFilter 来实现。

## 创建用于测试的 UDP 服务

创建一个 coredns，用于作为后端的测试 UDP 服务。

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: coredns
  namespace: default
  labels:
    app: coredns
spec:
  ports:
    - name: udp-dns
      port: 53
      protocol: UDP
      targetPort: 53
  selector:
    app: coredns
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  labels:
    app: coredns
spec:
  selector:
    matchLabels:
      app: coredns
  template:
    metadata:
      labels:
        app: coredns
    spec:
      containers:
        - args:
            - -conf
            - /root/Corefile
          image: coredns/coredns
          name: coredns
          volumeMounts:
            - mountPath: /root
              name: conf
      volumes:
        - configMap:
            defaultMode: 420
            name: coredns
          name: conf
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
data:
  Corefile: |
    .:53 {
        forward . 8.8.8.8 9.9.9.9
        log
        errors
    }

    foo.bar.com:53 {
      whoami
    }
EOF  
```

创建一个用于测试的 network-tool pod，该 pod 中包含了 dig 命令行工具。

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
 name: network-tool
 annotations:
   sidecar.istio.io/inject: "true"
spec:
 containers:
   - name: network-tool
     image: zhaohuabing/network-tool
     securityContext:
       capabilities:
         add:
           - NET_ADMIN
EOF
```

此时通过 network-tool 中的 dig 工具去查询 foo.bar.com 这个域名，可以查询成功。

```bash
➜  ~ kubectl exec network-tool -- dig @10.244.0.20 -p 53 foo.bar.com

; <<>> DiG 9.18.1-1ubuntu1.3-Ubuntu <<>> @10.244.0.20 -p 53 foo.bar.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 48665
;; flags: qr aa rd; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 3
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 79265989d39004a6 (echoed)
;; QUESTION SECTION:
;foo.bar.com.			IN	A

;; ADDITIONAL SECTION:
foo.bar.com.		0	IN	A	10.244.0.21
_udp.foo.bar.com.	0	IN	SRV	0 0 37336 .

;; Query time: 1 msec
;; SERVER: 10.244.0.20#53(10.244.0.20) (UDP)
;; WHEN: Tue Apr 11 02:41:01 UTC 2023
;; MSG SIZE  rcvd: 114
```

## 通过 EnvoyFilter 在 Ingress Gateway 创建 UDP Listener 和 对应的 Cluster

EnvoyFilter 如下所示，该 EnvoyFilter 在 Ingress Gateway 上创建了一个 UDP Listener，该 UDP Listener 在 5300 端口上监听来自客户端的请求，并将请求转发到后端的 Coredns 服务上。

> 备注：
此处的 EnvoyFilter 中硬编码了 Cluster 中 Endpoint 地址。由于 UDP 服务的 pod 地址会变化，因此在实际使用时，我们需要编写一个 Controller 来监听 UDP 服务，以动态生成该 EnvoyFilter。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: udp-listener
  namespace: istio-system
spec:
  workloadSelector:
    labels:
      istio: ingressgateway
  configPatches:
  - applyTo: LISTENER
    match:
      context: GATEWAY
    patch:
      operation: ADD
      value:
        name: udp_listener
        address:
          socket_address:
            protocol: UDP
            address: 0.0.0.0
            port_value: 5300
        udp_listener_config:
          downstream_socket_config:
            max_rx_datagram_size: 9000
        listener_filters:
        - name: envoy.filters.udp_listener.udp_proxy
          typed_config:
            '@type': type.googleapis.com/envoy.extensions.filters.udp.udp_proxy.v3.UdpProxyConfig
            stat_prefix: coredns
            matcher:
              on_no_match:
                action:
                  name: route
                  typed_config:
                    '@type': type.googleapis.com/envoy.extensions.filters.udp.udp_proxy.v3.Route
                    cluster: coredns
  - applyTo: CLUSTER
    match:
      context: GATEWAY
    patch:
      operation: ADD
      value:
        name: coredns
        type: STATIC
        lb_policy: ROUND_ROBIN
        load_assignment:
          cluster_name: coredns
          endpoints:
          - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: 10.244.0.20
                    port_value: 53
```

此时通过 network-tool 中的 dig 命令访问 Ingress Gateway 的 5300 端口，可以查询到 foo.bar.com 的 地址，说明 UDP Listener 创建成功。

```bash
➜  ~ kubectl exec network-tool -- dig @10.244.0.14 -p 5300 foo.bar.com

; <<>> DiG 9.18.1-1ubuntu1.3-Ubuntu <<>> @10.244.0.14 -p 5300 foo.bar.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 32291
;; flags: qr aa rd; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 3
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: c4486921ff737611 (echoed)
;; QUESTION SECTION:
;foo.bar.com.			IN	A

;; ADDITIONAL SECTION:
foo.bar.com.		0	IN	A	10.244.0.14
_udp.foo.bar.com.	0	IN	SRV	0 0 32875 .

;; Query time: 1 msec
;; SERVER: 10.244.0.14#5300(10.244.0.14) (UDP)
;; WHEN: Tue Apr 11 02:51:43 UTC 2023
;; MSG SIZE  rcvd: 114
```