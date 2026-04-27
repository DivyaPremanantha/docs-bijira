# API Platform Gateway: Memory Scalability Fix and Load Test Results

When we started load-testing the API Platform Gateway at scale, we hit a problem: router memory didn't plateau. It kept climbing. By the time we reached 500 deployed APIs, the router was consuming **5.83 GiB** — a 207x increase from where it started. That's not a scaling curve. That's a leak.

This post is about how we found it, fixed it, and what the gateway looks like after the fix — including throughput and latency numbers we're confident publishing.

## Test Environment

The two tests were run on different environments. The load test used a minimum-requirement setup to establish a conservative, reproducible baseline. The scalability test used an increased resource allocation to stress-test API count limits.

### Scalability Test

| | |
|---|---|
| **Runner** | Local machine |
| **Gateway deployment** | Docker Compose, single-instance stack, no container resource caps (required to observe pre-fix unbounded memory growth) |
| **Test script** | `create_apis_and_capture_stats.sh` — sequential API creation, resource sampled after each |
| **APIs created** | 500 |
| **Success criterion** | HTTP 201 on every creation |

Resource allocations for the gateway during the scalability test:

| Container | Includes | CPU | Memory |
|-----------|----------|-----|--------|
| `gateway-controller` | Controller | uncapped | uncapped |
| `gateway-runtime` | Router (Envoy) + Policy Engine | uncapped | uncapped |

### Load Test

| | |
|---|---|
| **Runner** | GitHub Actions `ubuntu-latest` (4 vCPU, 16 GiB RAM) |
| **OS** | Ubuntu 24.04 |
| **Gateway deployment** | Docker Compose, single-instance stack |
| **Load generator** | Apache JMeter 5.6.3 (1 GiB heap, co-located on the same runner) |
| **Test plan** | `weather_perf_random_50.jmx` — randomised requests across deployed APIs |
| **Backend** | Netty echo service, in-process via Docker Compose (~0 ms backend latency) |

Resource allocations for the gateway during the load test:

| Container | Includes | CPU | Memory |
|-----------|----------|-----|--------|
| `gateway-controller` | Controller | 0.025 cores | 60 MiB |
| `gateway-runtime` | Router (Envoy) + Policy Engine | 0.175 cores | 180 MiB |
| **Total** | | **0.2 cores** | **240 MiB** |

!!! note
    The load generator (JMeter) ran on the same machine as the gateway. On a dedicated load generator with the gateway on isolated hardware, throughput figures would be higher. The numbers here represent a conservative, reproducible baseline.

## What Was Happening

The API Platform Gateway uses a router component to maintain an active route table for all deployed APIs. Every time a new API is created, the router receives an updated configuration. What we discovered was that route configuration objects were accumulating in memory without being released — each API creation added to the heap, and nothing was clearing the stale entries.

The effect was predictable in hindsight: memory grew roughly linearly at **~11.9 MiB per API**, and CPU followed as the router spent increasing time reconciling an ever-growing configuration set.

At 200 APIs, router CPU had already spiked to **16.73%**. At 300 APIs, it was sitting at **23.24%**. By 500 APIs, it peaked at **40.56%** — with a recorded high of **46.07% at 277 APIs** during the test run.

## The Fix

The root cause was unbounded accumulation of route configuration state. The fix ensured that stale route entries are evicted when APIs are updated or when the configuration is reconciled, keeping the active route table bounded to only what is currently deployed.

Specifically, the fix switched the Envoy HTTP connection managers from inline route configuration to **RDS (Route Discovery Service)** with a single shared `RouteConfiguration` named `shared_route_config`. Previously, each listener update embedded route configs inline — one per API — causing Envoy to accumulate `RouteConfiguration` objects in memory without releasing the stale ones. With RDS, all API routes are merged into a single `RouteConfiguration` that is atomically replaced on every update by the xDS control plane. Envoy tracks exactly one route config object regardless of how many APIs are deployed, which is why memory stops growing linearly with API count.

The structural difference in the xDS listener config looks like this:

```yaml
# Before — inline route config (one RouteConfiguration object per listener update)
filter_chains:
  - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          route_config:                      # embedded inline
            virtual_hosts:
              - name: api_1
                routes: [...]
              - name: api_2
                routes: [...]
              # ... grows with every API added

# After — RDS with a single shared RouteConfiguration
filter_chains:
  - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          rds:
            route_config_name: shared_route_config   # one object, atomically replaced
            config_source:
              ads: {}
```

With the inline approach, each update from the xDS control plane pushed a new `RouteConfiguration` object into Envoy's memory alongside the previous one. With RDS, updates replace the single `shared_route_config` in place — memory stays constant regardless of how many times routes are updated or how many APIs are deployed.

## Scalability Results: Before and After

All tests were run by sequentially creating 500 APIs and sampling component resource usage after each creation. All 500 requests returned HTTP 201 in both runs.

### Router Memory

This is where the difference is most visible.

![Router Memory Before vs. After Optimization](../assets/img/performance/router_memory_vs_api_count.png)

| API Count | Before — Memory | After — Memory | Reduction |
|-----------|----------------|----------------|-----------|
| 1 | 28.10 MiB | 25.09 MiB | — |
| 100 | 680.2 MiB | 47.45 MiB | −93% |
| 200 | 1.98 GiB | 67.49 MiB | −97% |
| 300 | 3.49 GiB | 87.52 MiB | −98% |
| 400 | 4.63 GiB | 103.6 MiB | −98% |
| 500 | 5.83 GiB | 123.7 MiB | −97.9% |

At 500 APIs, the router now uses **123.7 MiB** — a memory growth rate of **~0.2 MiB per API**, down from ~11.9 MiB. The total gateway footprint across all three components at 500 APIs is under **240 MiB**.

### Router CPU

The memory fix also eliminated the CPU overhead caused by reconciling a bloated route table.

![Router CPU Before vs. After Optimization](../assets/img/performance/router_cpu_comparison.png)

| API Count | Before — CPU (%) | After — CPU (%) |
|-----------|-----------------|-----------------|
| 1 | 0.28 | 0.27 |
| 100 | 0.48 | 0.36 |
| 200 | 16.73 | 0.33 |
| 300 | 23.24 | 5.89 |
| 400 | 30.51 | 6.68 |
| 500 | 40.56 | 10.57 |

Peak router CPU dropped from **46.07% to 12.45%** — a 73% reduction. More importantly, CPU no longer trends upward in proportion to API count.

### Controller

The controller showed improvement too, though the effect was less dramatic since it wasn't directly responsible for the memory accumulation.

![Controller Memory Before vs. After Optimization](../assets/img/performance/controller_memory_comparison.png)

| API Count | Before — CPU (%) | Before — Memory | After — CPU (%) | After — Memory |
|-----------|-----------------|-----------------|-----------------|----------------|
| 1 | 0.00 | 15.69 MiB | 0.00 | 18.34 MiB |
| 100 | 0.00 | 37.99 MiB | 0.00 | 41.35 MiB |
| 200 | 0.00 | 47.04 MiB | 0.00 | 64.54 MiB |
| 300 | 0.00 | 56.91 MiB | 0.02 | 86.79 MiB |
| 400 | 0.00 | 65.59 MiB | 0.00 | 87.96 MiB |
| 500 | 0.00 | 57.34 MiB | 0.00 | 103.8 MiB |

Peak controller CPU went from **6.28% to 0.36%**. Controller memory is slightly higher after the fix, growing from a range of 15–88 MiB to 18–108 MiB across 500 APIs.

### Policy Engine

The policy engine CPU remained at or near 0.00% in both runs. Memory was **31–55 MiB** before the fix and **10–12 MiB** after. Policy enforcement overhead scales with traffic volume, not API count.

### Before vs. After Summary

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Router memory at 500 APIs | 5.83 GiB | 123.7 MiB | −97.9% |
| Router memory growth rate | ~11.9 MiB/API | ~0.20 MiB/API | −98.3% |
| Peak router CPU | 46.07% | 12.45% | −73% |
| Peak controller CPU | 6.28% | 0.36% | −94% |
| API creation success rate | 100% | 100% | — |

## Throughput and Latency

With the scalability fix in place, we ran load tests to establish the gateway's throughput and latency profile. These tests measure the gateway in steady state — not under configuration churn, but handling live traffic against a stable set of deployed APIs.

A secured API proxying a Netty echo backend was used as the test target, with concurrent user load stepped from 1 to 401. The Netty backend has near-zero processing latency, which means the measured response times represent gateway overhead in isolation — in production with real backends adding 50–200 ms, latency will shift upward proportionally, but throughput ceiling is determined by the gateway, not the backend.

![Transactions Per Second vs. Number of Users](../assets/img/performance/transactions_per_second_vs_number_of_users.png)

| Concurrent Users | Throughput (TPS) | Avg Response Time (ms) | Error Rate |
|-----------------|-----------------|------------------------|------------|
| 1 | 454.79 | 2.1 ms | 0.00% |
| 51 | 3,352.84 | 12.7 ms | 0.00% |
| 101 | 4,045.35 | 20.8 ms | 0.00% |
| 151 | 4,406.73 | 28.7 ms | 0.00% |
| 201 | 4,619.13 | 36.3 ms | 0.00% |
| 251 | 4,764.09 | 44.0 ms | 0.00% |
| 301 | 4,691.80 | 53.5 ms | 0.00% |
| 351 | 4,910.02 | 60.2 ms | 0.00% |
| 401 | 4,820.31 | 70.3 ms | 0.00% |

Peak throughput reached **4,910.02 TPS at 351 concurrent users**. Beyond that point the gateway holds steady — throughput doesn't collapse under pressure, it plateaus. Latency grows linearly from **2.1 ms at 1 user to 70.3 ms at 401 users**, and the error rate held at **0.00% across every load level**.

That last point matters. Some gateways trade correctness for throughput at high concurrency — dropped connections, partial responses, timeout cascades. None of that appeared here.

## What This Means in Practice

Before the fix, the API Platform Gateway had a hard ceiling on practical API count. Memory would exhaust available resources long before you hit any business limit on the number of APIs you wanted to deploy. That ceiling is gone.

After the fix:

- **At 500 deployed APIs, the router uses 123.7 MiB and the controller uses 103.8 MiB.** Memory consumption is bounded and predictable — you can capacity-plan confidently as your API catalogue grows.
- **CPU stays predictable.** Router CPU no longer climbs in proportion to API count — it reflects actual traffic load, which is what you want.
- **Throughput peaks at 4,910 TPS with zero errors.** For the vast majority of API workloads, the gateway is not the bottleneck.

We're publishing these numbers because we think transparency about how we test, what we found, and how we fixed it is more useful than a polished benchmark that hides the work behind it. If you want to reproduce these results or run your own tests against the API Platform Gateway, the full test data is available in the [`docs/performance/`](docs/performance/) directory of the repository, and the CI workflow that generates it is at [`.github/workflows/perf-gateway.yml`](.github/workflows/perf-gateway.yml).
