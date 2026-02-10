# Initial Setup

0. Setup Cluster

```
k3d cluster create istio --k3s-arg "--disable=traefik@server:*" -p "80:80@loadbalancer" -p "443:443@loadbalancer"
```

1. Install Istio

```
curl -L https://istio.io/downloadIstio | sh -
export PATH="$PATH:/Users/gtrekter/Desktop/istio/istio-1.28.3/bin"
istioctl install --set profile=demo
```

2. Deploy Simple-App and Inject with Istio

```
kubectl apply -k ./application 
kubectl label namespace simple-app istio-injection=enabled
kubectl rollout restart deploy -n simple-app
```

# Deploy an istio example

## Virtual Service

A VirtualService defines a set of traffic routing rules to apply when a host is addressed. Each routing rule defines matching criteria for traffic of a specific protocol. If the traffic is matched, then it is sent to a named destination service (or subset/version of it) defined in the registry.

### 01. HTTP

```
kubectl apply -f ./examples/01.yaml

curl -H "Host: simple-app.127.0.0.1.sslip.io" http://127.0.0.1/v1/
Simple App v1

curl -i -X POST -H "Host: simple-app.127.0.0.1.sslip.io" -H "version: v1" http://127.0.0.1/ -d 'test'
HTTP/1.1 200 OK
x-app-name: http-echo
x-app-version: 1.0.0
date: Tue, 10 Feb 2026 05:28:28 GMT
content-length: 14
content-type: text/plain; charset=utf-8
x-envoy-upstream-service-time: 10
server: istio-envoy
Simple App v1

curl -H "Host: simple-app.127.0.0.1.sslip.io" http://127.0.0.1/v2/
Simple App v2

curl -i -X POST -H "Host: simple-app.127.0.0.1.sslip.io" -H "version: v55" http://127.0.0.1/ -d 'test'
HTTP/1.1 200 OK
x-app-name: http-echo
x-app-version: 1.0.0
date: Tue, 10 Feb 2026 06:41:14 GMT
content-length: 14
content-type: text/plain; charset=utf-8
x-envoy-upstream-service-time: 10
server: istio-envoy
Simple App v3
```

### 02. TCP

```
kubectl apply -f ./examples/02.yaml

nc simple-app.127.0.0.1.sslip.io 443
Simple App v1
```

### 03. Virtual Service TLS

```
kubectl apply -f ./examples/03.yaml

echo | openssl s_client -connect v1.smihost.127.0.0.1.sslip.io:443 -servername v5.smihost.127.0.0.1.sslip.io -quiet
Connecting to 127.0.0.1
depth=0 CN=localhost
verify error:num=18:self-signed certificate
verify return:1
depth=0 CN=localhost
verify return:1
Simple App v1
```

### 04. HTTP with Weight

```
kubectl apply -f ./examples/04.yaml

curl -H "Host: simple-app.127.0.0.1.sslip.io" http://127.0.0.1/v1/
Simple App v1
curl -H "Host: simple-app.127.0.0.1.sslip.io" http://127.0.0.1/v1/
Simple App v2
```

# 05. HTTP with Timeout

```
kubectl apply -f ./examples/05.yaml

curl -H "Host: simple-app.127.0.0.1.sslip.io" http://127.0.0.1/v1/
upstream request timeout             
```

# 06. HTTP with Retries

```
kubectl apply -f ./examples/06.yaml

curl -H "Host: simple-app.127.0.0.1.sslip.io" http://127.0.0.1/v1/
upstream request timeout             
```

## Destination Rule

DestinationRule defines policies that apply to traffic intended for a service after routing has occurred. These rules specify configuration for load balancing, connection pool size from the sidecar, and outlier detection settings to detect and evict unhealthy hosts from the load balancing pool. 
