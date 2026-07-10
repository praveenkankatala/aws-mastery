# API Gateway — Troubleshooting Guide

Organized by symptom. Each entry: **Symptom → Likely Cause(s) → Fix / Diagnostic Command**.

---

## 1. Gateway Won't Start

**Symptom:** Container exits immediately or crash-loops.

- **Port already in use** — another process is bound to 8000/8001/443.
  ```bash
  sudo lsof -i -P -n | grep LISTEN
  ```
  Kill the conflicting process or remap the container's port.
- **DB connection failure** (DB-backed Kong) — Postgres not reachable yet.
  ```bash
  docker logs kong | grep -i "could not connect"
  ```
  Fix: ensure Postgres container is healthy *before* Kong starts (`depends_on` + healthcheck in Compose, not just container start order).
- **Invalid declarative config** — YAML syntax error in `kong.yml` / `envoy.yaml`.
  ```bash
  docker logs kong | grep -i "error"
  # Envoy specifically:
  docker run --rm -v $(pwd)/envoy.yaml:/etc/envoy/envoy.yaml envoyproxy/envoy --mode validate -c /etc/envoy/envoy.yaml
  ```

---

## 2. 502 / 503 / 504 Errors (Bad Gateway / Unavailable / Timeout)

**Symptom:** Gateway responds, but with an error instead of forwarding successfully.

- **Upstream is down or unreachable** —
  ```bash
  docker ps | grep backend-a          # is it even running?
  docker exec kong getent hosts backend-a   # can the gateway resolve it?
  ```
- **Wrong upstream port in config** — double check the service definition points at the container's *internal* port, not the host-mapped port.
- **Health check marking a healthy upstream as down** — check thresholds; an overly aggressive health check (e.g. 1 failure = ejected) will flap under normal latency jitter.
  ```bash
  curl -s http://localhost:8001/upstreams/user-upstream/health | jq
  ```
- **Timeout too low for a genuinely slow backend** — increase `upstream_connect_timeout` / `upstream_read_timeout` rather than assuming the service is broken.

---

## 3. JWT / OAuth Validation Failing (401/403 when you expect 200)

- **Clock skew** — JWT `exp`/`iat` validation fails if the client or gateway server clock has drifted. Sync both with NTP.
- **Wrong signing secret / key ID (`kid`) mismatch** — confirm the consumer's registered secret matches what signed the token.
- **Token sent in the wrong place** — most gateways expect `Authorization: Bearer <token>`; a token passed as a query param or custom header won't be picked up unless explicitly configured.
- **Algorithm mismatch** — token signed with `RS256` but plugin configured for `HS256` (or vice versa).
- **Diagnostic:** decode the token locally (base64-decode the header/payload — never trust "it looks fine") and compare `alg`, `exp`, and `kid` against what the gateway expects.

---

## 4. Rate Limiting Behaving Unexpectedly

**Too permissive (limit never triggers):**
- Using `policy=local` counters on a multi-node gateway — each node counts independently, so the *effective* global limit is `limit × node count`. Switch to `policy=redis` (or cluster/shared counters) for a true global limit.

**Too aggressive (legitimate traffic throttled):**
- Limiting by IP behind a shared NAT/corporate proxy — many real users share one apparent IP. Consider limiting by API key or user ID instead of raw IP.
- Burst traffic from a single legitimate client (e.g. a mobile app retry storm) — check if a burst/leaky-bucket allowance is configured, not just a flat per-minute cap.

**Diagnostic:**
```bash
curl -i http://localhost:8000/v1/users | grep -i ratelimit
# Inspect X-RateLimit-Limit / X-RateLimit-Remaining / Retry-After
```

---

## 5. TLS / SSL Handshake Failures

- **Self-signed cert rejected** — expected in local dev; use `-k` in curl for testing, or trust the CA properly, never disable verification in production.
- **Cert/key mismatch** — `nginx -t` will usually catch this before reload.
  ```bash
  docker exec nginx-gw nginx -t
  ```
- **SNI mismatch** — client requests one hostname, gateway serves a cert for another. Confirm the `server_name` block matches the `Host` the client is actually sending.
- **Expired certificate** — check expiry directly:
  ```bash
  echo | openssl s_client -connect localhost:8443 2>/dev/null | openssl x509 -noout -dates
  ```

---

## 6. CORS Errors in the Browser

- **Preflight (`OPTIONS`) not handled** — many gateways need an explicit CORS plugin/config; a route that only matches `GET`/`POST` will 404 on the browser's automatic `OPTIONS` preflight.
- **`Access-Control-Allow-Origin` too strict or missing** — confirm the gateway (not just the backend) is the one setting this header; if both set it differently, the browser will reject the mismatched/duplicate header.
- **Credentials + wildcard origin conflict** — `Access-Control-Allow-Credentials: true` cannot be paired with `Access-Control-Allow-Origin: *`; the origin must be explicit.

---

## 7. Circuit Breaker Stuck Open (Healthy Service Still Rejected)

- **Ejection window too long** — the outlier detection `base_ejection_time` may be set high (minutes), so a service that recovered in seconds still looks "down" to the gateway. Lower it for faster recovery, but not so low that flapping services get re-admitted before they've actually stabilized.
- **Health check endpoint itself is broken** — the *service* is fine, but its `/health` endpoint depends on a DB connection that's slow, causing false negatives. Health checks should be as dependency-light as possible.
- **Diagnostic (Envoy):**
  ```bash
  curl -s http://localhost:9901/clusters | grep -i outlier
  ```

---

## 8. Canary / Blue-Green Split Not Matching Configured Percentages

- **Session affinity / sticky sessions overriding weights** — if the gateway is configured to pin a client to one target by cookie or IP hash, weighted round-robin won't apply per-request; it'll apply per unique client.
- **Too small a sample size** — 90/10 over 10 requests will not look like 9/1 reliably; test over hundreds of requests before concluding the split is broken.
- **Weights configured on the wrong object** — double check weights are set on upstream *targets*, not on the service or route (which don't support weighting directly).

---

## 9. High Latency / "The Gateway Is Slow"

- **Confirm it's actually the gateway and not the backend:**
  ```bash
  curl -o /dev/null -s -w "DNS:%{time_namelookup}s Connect:%{time_connect}s TLS:%{time_appconnect}s TTFB:%{time_starttransfer}s Total:%{time_total}s\n" https://your-gateway/endpoint
  ```
  Compare against calling the backend directly (bypassing the gateway) to isolate whether the added hop, or the backend itself, is the bottleneck.
- **Plugin chain overhead** — each plugin (auth, transformation, logging) adds processing time; a long chain of synchronous plugins compounds. Check whether any plugin makes a blocking external call (e.g. a remote auth check) on every single request.
- **DNS resolution on every request** — if the gateway isn't caching upstream DNS lookups, this alone can add tens of milliseconds per request.
- **Missing connection pooling/keep-alive to upstreams** — re-establishing a TCP (and TLS) connection per request to the backend is expensive; verify keep-alive is enabled between gateway and upstream.

---

## 10. Config Changes Not Reaching All Nodes (Control Plane / Data Plane Split)

- **mTLS between control plane and data plane broken** — data plane nodes silently fail to pull new config if their cert to authenticate against the control plane expired or is misconfigured. Check data plane logs specifically for connection/auth errors to the control plane, not just proxy errors.
- **Network policy blocking the control-plane port** — in Kubernetes, a `NetworkPolicy` or firewall rule may allow proxy traffic (data plane) but block the control-plane sync port.
- **Stale config due to caching** — some gateways cache declarative config in memory and need an explicit reload signal, not just a file change on disk.
  ```bash
  curl -X POST http://localhost:8001/config -F config=@kong.yml
  ```

---

## 11. Service Discovery Issues

- **Stale DNS cache** — the gateway resolved an upstream's IP once and kept using it after the pod/container was rescheduled to a new IP. Check the gateway's DNS TTL/refresh interval.
- **Health checks passing against a dead endpoint due to caching** — force a re-resolve:
  ```bash
  docker exec kong getent hosts backend-a
  ```
- **Kubernetes-specific:** confirm the Service (not just the Pod) is what the gateway targets — Pod IPs change on every restart, Service IPs (ClusterIP) don't.

---

## 12. General Debugging Checklist (run through this before escalating)

```bash
# 1. Is the gateway container healthy?
docker ps --filter "name=kong" --format "{{.Status}}"

# 2. What do the access/error logs say about the failing request?
docker logs --since 5m kong

# 3. Can you reach the backend directly, bypassing the gateway entirely?
curl -i http://backend-a:80/get

# 4. Is DNS resolving correctly from inside the gateway's network namespace?
docker exec kong getent hosts backend-a

# 5. Is the route/plugin config what you think it is (not what you last edited)?
curl -s http://localhost:8001/routes | jq
curl -s http://localhost:8001/services | jq
curl -s http://localhost:8001/plugins | jq

# 6. Reproduce with verbose curl to see every header, redirect, and timing detail
curl -v http://localhost:8000/v1/users
```

**Golden rule:** always isolate *gateway* behavior from *backend* behavior by testing the backend directly first. Most "the gateway is broken" reports turn out to be a backend issue the gateway is just faithfully reporting.
