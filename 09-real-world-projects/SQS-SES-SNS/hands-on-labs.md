# API Gateway — Hands-On Labs

14 progressive labs. Each one builds on the last. All labs use Docker so they run identically on any machine. Every lab maps to a concept in `README.md` — do them in order the first time through.

**Prerequisites:** Docker + Docker Compose installed, `curl`, `jq` (optional but recommended), `openssl`.

---

## Lab 1 — Environment Setup & Mock Backends

**Goal:** Stand up two "microservices" to route traffic to.

1. Create an isolated network:
   ```bash
   docker network create gateway-net
   ```
2. Start two mock backend services using `httpbin` (it echoes back request details, perfect for seeing what the gateway actually forwards):
   ```bash
   docker run -d --name backend-a --network gateway-net -p 8081:80 kennethreitz/httpbin
   docker run -d --name backend-b --network gateway-net -p 8082:80 kennethreitz/httpbin
   ```
3. **Verify:**
   ```bash
   curl http://localhost:8081/get
   curl http://localhost:8082/get
   ```
   You should get a JSON response echoing your request headers back.

**Checkpoint:** Two backend containers running on the same Docker network, reachable directly.

---

## Lab 2 — Deploy Your First Gateway (Kong, DB-less)

**Goal:** Get Kong running in front of your backends with zero database dependency.

1. Create `kong.yml`:
   ```yaml
   _format_version: "3.0"
   services:
     - name: user-service
       url: http://backend-a:80
       routes:
         - name: user-route
           paths:
             - /v1/users
   ```
2. Run Kong (see cheatsheet §2 for the full command):
   ```bash
   docker run -d --name kong --network gateway-net \
     -e "KONG_DATABASE=off" \
     -e "KONG_DECLARATIVE_CONFIG=/kong/declarative/kong.yml" \
     -e "KONG_ADMIN_LISTEN=0.0.0.0:8001" \
     -v $(pwd)/kong.yml:/kong/declarative/kong.yml \
     -p 8000:8000 -p 8001:8001 \
     kong:latest
   ```
3. **Verify routing works:**
   ```bash
   curl -i http://localhost:8000/v1/users
   ```
   Expect a `200` and a JSON body from `backend-a`, proving the gateway proxied your request to the right upstream.

**Checkpoint:** You've reproduced the exact `Clients → Gateway → Services` diagram from the README, for real.

---

## Lab 3 — Path-Based Routing to Multiple Services

**Goal:** Route different paths to different backends (the Routing/Reverse-Proxy function).

1. Extend `kong.yml` with a second service:
   ```yaml
   services:
     - name: user-service
       url: http://backend-a:80
       routes:
         - name: user-route
           paths: ["/v1/users"]
     - name: order-service
       url: http://backend-b:80
       routes:
         - name: order-route
           paths: ["/v1/orders"]
   ```
2. Reload config: `curl -X POST http://localhost:8001/config -F config=@kong.yml`
3. **Verify:**
   ```bash
   curl -i http://localhost:8000/v1/users   # → backend-a
   curl -i http://localhost:8000/v1/orders  # → backend-b
   ```

**Checkpoint:** Same gateway, same port, two completely different backends — clients never need to know either backend's real address.

---

## Lab 4 — JWT Authentication

**Goal:** Reject unauthenticated requests at the gateway, before they ever reach `user-service`.

1. Enable the JWT plugin and create a consumer + credential (see cheatsheet §2).
2. **Verify a request WITHOUT a token is rejected:**
   ```bash
   curl -i http://localhost:8000/v1/users
   # Expect: 401 Unauthorized
   ```
3. Generate a signed JWT using the consumer's secret (use jwt.io debugger locally, or a small script with a JWT library in your language of choice) and retry with:
   ```bash
   curl -i http://localhost:8000/v1/users -H "Authorization: Bearer <TOKEN>"
   # Expect: 200 OK
   ```

**Checkpoint:** You've centralized auth at the perimeter — `backend-a` never had to implement any login logic.

---

## Lab 5 — Rate Limiting

**Goal:** Watch the Token Bucket algorithm reject a client after a threshold.

1. Add the rate-limiting plugin capped at a low number for testing (e.g. 5/minute) — see cheatsheet §2.
2. **Verify by hammering the endpoint:**
   ```bash
   for i in $(seq 1 10); do
     curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8000/v1/users
   done
   ```
   Expect the first 5 to return `200`, the rest `429 Too Many Requests`.
3. Check the response headers on a `429` — note `X-RateLimit-Remaining` and `Retry-After`.

**Checkpoint:** You've protected a backend from being hammered, without writing a single line of rate-limit code in the service itself.

---

## Lab 6 — TLS Termination with Nginx

**Goal:** Terminate HTTPS at the edge; backend stays on plain HTTP.

1. Generate a self-signed cert (cheatsheet §4).
2. Write a minimal `nginx.conf` that listens on 443 with your cert/key and proxies to `backend-a:80`.
3. Run Nginx (cheatsheet §4).
4. **Verify:**
   ```bash
   curl -k -i https://localhost:8443/
   ```
   The `-k` flag skips cert validation since it's self-signed — in production you'd use a CA-signed cert (e.g. via Let's Encrypt) and drop `-k`.

**Checkpoint:** Client-facing traffic is encrypted; internal traffic to `backend-a` stays fast, plain HTTP.

---

## Lab 7 — Circuit Breaking (Envoy)

**Goal:** Watch a gateway stop sending traffic to a failing upstream automatically.

1. Write a minimal `envoy.yaml` with an `outlier_detection` block on the cluster pointing to `backend-a`.
2. Run Envoy (cheatsheet §5).
3. Kill `backend-a` mid-test: `docker stop backend-a`
4. **Verify:** send several requests — after a few consecutive failures, Envoy should stop even attempting connections (check `curl -s http://localhost:9901/clusters` for the `outlier_detection` status showing the host ejected).
5. Restart `backend-a` (`docker start backend-a`) and confirm Envoy re-admits it after its health check window passes.

**Checkpoint:** You've seen a circuit "open" and "close" — the core resiliency pattern that stops one failing service from cascading into a full outage.

---

## Lab 8 — Canary / Blue-Green Deployment

**Goal:** Split traffic 90/10 between two backend "versions."

1. Create a Kong upstream with two weighted targets (cheatsheet §9): `backend-a` (weight 90) and `backend-b` (weight 10).
2. Point `user-service` at the upstream instead of a direct URL.
3. **Verify the split:**
   ```bash
   for i in $(seq 1 100); do curl -s http://localhost:8000/v1/users | jq -r '.origin'; done | sort | uniq -c
   ```
   You should see roughly a 90/10 distribution between the two container IPs.
4. Change the weights to 50/50 and re-run to see the split shift live, with zero downtime and no redeploy.

**Checkpoint:** This is exactly how teams roll out risky changes gradually in production.

---

## Lab 9 — Protocol Translation (REST → gRPC)

**Goal:** Accept a REST/JSON call at the edge, forward it as gRPC internally.

1. Use Envoy's `grpc_json_transcoder` filter, pointed at a `.proto` file describing a simple service (e.g. a `Greeter` service with a `SayHello` RPC).
2. Run a minimal gRPC server implementing that proto (any language — Go/Python/Node all have quick-start gRPC examples).
3. **Verify:**
   ```bash
   curl -X POST http://localhost:10000/say-hello -d '{"name":"world"}'
   ```
   Envoy should transcode this JSON POST into a gRPC call and translate the gRPC response back to JSON.

**Checkpoint:** Clients never need to know or care that your backend runs gRPC.

---

## Lab 10 — Response Aggregation

**Goal:** One client call → gateway fans out to two services → one combined response.

1. Write a tiny Node.js/Express (or any language) "aggregation gateway" that, on `GET /dashboard`, calls both `backend-a:80/get` and `backend-b:80/get` in parallel (`Promise.all`), then merges both JSON bodies into one response object.
2. Run it in the same Docker network.
3. **Verify:**
   ```bash
   curl -s http://localhost:<your-port>/dashboard | jq
   ```
   Confirm the response contains fields from both backends in a single payload.

**Checkpoint:** This is the pattern behind most "dashboard" or "home screen" APIs that stitch together many microservices for one screen.

---

## Lab 11 — Observability: Metrics, Logs, and Tracing

**Goal:** See gateway traffic in Prometheus/Grafana, and trace one request across services.

1. Enable Kong's Prometheus plugin: `curl -i -X POST http://localhost:8001/plugins --data name=prometheus`
2. Run Prometheus configured to scrape `kong:8001/metrics` (cheatsheet §8), and Grafana pointed at Prometheus as a data source.
3. Generate some traffic, then open Grafana (`localhost:3000`) and build a simple panel graphing request rate/latency.
4. Separately, run Zipkin and enable Kong's `zipkin` plugin pointing at it. Send a request, then search for its trace in the Zipkin UI (`localhost:9411`) using the `X-Request-ID` or trace header.

**Checkpoint:** You can now answer "how many requests hit this route in the last hour, and what did request `abc123` actually do end to end?"

---

## Lab 12 — Control Plane / Data Plane Split (Kong Hybrid Mode)

**Goal:** Separate "where I configure routes" from "where traffic is actually served."

1. Run one Kong node in `control_plane` role (connects to a Postgres DB) and one or more Kong nodes in `data_plane` role (no DB, they pull config from the control plane over mTLS).
2. Push a new route via the control plane's Admin API.
3. **Verify:** the data plane node picks up the new route *without you touching it directly* — confirm by curling the data plane's proxy port and seeing the new route respond, then check its logs to confirm it never touched the Admin API directly.

**Checkpoint:** This is how a single config change safely reaches gateway nodes in multiple regions with zero downtime.

---

## Lab 13 — Developer Portal & Monetization

**Goal:** Simulate exposing an API to external developers with tiered quotas.

1. Create two Kong consumers: `free-tier-dev` and `premium-tier-dev`.
2. Apply a rate-limiting plugin scoped to each consumer individually — 10/day for free tier, unlimited (very high limit) for premium.
3. Issue an API key to each consumer (`key-auth` plugin).
4. **Verify:** call the API 11 times as `free-tier-dev` and confirm the 11th call gets throttled, while `premium-tier-dev` sails through the same volume.

**Checkpoint:** This is the mechanism behind every "Free / Pro / Enterprise" public API pricing page you've ever seen.

---

## Lab 14 — Backend-for-Frontend (BFF) Pattern

**Goal:** Build two different gateways for the same backend data, each optimized for a different client.

1. Build a minimal **Mobile BFF** (Node/Express) that calls `backend-a` and strips the response down to only 3 essential fields, gzip-compressed.
2. Build a minimal **Web BFF** that calls the same `backend-a` but returns the full payload plus extra fields the desktop UI needs, and sets a session cookie.
3. **Verify:**
   ```bash
   curl -s http://localhost:<mobile-bff-port>/profile | jq   # small payload
   curl -s -i http://localhost:<web-bff-port>/profile         # full payload + Set-Cookie header
   ```

**Checkpoint:** Same underlying service, two purpose-built gateways — proving BFF isn't "duplicate work," it's tailoring the contract to the consumer.

---

## Cleanup (run after any lab)

```bash
docker stop $(docker ps -aq)
docker rm $(docker ps -aq)
docker network rm gateway-net
```
