# API Gateway — Commands Cheatsheet

Copy-paste reference for everything in `hands-on-labs.md`. Organized by tool/task.

---

## 1. Environment Setup

```bash
# Check Docker & Compose are installed
docker --version
docker compose version

# Create a shared network all labs will use
docker network create gateway-net

# Spin up a mock backend service (httpbin echoes back whatever you send it)
docker run -d --name backend-a --network gateway-net -p 8081:80 kennethreitz/httpbin
docker run -d --name backend-b --network gateway-net -p 8082:80 kennethreitz/httpbin

# Tear everything down when done
docker stop $(docker ps -aq) && docker rm $(docker ps -aq)
docker network rm gateway-net
```

---

## 2. Kong Gateway (Docker, DB-less mode)

```bash
# Pull the image
docker pull kong:latest

# Run Kong in DB-less (declarative) mode
docker run -d --name kong \
  --network gateway-net \
  -e "KONG_DATABASE=off" \
  -e "KONG_DECLARATIVE_CONFIG=/kong/declarative/kong.yml" \
  -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
  -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
  -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
  -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
  -e "KONG_ADMIN_LISTEN=0.0.0.0:8001" \
  -v $(pwd)/kong.yml:/kong/declarative/kong.yml \
  -p 8000:8000 -p 8001:8001 \
  kong:latest

# Reload config after editing kong.yml (DB-less mode)
curl -X POST http://localhost:8001/config \
  -F config=@kong.yml

# List all configured services
curl -s http://localhost:8001/services | jq

# List all routes
curl -s http://localhost:8001/routes | jq

# List all active plugins
curl -s http://localhost:8001/plugins | jq

# Add a service via Admin API (DB-backed mode only)
curl -i -X POST http://localhost:8001/services \
  --data name=user-service \
  --data url=http://backend-a:80

# Add a route to that service
curl -i -X POST http://localhost:8001/services/user-service/routes \
  --data paths[]=/v1/users \
  --data methods[]=GET

# Enable rate limiting on a service
curl -i -X POST http://localhost:8001/services/user-service/plugins \
  --data name=rate-limiting \
  --data config.minute=100 \
  --data config.policy=local

# Enable JWT auth plugin
curl -i -X POST http://localhost:8001/services/user-service/plugins \
  --data name=jwt

# Create a consumer + JWT credential
curl -i -X POST http://localhost:8001/consumers --data username=demo-user
curl -i -X POST http://localhost:8001/consumers/demo-user/jwt

# Check upstream health (if using upstreams/targets, not raw service URL)
curl -s http://localhost:8001/upstreams/user-upstream/health | jq
```

---

## 3. Testing Requests Through the Gateway

```bash
# Basic proxied request (Kong proxy defaults to port 8000)
curl -i http://localhost:8000/v1/users

# Send a request with a JWT
curl -i http://localhost:8000/v1/users \
  -H "Authorization: Bearer <JWT_TOKEN>"

# Hammer an endpoint to trigger rate limiting (100 requests, watch for 429s)
for i in $(seq 1 120); do
  curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8000/v1/users
done | sort | uniq -c

# Verbose mode to inspect headers added/stripped by the gateway
curl -v http://localhost:8000/v1/users

# Test with a custom Host header (host-based routing)
curl -i http://localhost:8000/ -H "Host: api.partner.com"
```

---

## 4. Nginx as a Gateway (TLS Termination + Caching)

```bash
# Generate a self-signed cert for local TLS testing
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx.key -out nginx.crt \
  -subj "/CN=localhost"

# Run Nginx with a mounted config + certs
docker run -d --name nginx-gw \
  --network gateway-net \
  -p 8443:443 \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
  -v $(pwd)/nginx.crt:/etc/nginx/nginx.crt:ro \
  -v $(pwd)/nginx.key:/etc/nginx/nginx.key:ro \
  nginx:latest

# Reload config without downtime after edits
docker exec nginx-gw nginx -s reload

# Test config syntax before reloading (catches typos before they break prod)
docker exec nginx-gw nginx -t

# Test TLS termination
curl -k -i https://localhost:8443/

# Check cache HIT/MISS headers (if X-Cache-Status configured)
curl -i https://localhost:8443/products -k | grep -i x-cache
```

---

## 5. Envoy (Circuit Breaking + gRPC Protocol Translation)

```bash
# Run Envoy with a local config file
docker run -d --name envoy-gw \
  --network gateway-net \
  -p 10000:10000 -p 9901:9901 \
  -v $(pwd)/envoy.yaml:/etc/envoy/envoy.yaml:ro \
  envoyproxy/envoy:v1.29-latest

# View Envoy's live admin stats (circuit breaker state, connection pools)
curl -s http://localhost:9901/stats | grep circuit_breakers

# View cluster health as Envoy sees it
curl -s http://localhost:9901/clusters

# Force-drain and hot-restart config after editing envoy.yaml
curl -X POST http://localhost:9901/drain_listeners
docker restart envoy-gw

# Send a request through Envoy to the backend cluster
curl -i http://localhost:10000/
```

---

## 6. AWS API Gateway (CLI)

```bash
# List existing REST APIs
aws apigateway get-rest-apis

# Create a new REST API
aws apigateway create-rest-api --name "demo-api"

# Get the root resource ID (needed to attach methods)
aws apigateway get-resources --rest-api-id <API_ID>

# Create a GET method on the root resource
aws apigateway put-method \
  --rest-api-id <API_ID> \
  --resource-id <RESOURCE_ID> \
  --http-method GET \
  --authorization-type NONE

# Deploy the API to a stage
aws apigateway create-deployment \
  --rest-api-id <API_ID> \
  --stage-name dev

# Create a usage plan (monetization / quotas)
aws apigateway create-usage-plan \
  --name "free-tier" \
  --throttle burstLimit=20,rateLimit=10 \
  --quota limit=1000,period=DAY

# Create an API key and associate it with a usage plan
aws apigateway create-api-key --name "demo-key" --enabled
aws apigateway create-usage-plan-key \
  --usage-plan-id <PLAN_ID> \
  --key-id <KEY_ID> \
  --key-type API_KEY
```

---

## 7. Kubernetes-based Gateways (Ingress / Gateway API)

```bash
# Apply a Gateway API resource
kubectl apply -f gateway.yaml

# List Gateways and HTTPRoutes
kubectl get gateway
kubectl get httproute

# Describe a route to debug why traffic isn't matching
kubectl describe httproute user-route

# Check gateway controller logs
kubectl logs -l app=envoy-gateway -n gateway-system --tail=100 -f

# Port-forward to test locally without an external LB
kubectl port-forward svc/envoy-gateway 8080:80 -n gateway-system
```

---

## 8. Observability Stack (Prometheus + Grafana + Zipkin)

```bash
# Run Prometheus scraping the gateway's /metrics endpoint
docker run -d --name prometheus \
  --network gateway-net \
  -p 9090:9090 \
  -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml:ro \
  prom/prometheus

# Run Grafana
docker run -d --name grafana \
  --network gateway-net \
  -p 3000:3000 \
  grafana/grafana

# Run Zipkin for distributed tracing
docker run -d --name zipkin \
  --network gateway-net \
  -p 9411:9411 \
  openzipkin/zipkin

# Query Prometheus directly for gateway request rate
curl -s 'http://localhost:9090/api/v1/query?query=rate(kong_http_requests_total[1m])'

# Check a trace by ID in Zipkin
curl -s http://localhost:9411/api/v2/trace/<TRACE_ID>
```

---

## 9. Canary / Blue-Green Traffic Splitting (Kong Upstream Targets)

```bash
# Create an upstream (a named load-balanced pool)
curl -i -X POST http://localhost:8001/upstreams --data name=user-upstream

# Add v1 target with 90% weight
curl -i -X POST http://localhost:8001/upstreams/user-upstream/targets \
  --data target=backend-a:80 --data weight=90

# Add v2 (canary) target with 10% weight
curl -i -X POST http://localhost:8001/upstreams/user-upstream/targets \
  --data target=backend-b:80 --data weight=10

# Point the service at the upstream instead of a single backend
curl -i -X PATCH http://localhost:8001/services/user-service \
  --data host=user-upstream

# Fire 50 requests and count how many hit each version (via a response header each backend sets)
for i in $(seq 1 50); do curl -s http://localhost:8000/v1/users -o /dev/null -w "%{http_code}\n"; done
```

---

## 10. Quick Diagnostic Commands

```bash
# Is the gateway container even running?
docker ps | grep -E "kong|nginx|envoy"

# Tail logs live
docker logs -f kong

# Check what's actually listening on expected ports
sudo lsof -i -P -n | grep LISTEN

# DNS resolution check from inside the gateway container (service discovery issues)
docker exec kong getent hosts backend-a

# Round-trip latency test
curl -o /dev/null -s -w "DNS: %{time_namelookup}s | Connect: %{time_connect}s | TLS: %{time_appconnect}s | Total: %{time_total}s\n" https://localhost:8443/
```
