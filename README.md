# Simple App + Istio Playground

A minimal k3d + Istio setup to practice common traffic‑management patterns (HTTP, TCP, TLS, weights, timeouts, retries, fault injection) using the sample “simple-app”.

## Prerequisites
- k3d (or another local Kubernetes with LoadBalancer support)
- kubectl, kustomize (kubectl ≥1.21 has kustomize built‑in)
- istioctl (1.28.x recommended)
- curl, nc, openssl for testing

## Quick Start (Mac/Linux)
```bash
# 0) Create a local cluster with ports 80/443 exposed and two nodes in differrent availability zone
k3d cluster create istio \
  --k3s-arg "--disable=traefik@server:*" \
  --k3s-arg "--node-taint=node-role.kubernetes.io/master:NoSchedule@server:*" \
  -p "80:80@loadbalancer" \
  -p "443:443@loadbalancer" \
  -p "8080:8080@loadbalancer" \
  --agents 2 \
  --k3s-arg "--node-label=topology.kubernetes.io/zone=apne2-az1@agent:0" \
  --k3s-arg "--node-label=topology.kubernetes.io/zone=apne2-az2@agent:1"

# 1) Install Istio demo profile
curl -L https://istio.io/downloadIstio | sh -
export PATH="$PATH:$(pwd)/istio-1.28.3/bin"
istioctl install --set profile=demo

# 2) Deploy the sample app and enable sidecar injection
kubectl apply -k ./application
kubectl label namespace simple-app istio-injection=enabled --overwrite
kubectl rollout restart deploy -n simple-app
kubectl get pods -n simple-app
```

## Repo Layout
- `application/` — namespace, Deployments (v1/v2/v3; http/tcp/tls), Services, kustomization.
- `examples/` — Istio Gateway/VirtualService/DestinationRule examples (01–06).

## Usage: Examples (run from repo root)
Set a host variable for convenience:
```bash
export SIMPLE_HOST=simple-app.127.0.0.1.sslip.io
```

### Virtual Service
- **01 HTTP routing** – path- and header-based:
  Routes traffic based on URI prefixes (`/v1/`, `/v2/`) and custom request headers (`version: v1`). Requests that don't match any specific rule fall through to the default v3 backend.
  ```bash
  kubectl apply -f ./examples/virtual_service/01-http-routing.yaml

  curl -H "Host: $SIMPLE_HOST" http://127.0.0.1/v1/
  Simple App v1

  curl -i -X POST -H "Host: $SIMPLE_HOST" -H "version: v1" http://127.0.0.1/ -d 'test'
  HTTP/1.1 200 OK
  x-app-name: http-echo
  x-app-version: 1.0.0
  date: Tue, 10 Feb 2026 05:28:28 GMT
  content-length: 14
  content-type: text/plain; charset=utf-8
  x-envoy-upstream-service-time: 10
  server: istio-envoy
  Simple App v1

  curl -H "Host: $SIMPLE_HOST" http://127.0.0.1/v2/
  Simple App v2

  curl -i -X POST -H "Host: $SIMPLE_HOST" -H "version: v55" http://127.0.0.1/ -d 'test'
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
- **02 TCP passthrough**:
  Opens a raw TCP gateway on port 443 and forwards traffic directly to the backend. Useful for non-HTTP protocols or when the application handles its own protocol.
  ```bash
  kubectl apply -f examples/virtual_service/02-tcp-passthrough.yaml

  nc $SIMPLE_HOST 443
  Simple App v1
  ```
- **03 TLS SNI routing** (gateway passthrough to backend HTTP):
  Uses TLS PASSTHROUGH mode on the gateway so the TLS handshake reaches the backend pod. Routes are selected based on the SNI hostname, allowing multiple TLS services on the same port.
  ```bash
  kubectl apply -f examples/virtual_service/03-tls-sni-routing.yaml

  echo | openssl s_client -connect v1.smihost.127.0.0.1.sslip.io:443 -servername v5.smihost.127.0.0.1.sslip.io -quiet
  Connecting to 127.0.0.1
  depth=0 CN=localhost
  verify error:num=18:self-signed certificate
  verify return:1
  depth=0 CN=localhost
  verify return:1
  Simple App v1
  ```
- **04 HTTP weighted split (50/50 v1/v2)**:
  Splits traffic evenly between v1 and v2 using percentage-based weights. This is the foundation for canary deployments and A/B testing.
  ```bash
  kubectl apply -f examples/virtual_service/04-http-weighted-split.yaml
  curl -H "Host: $SIMPLE_HOST" http://127.0.0.1/v1/
  Simple App v1

  curl -H "Host: $SIMPLE_HOST" http://127.0.0.1/v1/
  Simple App v2
  ```
- **05 HTTP timeout (1ms)**:
  Sets an intentionally short 1ms request timeout on the route. Any backend response slower than 1ms triggers a `408 upstream request timeout`, demonstrating how Istio enforces deadline budgets.
  ```bash
  kubectl apply -f examples/virtual_service/05-http-timeout.yaml
  curl -H "Host: $SIMPLE_HOST" http://127.0.0.1/v1/
  upstream request timeout   
  ```
- **06 HTTP retries**:
  Configures automatic retries (3 attempts, 1s per-try timeout) on gateway errors, connection failures, and refused streams. The sidecar proxy retries transparently; you can verify this by watching the proxy logs where the same `x-request-id` appears multiple times.
  Note: You can see the retries thanks to the request ID `5c6994e9-27e8-94e1-92d1-506fe5383b5b`
  ```bash
  kubectl apply -f examples/virtual_service/06-http-retries.yaml

  kubectl logs -n simple-app deployment/simple-app-v4-http -c istio-proxy --tail=0 -f & curl -H "Host: $SIMPLE_HOST" http://127.0.0.1/v1/
  [2026-02-13T08:53:32.766Z] "GET /v1/ HTTP/1.1" 503 - via_upstream - "-" 0 14 0 0 "10.42.0.1" "curl/8.7.1" "5c6994e9-27e8-94e1-92d1-506fe5383b5b" "simple-app.127.0.0.1.sslip.io" "10.42.0.46:5678" inbound|5678|| 127.0.0.6:34969 10.42.0.46:5678 10.42.0.1:0 outbound_.80_._.simple-app-v4-http.simple-app.svc.cluster.local default
  [2026-02-13T08:53:32.780Z] "GET /v1/ HTTP/1.1" 503 - via_upstream - "-" 0 14 1 0 "10.42.0.1" "curl/8.7.1" "5c6994e9-27e8-94e1-92d1-506fe5383b5b" "simple-app.127.0.0.1.sslip.io" "10.42.0.46:5678" inbound|5678|| 127.0.0.6:34251 10.42.0.46:5678 10.42.0.1:0 outbound_.80_._.simple-app-v4-http.simple-app.svc.cluster.local default
  [2026-02-13T08:53:32.804Z] "GET /v1/ HTTP/1.1" 503 - via_upstream - "-" 0 14 0 0 "10.42.0.1" "curl/8.7.1" "5c6994e9-27e8-94e1-92d1-506fe5383b5b" "simple-app.127.0.0.1.sslip.io" "10.42.0.46:5678" inbound|5678|| 127.0.0.6:34251 10.42.0.46:5678 10.42.0.1:0 outbound_.80_._.simple-app-v4-http.simple-app.svc.cluster.local default
  [2026-02-13T08:53:32.894Z] "GET /v1/ HTTP/1.1" 503 - via_upstream - "-" 0 14 0 0 "10.42.0.1" "curl/8.7.1" "5c6994e9-27e8-94e1-92d1-506fe5383b5b" "simple-app.127.0.0.1.sslip.io" "10.42.0.46:5678" inbound|5678|| 127.0.0.6:34251 10.42.0.46:5678 10.42.0.1:0 outbound_.80_._.simple-app-v4-http.simple-app.svc.cluster.local default
  ```
- **07 HTTP fault injection**:
  Injects a 503 abort fault on 50% of requests to `/v1/`. This simulates service failures without modifying application code, useful for testing resilience and observability.
  ```bash
  kubectl apply -f examples/virtual_service/07-http-fault-injection.yaml

  curl -H "Host: $SIMPLE_HOST" http://127.0.0.1/v1/
  fault filter abort 
  ```
- **08 HTTP mirroring**:
  Mirrors 100% of live traffic to v2 while routing real requests to v1. The mirrored requests are fire-and-forget and their responses are discarded. This lets you test a new version against production traffic without impacting users.
  Note: Traffic is routed to v1 but mirrored to v2. Check the v2 app container logs to see the mirrored request.
  ```bash
  kubectl apply -f examples/virtual_service/08-http-mirroring.yaml

  kubectl logs -n simple-app deployment/simple-app-v2-http -c http-app --tail=1
  2026/02/13 09:39:42 simple-app.127.0.0.1.sslip.io 127.0.0.6:33433 "GET /v1/ HTTP/1.1" 200 14 "curl/8.7.1" 6µs
  
  curl -H "Host: $SIMPLE_HOST" http://127.0.0.1/v1/
  Simple App v1

  kubectl logs -n simple-app deployment/simple-app-v2-http -c http-app --tail=1
  2026/02/13 09:44:57 simple-app.127.0.0.1.sslip.io 127.0.0.6:33433 "GET /v1/ HTTP/1.1" 200 14 "curl/8.7.1" 6.917µs
  ```
- **09 HTTP redirect/rewrite**:
  Rewrites the URI path from `/` to `/v1` before forwarding to the backend. The client is unaware of the rewrite; it happens transparently inside the mesh.
  ```bash
  kubectl apply -f examples/virtual_service/09-http-redirect-rewrite.yaml

  curl -H "Host: $SIMPLE_HOST" http://127.0.0.1/ 
  Simple App v1
  ```
- **10 Header manipulation**:
  Demonstrates adding, setting, and removing HTTP headers on both requests and responses. Routes traffic to v2 when the `x-version: v2` header is present, and adds a custom `x-routed-by: istio-vs` response header.
  ```bash
  kubectl apply -f examples/virtual_service/10-header-manipulation.yaml

  curl -i -H "Host: $SIMPLE_HOST" http://127.0.0.1/v1/ 
  HTTP/1.1 200 OK
  x-app-name: http-echo
  x-app-version: 1.0.0
  date: Mon, 16 Feb 2026 08:47:26 GMT
  content-length: 14
  content-type: text/plain; charset=utf-8
  x-envoy-upstream-service-time: 1
  server: istio-envoy
  Simple App v1

  curl -i -H "Host: $SIMPLE_HOST" -H "x-version: v2" http://127.0.0.1/v1/
  HTTP/1.1 200 OK
  x-app-name: http-echo
  x-app-version: 1.0.0
  date: Mon, 16 Feb 2026 08:46:46 GMT
  content-length: 14
  content-type: text/plain; charset=utf-8
  server: istio-envoy
  x-routed-by: istio-vs
  Simple App v2
  ```

### DestinationRule

It defines policies that apply to traffic intended for a service after routing has occurred. 

- **1 Load Balancing**:
  Configures different load-balancing algorithms per port: LEAST_REQUEST on port 80 and ROUND_ROBIN on port 8080. Send 500 requests and inspect the Envoy cluster stats to see how each algorithm distributes traffic across replicas.
  ```bash
  kubectl apply -f examples/destination_rule/01-load-balancing.yaml

  kubectl patch svc istio-ingressgateway -n istio-system --type=json \
  -p '[{"op":"add","path":"/spec/ports/-","value":{"name":"http-8080","port":8080,"protocol":"TCP","targetPort":8080}}]'

  kubectl scale deployment simple-app-v1-http -n simple-app --replicas=3

  for i in $(seq 1 500); do
    curl -s http://simple-app.127.0.0.1.sslip.io:80
  done

  kubectl exec deploy/istio-ingressgateway -n istio-system -c istio-proxy -- curl -s localhost:15000/clusters | grep "simple-app-v1-http" | grep "rq_total"
  outbound|80||simple-app-v1-http.simple-app.svc.cluster.local::10.42.0.58:5678::rq_total::161
  outbound|80||simple-app-v1-http.simple-app.svc.cluster.local::10.42.0.59:5678::rq_total::182
  outbound|80||simple-app-v1-http.simple-app.svc.cluster.local::10.42.0.60:5678::rq_total::157

  kubectl rollout restart deployment simple-app-v1-http -n simple-app 

  for i in $(seq 1 500); do
    curl -s http://simple-app.127.0.0.1.sslip.io:8080
  done

  kubectl exec deploy/istio-ingressgateway -n istio-system -c istio-proxy -- curl -s localhost:15000/clusters | grep "simple-app-v1-http" | grep "rq_total"
  outbound|80||simple-app-v1-http.simple-app.svc.cluster.local::10.42.0.61:5678::rq_total::176
  outbound|80||simple-app-v1-http.simple-app.svc.cluster.local::10.42.0.62:5678::rq_total::174
  outbound|80||simple-app-v1-http.simple-app.svc.cluster.local::10.42.0.63:5678::rq_total::177
  ```

- **2 Circuit Breaker**:
  Sets strict connection pool limits and outlier detection so that when a backend starts returning errors, the proxy ejects it from the load-balancing pool. Sending 20 concurrent requests triggers the circuit breaker, returning 503s with the `UO` (upstream overflow) response flag.
  ```bash
  kubectl apply -f examples/destination_rule/02-circuit-breaker.yaml

  seq 1 20 | xargs -P 20 -I {} curl -s -o /dev/null -w "HTTP %{http_code}\n" http://simple-app.127.0.0.1.sslip.io:80
  HTTP 503
  HTTP 503
  HTTP 200
  HTTP 200
  HTTP 503
  HTTP 503
  HTTP 503
  HTTP 503
  HTTP 503
  HTTP 200
  HTTP 200
  HTTP 200
  HTTP 503
  HTTP 503
  HTTP 503
  HTTP 503
  HTTP 503
  HTTP 503
  HTTP 503
  HTTP 200

  kubectl exec deploy/istio-ingressgateway -n istio-system -c istio-proxy -- curl -s localhost:15000/stats | grep "simple-app-v1-http" | grep "response_flags.UO"
  istiocustom.istio_requests_total.reporter.source.source_workload.istio-ingressgateway.source_canonical_service.istio-ingressgateway.source_canonical_revision.latest.source_workload_namespace.istio-system.source_principal.unknown.source_app.istio-ingressgateway.source_version.source_cluster.Kubernetes.destination_workload.unknown.destination_workload_namespace.unknown.destination_principal.unknown.destination_app.unknown.destination_version.unknown.destination_service.simple-app-v1-http.simple-app.svc.cluster.local.destination_canonical_service.unknown.destination_canonical_revision.latest.destination_service_name.simple-app-v1-http.destination_service_namespace.simple-app.destination_cluster.unknown.request_protocol.http.response_code.503.grpc_response_status.response_flags.UO.connection_security_policy.unknown: 58
  ```

- **3 Consistent Hash (Cookie-based sticky sessions)**:
  Uses consistent-hash load balancing keyed on an HTTP cookie (`session-id`, 30min TTL). The proxy generates the cookie on the first request and all subsequent requests with the same cookie are pinned to the same backend pod.
  ```bash
  kubectl apply -f examples/destination_rule/03-consistent-hash.yaml

  kubectl scale deployment simple-app-v1-http -n simple-app --replicas=3

  curl -v http://simple-app.127.0.0.1.sslip.io:80 2>&1 | grep -i "set-cookie"
  < set-cookie: session-id="c16e58e194011962"; Max-Age=1800; HttpOnly

  for i in $(seq 1 10); do
    curl -s -b "session-id=test123" http://simple-app.127.0.0.1.sslip.io:80
  done

  kubectl exec deploy/istio-ingressgateway -n istio-system -c istio-proxy -- curl -s localhost:15000/clusters | grep "simple-app-v1-http" | grep "rq_total"
  outbound|80||simple-app-v1-http.simple-app.svc.cluster.local::10.42.0.63:5678::rq_total::0
  outbound|80||simple-app-v1-http.simple-app.svc.cluster.local::10.42.0.64:5678::rq_total::0
  outbound|80||simple-app-v1-http.simple-app.svc.cluster.local::10.42.0.65:5678::rq_total::10
  ```

- **WIP: 4 TCP with TLS Origination**:
  Accepts plain TCP on port 80 at the gateway and upgrades the connection to TLS (SIMPLE mode with `insecureSkipVerify`) before forwarding to the backend. This lets clients connect without TLS while the mesh handles encryption.
  ```bash
  kubectl apply -f examples/destination_rule/04-tls-origination.yaml

  echo | nc simple-app.127.0.0.1.sslip.io 80
  Simple App v1
  ```

- **5 Connection Pool (HTTP)**:
  Fine-tunes TCP and HTTP connection pool settings: max connections, keepalive probes, idle timeouts, and H2 upgrade policy. Sending concurrent requests exceeds the pool limits, causing 503 responses from the proxy.
  ```bash
  kubectl apply -f examples/destination_rule/05-connection-pool.yaml

  seq 1 10 | xargs -P 100 -I {} curl -s -o /dev/null -w "HTTP %{http_code}\n" http://simple-app.127.0.0.1.sslip.io:80 
  HTTP 503
  HTTP 503
  HTTP 200
  HTTP 503
  HTTP 503
  HTTP 503
  HTTP 503
  HTTP 503
  HTTP 200
  HTTP 200
  ```

- **6 Locality Failover**:
  Configures locality-aware routing so traffic prefers pods in the same availability zone as the ingress gateway (`apne2-az1`). When that zone's pods are unavailable (simulated by cordoning the node), traffic automatically fails over to `apne2-az2`.
  ```bash
  kubectl apply -f examples/destination_rule/06-locality-failover.yaml

  kubectl scale deployment simple-app-v1-http -n simple-app --replicas=2

  kubectl get pods -n simple-app -o wide
  NAME                                  READY   STATUS        RESTARTS      AGE   IP           NODE                NOMINATED NODE   READINESS GATES
  simple-app-v1-http-69846dfddd-sn6d9   2/2     Running       0             37s   10.42.1.9    k3d-istio-agent-0   <none>           <none>
  simple-app-v1-http-69846dfddd-zn2j9   2/2     Running       0             5s    10.42.2.12   k3d-istio-agent-1   <none>           <none>

  for i in $(seq 1 100); do
    curl -s http://simple-app.127.0.0.1.sslip.io:80
  done

  kubectl exec deploy/istio-ingressgateway -n istio-system -c istio-proxy -- curl -s localhost:15000/clusters | grep "simple-app-v1-http" | grep "rq_total"
  outbound|80||simple-app-v1-http.simple-app.svc.cluster.local::10.42.2.12:5678::rq_total::100
  outbound|80||simple-app-v1-http.simple-app.svc.cluster.local::10.42.1.9:5678::rq_total::0

  kubectl cordon k3d-istio-agent-0
  kubectl delete pod -n simple-app -l app=simple-app --field-selector spec.nodeName=k3d-istio-agent-0

  for i in $(seq 1 10); do
    curl -s http://simple-app.127.0.0.1.sslip.io:80
  done

  kubectl exec deploy/istio-ingressgateway -n istio-system -c istio-proxy -- curl -s localhost:15000/clusters | grep "simple-app-v1-http" | grep "rq_total"
  outbound|80||simple-app-v1-http.simple-app.svc.cluster.local::10.42.2.12:5678::rq_total::110

  kubectl uncordon k3d-istio-agent-0
  ```

## Cleanup
```bash
kubectl delete -k ./application
k3d cluster delete istio
```
